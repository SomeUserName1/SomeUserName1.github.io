---
title: "Discovering Hidden Taxonomies: Automatic Concept Hierarchy Formation in Property Graphs"
date: 2020-09-01T00:00:00+01:00
tags: ["AI Project Summary"]
summary: "How an unsupervised clustering pipeline can extract meaningful type hierarchies from graph databases — and why that matters for query optimization."
---
> *This post was AI-generated from the project's source code, thesis, and documentation. It is an automated summary, not original writing.*

## The Problem: Implicit Structure Hiding in Plain Sight

Think about a database of businesses. Every node carries the label "Business" and a bag of community-assigned tags: *restaurant, Italian, cafe, WiFi, shopping, mall*. A human glancing at the data immediately sees a taxonomy --- restaurants branch into Italian, Vietnamese, Thai; cafes split by amenity; shopping venues by price range. But the database has no idea this hierarchy exists.

This isn't an edge case. Relational databases store rows in tables with no notion of sub-types across or within relations. Property graph databases like Neo4j allow flexible labeling and rich properties, but still have no built-in mechanism for automatically surfacing the latent type hierarchies embedded in the data. The hierarchy is there, implicit in patterns of labels, properties, and graph structure --- it just needs to be extracted.

This project, carried out as a bachelor thesis at the University of Konstanz's Database & Information Systems group, tackles exactly that: **given a property graph, automatically infer a concept hierarchy (taxonomy) from node labels, properties, and structural features of the graph.**

## Why Bother?

The motivation goes beyond tidy data organization. A well-formed concept hierarchy has concrete applications:

- **Cardinality estimation in query optimization.** If the database knows that "Italian restaurant in Manhattan" is a sub-concept containing roughly 3% of all business nodes, it can make much better decisions about join ordering and index selection. Instead of the uniformity assumption (treating all businesses as equally likely to match any predicate), the query optimizer can look up the concept tree and retrieve a probabilistic estimate.
- **Data exploration.** Analysts get a structured overview of what kinds of entities exist and how they relate, without manually sifting through millions of records.
- **Schema enrichment.** The inferred hierarchy can suggest more expressive labels or types that make queries more precise.

## The Approach: A Two-Part Pipeline

The project unfolds in two stages, each building on the last.

### Part 1: Surveying Hierarchical Clustering Algorithms

Before committing to a single method, a broad survey was conducted. Sixteen clustering algorithms were benchmarked --- spanning partition-based (k-Means, TTSAS), density-based (DBSCAN, OPTICS, HDBSCAN), hierarchical (Single Linkage, Robust Single Linkage), and model-based (Trestle/Cobweb) approaches. Implementations from scikit-learn, PyClustering, and the concept_formation library were used, with a slightly patched version of HDBSCAN from scikit-contrib. (The author even found and fixed a bug in scikit-learn along the way.)

**Synthetic data** with a known ground-truth hierarchy served as the evaluation benchmark. A custom Java generator produces tag sets that encode a tree of configurable width and depth, with tunable noise levels (0% to 33% of tags randomly added, removed, or altered). Quality was measured using **Tree Edit Distance (APTED)** between the inferred and ground-truth hierarchies.

Key findings from the survey:

- **Density-based methods** (DBSCAN, HDBSCAN, OPTICS) are robust to noise but rarely reconstruct the hierarchy exactly.
- **Single Linkage** is fundamentally unsuited --- it merges one pair at a time, producing bloated dendrograms far from the target taxonomy.
- **Robust Single Linkage** dramatically improves on classic Single Linkage by allowing multi-object merges in dense regions.
- **Trestle** (a Cobweb variant) is the only algorithm whose quality *improves* with hierarchy depth and *degrades* with noise, because it is the only one that dynamically adjusts tree depth during clustering. For deep hierarchies with low noise, it outperforms density-based methods.
- **Non-hierarchical methods** (k-Means, TTSAS, DBSCAN) can be composed with hierarchical agglomerative clustering in a two-step pipeline: first cluster flat, then build a hierarchy over cluster representatives. This trades some information loss for much better scalability.

A post-processing step converts the dendrogram produced by agglomerative clustering into a usable taxonomy by collapsing consecutive merges at the same distance into single branching points.

### Part 2: Going Graph-Aware with Cobweb in Neo4j

Armed with survey results, the project narrows its focus. **Cobweb** --- Fisher's 1987 conceptual clustering algorithm --- was selected for the graph-aware pipeline because of several compelling properties:

1. **Parameter-free.** No k to guess, no epsilon to tune. The category utility heuristic drives all decisions.
2. **Incremental.** Nodes are integrated one at a time, making it naturally suited to databases where data arrives over time.
3. **Probabilistic concept descriptions.** Each node in the concept tree stores attribute-value distributions, not just a cluster label. This directly supports cardinality estimation.
4. **Mixed data support.** Cobweb/3 extends the category utility to handle both nominal (labels, tags) and numeric (degree, coordinates) attributes using Gaussian modeling for continuous values.
5. **Linear memory.** O(n) space, no distance matrices required.
6. **Native hierarchy.** The output is a tree of variable branching factor --- not a dendrogram that needs flattening.

The algorithm was implemented from scratch in Java as a **Neo4j user-defined procedure**, callable directly from Cypher:

```cypher
MATCH (n)
WITH COLLECT(n) AS nodes
CALL kn.uni.dbis.neo4j.conceptual.PropertyGraphCobwebStream(nodes)
YIELD node, partitionID
RETURN node, partitionID
```

#### Feature Vectors: What Makes a Node "Similar"?

The crucial design decision is *what information to feed the clustering*. Several combinations were tested:

| Feature Set | What It Captures |
|---|---|
| **Labels only** | Node type identity (e.g., Person, Message, Post) |
| **Labels + Properties** | Attribute values (browser, creation date, coordinates...) |
| **Labels + Characteristic Set** | The set of relationship types touching the node |
| **Labels + Structural Features** | Node degree, average neighbor degree, ego-net in/out edges |
| **All of the above** | The full picture |

Feature extraction code walks the graph via Neo4j traversals, computing degree, average neighbor degree, ego-net edge counts, and characteristic sets (the set of all relationship types for a node, inspired by Neumann & Moerkotte's work on RDF cardinality estimation).

#### Results on Real Datasets

Three datasets were used for graph-aware evaluation:

**LDBC Social Network Benchmark** --- a synthetic but realistic social graph with Person, Message, Post, Forum, Tag, and Organization nodes. With labels only, Cobweb cleanly separates Comments from Posts at the top level. Adding structural features reveals further sub-structure: high-degree messages (many tags and replies) vs. low-degree ones. The characteristic set adds relationship-type awareness, distinguishing tagged from untagged content.

**Yelp Business Dataset** --- a property-heavy, graph-light dataset where all nodes share the single label "Business." Labels alone produce a trivial single-node tree. Adding properties enables clustering by attributes (location, price range, categories), though the sheer diversity of properties introduces noise. This dataset highlights a fundamental tension: more features mean more information but also more noise.

**New York Road Network** --- a pure graph with no labels or properties, only spatial structure. Only structural features are informative here. Cobweb successfully separates dead ends (degree 1), T-intersections (degree 3), and full crossings (degree 4), further subdividing by ego-net density.

## The Central Insight: No Universal Feature Vector

Perhaps the most important takeaway is that **there is no single feature vector that works for all property graphs.** The right combination depends entirely on what information the data actually carries:

- Property-oriented datasets (like Yelp) gain nothing from structural features.
- Pure graphs (like road networks) gain nothing from labels or properties.
- Adding properties to a graph-rich dataset (LDBC) can actually *hurt* by drowning out the signal from labels and structure with noisy, weakly-predictive attributes.

This suggests that an adaptive approach --- detecting which features carry discriminative information and weighting or selecting accordingly --- is essential for a general-purpose solution.

## Under the Hood: The Category Utility

At the heart of Cobweb is the **category utility**, a measure introduced by Gluck and Corter rooted in cognitive psychology. It quantifies how much knowing an instance's category improves your ability to predict its attribute values. Formally, it measures the increase in the expected number of correctly guessed attribute-value pairs given access to the partition, versus guessing without it.

The category utility naturally balances intra-cluster similarity (members of a concept share attribute values) and inter-cluster dissimilarity (different concepts have different attribute distributions). It uses probability matching rather than probability maximization --- a design choice inspired by ecological rationality research showing that in competitive, non-deterministic environments, matching the underlying distribution outperforms always picking the mode.

Cobweb treats clustering as a search through three interleaved spaces --- characterizations, aggregations, and hierarchies --- using four operators: classify into existing class, create new class, merge two classes, split a class into its children. Hill climbing on category utility drives the search, with merge and split providing bidirectional exploration to recover from local optima.

## Technical Details

The implementation stack:

- **Python pipeline** for the algorithm survey: scikit-learn, PyClustering, scipy, concept_formation (Trestle), with custom wrappers for parameter optimization via randomized search using the silhouette coefficient.
- **Java implementation** of Cobweb for Neo4j: implements the Cobweb/3 category utility with Chan's algorithm for numerically stable online mean/variance computation. Exposes a Cypher-callable procedure via the Neo4j procedure API.
- **Synthetic data generator** in Java: produces hierarchical tag sets with configurable width, depth, and noise.
- **Evaluation**: Tree Edit Distance (APTED) for quantitative comparison; visual inspection of inferred concept trees and their probabilistic descriptions.

The whole pipeline ran on a dual-socket AMD EPYC 7531 system with 512 GB RAM.

## Future Directions

The thesis identifies several promising avenues:

- **Bayesian extensions** like AutoClass or learning Bayesian networks to capture conditional selectivities between features --- moving from point estimates to full posterior distributions over concept membership.
- **Adaptive feature selection** that automatically identifies the most discriminative attributes, potentially using information gain or mutual information.
- **Integration into the query optimizer** as a cardinality estimator, replacing or augmenting histogram-based approaches with concept-tree lookups.
- **Handling spatial and temporal data types** with appropriate distance semantics rather than treating them as generic numerics.
- **LABYRINTH** and other modern extensions of conceptual clustering that address ordering sensitivity and improve convergence.

## Closing Thoughts

This project sits at a compelling intersection: cognitive science (concept formation), database internals (cardinality estimation), and unsupervised learning (hierarchical clustering). The core idea --- that the implicit type structure of a database can be automatically surfaced and leveraged --- is both practically useful and theoretically rich.

The Cobweb algorithm, despite being from 1987, turns out to be remarkably well-suited to this task. Its parameter-free nature, incremental operation, and probabilistic concept descriptions align almost perfectly with what a database system needs. The challenge, as the results show, is less about the algorithm itself and more about engineering the right feature representation for each dataset.

---

**Full documents:** [Thesis (PDF)](/files/Label_Hierarchy_Inference_in_Property_Graph_Databases-thesis.pdf) · [Presentation (PDF)](/files/Label_Hierarchy_Inference_in_Property_Graph_Databases-presentation.pdf)

The code is open source and available in the project repository, including the survey pipeline, the Neo4j procedure, the synthetic data generator, and all evaluation tooling.
