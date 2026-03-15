---
title: "Why the Order of Your Graph Data on Disk Matters More Than You Think"
date: 2026-03-15T17:01:00+01:00
summary: "How rearranging records in a graph database can dramatically reduce disk I/O for traversal queries."
---

When you ask a graph database to find the shortest path between two cities, or to traverse a social network breadth-first, the database doesn't just think about algorithms. Behind the scenes, it's frantically loading blocks of data from disk into memory. And here's the thing most people don't consider: **the physical order in which graph records are stored on disk has a massive impact on how many of those expensive disk reads are needed.**

This is the central insight behind the *Graph Order Evaluation Database* -- a research project and master's thesis by Fabian Klopfer at the University of Konstanz that built a graph database from scratch in C, specifically to measure and optimize this effect.

## The Problem: Graphs Don't Fit Neatly on Disk

Relational databases have had decades to optimize how rows are stored. Tables are sorted by primary keys, and joins can be planned around that ordering. But graphs are fundamentally different. When you traverse a graph, the next record you need depends entirely on the graph's structure -- which neighbor comes next, which edge leads where. The access pattern is dictated by connectivity, not by any simple sort order.

Consider how Neo4j -- the most popular native graph database -- stores its data. Nodes are 15-byte fixed-size records in one file. Relationships are 34-byte records in another. Each node points to the head of a doubly-linked list of its incident edges. When you do a breadth-first search, you bounce back and forth between these two files, chasing pointers.

If the records happen to be stored in insertion order (the default), neighboring nodes in the graph might be scattered across completely different disk blocks. Every hop in your traversal becomes a random disk access. On a dataset like the California road network, that's 134,000 node blocks and 389,000 relationship blocks. Random access across that space is brutal.

## Locality: The Principle That Makes Memory Hierarchies Work

The memory hierarchy -- from CPU caches to RAM to SSDs to hard drives -- works because of *locality of reference*. Most programs don't access their data uniformly; they tend to access the same data repeatedly (temporal locality) and data that's nearby (spatial locality).

Caches exploit temporal locality. Prefetching exploits spatial locality. But here's the key formula that drives this work:

> **P(X_{t+1} = A +/- epsilon | X_t = A)**

In plain terms: given that you just accessed address A, what's the probability that the next access will be nearby? If records that are accessed together are *stored* together, this probability goes up, cache hits go up, and the number of costly disk reads goes down.

For a graph database executing traversal queries, this means: **nodes that are neighbors in the graph should be neighbors on disk.**

## Three Dimensions of Graph Record Locality

The thesis identifies that locality in a graph database isn't just about grouping nodes. There are actually three distinct aspects that matter:

**1. Vertex Order** -- Which nodes share a disk block? Ideally, nodes that are connected to each other should be in the same block. When they are, loading one block during a BFS gives you multiple useful nodes at once instead of just one.

**2. Edge Order** -- Similarly, relationships connected to the same source node should be stored in nearby blocks. When you call `expand(node)` to get all of a node's edges, you want those edge records to be loadable in one sequential read, not scattered across dozens of blocks.

**3. Incidence List Order** -- This is the subtle one. Even if edges are grouped perfectly into blocks, the *order of the linked list pointers* within each node's incidence list matters. If the pointers jump back and forth between distant blocks, you lose the sequential access pattern that makes prefetching work. Sorting the incidence list so that pointers proceed in monotonically increasing block order can turn four random reads into one sequential read.

This third dimension -- **incidence list rearrangement** -- is the thesis's novel contribution to the field. Previous work by G-Store and ICBL addressed vertex and edge grouping, but nobody had considered reordering the linked list pointers after rearranging the records.

## The Methods: From Graph Partitioning to Record Layout

The project surveys and implements two state-of-the-art approaches for rearranging graph records, plus the proposed improvement:

### G-Store: Multilevel Partitioning

G-Store adapts the classic multilevel graph partitioning algorithm. It works in three phases:

1. **Coarsen** the graph by repeatedly merging neighboring vertices using heavy edge matching, until you have a tiny graph
2. **Partition** the coarsened graph -- assign vertices to disk blocks based on a weight threshold
3. **Uncoarsen** back to the original graph, refining the partition at each level using three objective functions that minimize inter-block distance, cross-block edges, and linked block count

The algorithm is essentially treating the problem as: *which vertices should share a disk block, and in what order should those blocks appear?*

### ICBL: Diffusion Set-Based Clustering

ICBL (Identify, Cluster, Block-form, Layout) takes a different approach. It characterizes each vertex's neighborhood by running multiple random walks and collecting the visited vertices into a "diffusion set." Then it clusters vertices with similar diffusion sets using a modified k-Means with Jaccard distance, forms blocks using hierarchical agglomerative clustering, and finally ranks and lays them out on disk.

The approach is elegant but has a practical scaling problem: the hierarchical clustering step can produce massively unbalanced clusters on larger graphs, leading to memory allocations of 335 GiB for a graph of 300,000 nodes. The thesis found this made ICBL infeasible for larger datasets.

### The Louvain Method

The community detection algorithm by Blondel et al. serves as a simpler baseline. It finds communities by greedily optimizing modularity -- placing densely-connected groups of nodes together. Records are then sorted by community membership. It's fast and works surprisingly well for some query types, but has no concept of block size and doesn't order records *within* a community.

## Results: Up to 50% Fewer Block Accesses

The evaluation uses five real-world datasets from the Stanford Network Analysis Project (SNAP), ranging from a 131-node neural network (C. elegans) to the 1.1-million-node YouTube social network. Queries include BFS, DFS, Dijkstra's algorithm, A*, and ALT.

Key findings:

- **The natural (insertion order) and random layouts consistently perform worst**, requiring roughly 1.5x the block I/O of partition-based layouts for full-graph traversals (BFS, DFS, Dijkstra).

- **G-Store performs best overall**, especially on larger datasets. On the YouTube dataset (1.1M nodes, 3M edges), it significantly reduces block accesses across all query types.

- **The Louvain method is unpredictable** -- sometimes competitive with G-Store, sometimes worse than the natural layout. Its lack of block-size awareness and non-hierarchical structure explain the inconsistency. On one dataset with A*, it actually performed *worse* than doing nothing.

- **Sorting the incidence lists consistently improves sequential access patterns.** Visualizations of 50-access windows show that sorted incidence lists produce smooth, monotonically increasing access sequences instead of the jagged, jumping patterns seen with unsorted lists.

- **ICBL fails on larger datasets** due to the memory explosion in its clustering step -- an issue the original papers didn't fully address.

## Building the Evaluation Platform

What makes this project particularly interesting from a software engineering perspective is that the entire evaluation platform was built from scratch. The codebase is approximately 15,000 lines of C, implementing:

- A complete storage layer (disk space manager, page cache with LRU eviction, page/frame management)
- Record structures inspired by Neo4j (fixed-size nodes and relationships with incidence lists)
- An access layer with CRUD operations and the `expand` operator
- All traversal and shortest-path algorithms (BFS, DFS, Dijkstra, A*, ALT, random walk)
- Community detection (Louvain method)
- A SNAP dataset importer
- The G-Store and ICBL layout algorithms
- Detailed I/O logging at every layer

The architecture cleanly separates I/O, cache, access, query, and layout concerns, with each layer instrumentable for measuring block accesses. A cost model that counts block-level I/O operations (rather than wall-clock time) provides reproducible, hardware-independent measurements.

## Why This Matters

Graph databases are everywhere -- powering social networks, recommendation engines, knowledge graphs, fraud detection, network routing, and biological modeling. As these graphs grow to billions of edges, the I/O bottleneck becomes the dominant performance factor.

The insight that **you can get significant performance improvements just by rearranging existing data** -- without changing algorithms, hardware, or data structures -- is powerful. It's the graph database equivalent of defragmenting a hard drive, but guided by the graph's structure rather than file creation timestamps.

And the finding that incidence list ordering matters is a practical contribution: it's a simple post-processing step (just sort the linked list pointers) that any graph database using incidence lists could adopt with minimal implementation effort.

## What's Next

The thesis focused on static rearrangement -- reorganizing records once after import. But real databases change over time, and queries change too. A layout optimized for BFS might hurt DFS performance. Dynamic rearrangement that adapts to observed access patterns, rather than being tied to a specific partitioning algorithm, is the natural next step.

There's also the question of whether the newer Leiden algorithm (an improvement over Louvain that guarantees well-connected communities) would produce better layouts. And implementing actual disk-based storage with a real buffer manager -- rather than the in-memory simulation used for evaluation -- would let researchers measure the true end-to-end impact on query latency.

The code is open source, designed as a research environment where new layout methods can be implemented against a simple C interface and benchmarked against the existing methods. If you're interested in graph database internals or storage optimization, it's a solid foundation to build on.

---

*This post is based on the master's thesis "Locality Optimization for Traversal-based Queries on Graph Databases" by Fabian Klopfer, University of Konstanz, 2020, and the accompanying Graph Record Layout Research Environment.*
