---
title: "Building a Graph Database from Scratch in C: Architecture of a Research Storage Engine"
date: 2021-06-01T00:00:00+01:00
tags: ["AI Project Summary"]
summary: "A deep dive into the layered architecture of a minimal graph database built for studying how record layout affects query performance."
---
> *This post was AI-generated from the project's source code, thesis, and documentation. It is an automated summary, not original writing.*

Most developers interact with databases through query languages and APIs. Few ever think about what happens below the surface -- how bytes are arranged in files, how pages are loaded into memory, or why the order of records on disk can make one query ten times faster than another.

This post walks through the architecture of a graph database built entirely from scratch in C. It was designed not as a production system, but as a research instrument: a transparent, fully instrumented storage engine where every disk read can be traced, every cache eviction logged, and every record rearranged to test hypotheses about data locality. The project accompanies a master's thesis at the University of Konstanz and weighs in at roughly 15,000 lines of C.

## Design Constraints

Before diving into the architecture, it's worth understanding the constraints that shaped it:

- **Research-first**: The primary goal is measuring I/O behavior, not production throughput. Every layer is instrumented for logging.
- **Memory-bounded**: The system must run on a machine with 8 GiB of RAM regardless of dataset size. This rules out "just load everything into memory."
- **Neo4j-compatible record format**: The record structures are modeled after Neo4j's storage engine (with simplifications), so results are relevant to real-world systems.
- **POSIX-only**: The system uses standard POSIX I/O interfaces for portability across Linux and macOS.
- **Extensibility**: New layout methods and query algorithms should be easy to add without modifying existing code.

## The Four Layers

The architecture follows the classic database layered pattern, with four distinct modules stacked on top of each other:

```
  +--------------------+     +------------------+
  |   Layout Methods   |     |     Queries      |
  +--------------------+     +------------------+
  |            Layout Utilities                  |
  +----------------------------------------------+
  |                  Access                      |
  +----------------------------------------------+
  |                  Cache                       |
  +----------------------------------------------+
  |                    I/O                       |
  +----------------------------------------------+
```

### Layer 1: I/O (Disk Space Manager)

At the bottom sits the physical I/O layer. It provides two abstractions:

**Files** consist of a set of pages. A file tracks which pages are in use via a page directory, and can grow or shrink dynamically. The interface exposes `read_page`, `write_page`, `grow`, `shrink`, and `clear_page` operations. Underneath, it uses POSIX `fopen`, `fread`, `fwrite`, `fseek`, `ftruncate`, and `fsync`.

**The Disk Manager** sits above individual files, providing `create_file`, `get_file`, `remove_file`, and `get_slots` operations. It manages the collection of database files and coordinates space allocation.

Every read and write operation can be logged to a dedicated file, allowing precise measurement of I/O patterns.

### Layer 2: Cache (Buffer Manager)

The page cache mediates between the I/O layer and everything above it. It splits a fixed amount of RAM into frames, each the size of one page. Its responsibilities:

- **Pin/Unpin**: When the access layer needs a page, it "pins" it -- the cache loads it from disk (if not already cached) and returns a pointer to the frame. When done, the page is "unpinned."
- **LRU Eviction**: When all frames are full and a new page is needed, the least recently used unpinned page is evicted. If it's dirty (modified), it's flushed to disk first.
- **Dirty Tracking**: Modified pages are marked dirty and written back to disk on eviction or explicit flush.
- **Statistics**: The cache tracks total accesses, cache hits, evictions, and dirty writes. These statistics are the primary measurement instrument for the research.

The cache can be resized at runtime (via `page_cache_change_n_frames`), and its log file can be swapped -- both features designed for running comparative experiments.

### Layer 3: Access (Record Structures and Operators)

This is where raw bytes become graph elements. The access layer provides:

**Node records** -- simplified versions of Neo4j's 15-byte node format. Each node has:
- An in-use flag
- A label (unsigned long acting as identifier)
- A pointer to the head of its incidence list (first relationship ID)

**Relationship records** -- inspired by Neo4j's 34-byte format. Each relationship stores:
- Source and target node IDs
- An edge weight
- A label
- Previous/next relationship pointers for *both* the source and target node's incidence lists (forming two interleaved doubly-linked lists)

**The Heap File** provides the CRUD interface: `create_node`, `read_node`, `update_node`, `delete_node` (and equivalents for relationships). Record IDs are implicit -- they encode the record's byte offset in the file, so address translation is just multiplication. No B-tree index is needed.

The critical operation for graph traversal is **`expand`**: given a node ID and a direction (OUTGOING, INCOMING, or BOTH), it follows the incidence list pointers to collect all connected edges. This is where the linked-list structure of the storage format meets the traversal algorithms above.

### Layer 4: Queries and Layout Methods

The top layer is split into two parallel modules:

**Queries** implement graph algorithms that use the access layer:
- BFS and DFS traversals
- Dijkstra's single-source shortest path (using a Fibonacci heap priority queue)
- A* with pluggable heuristics
- ALT (A* with landmark-based triangle inequality heuristics)
- Random walks
- Degree computations
- Louvain community detection
- SNAP dataset importer

Each algorithm is instrumented to log the sequence of node and relationship IDs it accesses. This access log is the raw data for evaluating layout quality.

**Layout Methods** implement record rearrangement strategies:
- Random reordering (as a baseline)
- G-Store (multilevel partitioning)
- ICBL (diffusion set clustering)
- Incidence list sorting (the thesis's novel contribution)

The layout method interface is deliberately simple: given a database, return a mapping from current record IDs to new record IDs. The layout utilities handle the actual record swapping.

## The Record Layout Interface

One of the cleanest design decisions is the layout method contract:

```c
unsigned long[] layout_nodes(graph_database db) {
    require db != null;
    require db.num_nodes > 0;
    ensure db.num_nodes == result array length;
    ensure unique node IDs;
}
```

Any new layout method just needs to implement this function (and the equivalent for relationships). It receives the full database and returns a permutation. The framework handles reordering records according to the permutation, including updating all cross-references (incidence list pointers, node-to-relationship links).

This separation of "what order" from "how to reorder" makes adding new methods straightforward.

## Data Structures

The project includes a substantial library of generic data structures, all in C:

- **Array lists** (dynamically resizing arrays) for unsigned longs, longs, nodes, and relationships
- **Linked lists** with derived queues and stacks
- **Hash tables** (open addressing with chaining) for key-value maps with various type combinations (`dict_ul_ul`, `dict_ul_int`, `dict_ul_d`, `dict_ul_node`, `dict_ul_rel`)
- **Sets** (hash-based) for unsigned longs
- **Fibonacci heaps** for Dijkstra's priority queue
- **In-memory adjacency lists** (`inm_alist_node`, `inm_alist_rel`) for the in-memory graph representation used by layout algorithms

Each data structure follows a consistent API pattern: `create`, `destroy`, `size`, `insert`/`append`, `remove`, `get`, `contains`, with iterators for hash-based structures.

## The Cost Model

Rather than measuring wall-clock time (which depends on hardware, OS scheduling, and background processes), the evaluation uses a deterministic cost model based on block access counting.

Given the sequence of accessed record IDs logged during a query:

1. Compute the block number for each access: `block = (record_id * record_size) / block_size`
2. Count a **block I/O** whenever the block number changes between consecutive accesses
3. Count a **non-sequential I/O** whenever the block number changes by more than one

The cache is modeled as holding exactly one block (worst case). This gives two clean metrics: total block accesses (measuring temporal locality) and non-sequential accesses (measuring spatial locality and prefetch-friendliness).

Setting the spatial locality radius to 8 blocks models a system with 512-byte blocks and 4096-byte pages -- realistic for actual storage hardware.

## Benchmarking Workflow

The intended workflow for evaluating a new layout method:

1. Import a SNAP dataset into the database
2. Run all queries on the natural (insertion-order) layout, logging accesses
3. Apply the layout method to rearrange records
4. Optionally sort the incidence lists
5. Re-run all queries, logging accesses
6. Compare block access counts and visualize access sequences

The visualization tools (Python with matplotlib/numpy) produce bar charts comparing total I/O across methods and line charts showing 50-access windows to reveal whether access patterns are sequential or jumping.

## Lessons from the Implementation

A few interesting engineering observations from the codebase:

**Implicit addressing is powerful.** Because record IDs directly encode file offsets (ID * record_size = byte offset), there's no need for an index to find a record. This is a key advantage of fixed-size records and is exactly how Neo4j works. The downside: moving a record changes its ID, so all references to it must be updated.

**Doubly-linked incidence lists are expensive to maintain but cheap to traverse.** Each relationship participates in *two* incidence lists simultaneously (one for the source node, one for the target node). Insertion is O(1) at the head, deletion is O(1) with the ID known. But the pointer-chasing nature of traversal is exactly what makes record order matter so much.

**The Louvain method isn't just for community detection.** Using it as a record layout strategy (group records by community) is creative, but the lack of block-size awareness means communities don't map cleanly to disk blocks. A community of 1000 nodes spans hundreds of blocks with no control over their ordering.

**ICBL's memory problems are fundamental, not incidental.** The hierarchical clustering step requires a pairwise distance matrix that scales as |V|^2. For 300,000 nodes, that's 335 GiB. The original papers didn't address this, making the method impractical for the very scale of data where layout optimization matters most.

## Who Is This For?

The project is designed for three audiences:

- **Researchers** studying graph data layout can implement new methods against the simple interface, benchmark them against existing approaches, and visualize the results.
- **Database developers** exploring graph storage internals can study a clean, instrumented implementation of the core database layers without the complexity of a production system.
- **Students** learning about database architecture can trace the full path from a high-level query (BFS) down through access, cache, and I/O layers to see exactly how a graph traversal translates to disk operations.

The code builds with CMake, compiles with Clang, includes unit tests, and has Doxygen documentation for the I/O and cache modules. It uses only two external libraries (curl for downloading SNAP datasets, zlib for decompression).

If you've ever wondered what happens below the query optimizer -- how bytes are arranged on disk and why it matters -- this is a good place to start looking.

---

**Full document:** [Thesis (PDF)](/files/locality-optimization-graph-db.pdf)

*This post describes the architecture of the Graph Record Layout Research Environment, part of the master's thesis "Locality Optimization for Traversal-based Queries on Graph Databases" by Fabian Klopfer, University of Konstanz, 2020.*
