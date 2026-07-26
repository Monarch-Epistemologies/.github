# Results — every method, one substrate, one answer key

*The running scoreboard for the line. Every method below is scored on the **same**
subgraph and the **same** gold set, so the columns are directly comparable. This
page is kept current as methods land; the posts and notebooks that produced each
number are dated and frozen, so where they disagree with this page, this page is
right.*

## The substrate

All numbers are measured on the human-relevance namespace cut of the Monarch KG:
**299,950 nodes, 4,097,434 edges**, keeping all 29,866 MONDO diseases. The
boundary is drawn by namespace rather than by capacity or hop radius, and every
predicate is kept, so no method under comparison is favoured by the cut. The
derivation is in KG-RAG-Monarch's design notebook, §1.

## The gold set

180 questions across three types — a disease's phenotypes, its causative genes,
its recorded treatments — sampled from the graph and answered *from* the graph, so
no answer is hand-labeled. These are **traversal-shaped**: the answer is a
disease's graph neighbours, not a text-similar node.

## Answer recall — four methods

Recall of the answer entities at k=20, except graph traversal, which has no top-k
(a walk returns every neighbour along the picked predicates).

| question type | node-text | triple-text | graph traversal | network (uniform) | network (typed) |
|---|---|---|---|---|---|
| a disease's phenotypes | 0.02 | 0.49 | **0.89** | 0.11 | 0.21 |
| its causative gene | 0.05 | 0.70 | 0.60 | 0.38 | **0.68** |
| its treatments | 0.00 | 0.52 | **0.88** | 0.16 | 0.30 |
| **overall** | 0.02 | 0.57 | **0.79** | 0.21 | 0.40 |

What the columns say:

- **Node-text embedding** is an excellent entity-linker (0.95 anchor recall) and a
  near-useless fact-retriever (0.02). "Scoliosis" is not textually similar to
  "symptoms of Marfan syndrome."
- **Triple-text embedding** clears that baseline more than twentyfold by making
  the *fact* the unit embedded rather than the entity.
- **Graph traversal** wins overall. Its ceiling is entity-linking, not the walk —
  the gold answers are exactly the edges it follows, so every point lost is an
  anchor mispick or a predicate miss (anchor accuracy 0.79).
- **Network embedding** trails on this gold by construction: it is a lossy
  compression of adjacency, scored against exact adjacency. Typing the walks to
  disease-phenotype/gene/drug metapaths roughly doubles it.

The standout cell is genes under typed network embedding (0.68), which beats
exact traversal (0.60). Traversal commits to one anchor and is punished for
disease-subtype ambiguity; the structural embedding returns a neighbourhood and is
forgiven it.

## Link prediction — the soft-retrieval test

The gold above scores only edges that exist, which the structural embedding cannot
win. This test removes 400 disease-neighbour edges, retrains on the reduced graph,
and asks whether the disease still ranks a target it was never trained to connect
to. A hit is genuine prediction, not recall of a memorised edge.

| held-out edge | hit@20 | hit@100 | hit@250 |
|---|---|---|---|
| disease → its gene | 0.22 | 0.39 | **0.51** |
| disease → a phenotype | 0.02 | 0.06 | 0.09 |
| **overall** | 0.12 | 0.22 | 0.30 |

Genes predict far better than phenotypes across all three measurements: a gene is
a specific, low-degree node whose structural fingerprint survives the loss of one
edge; a phenotype is a generic hub. The structural embedding is a gene method that
also sees phenotypes.

## Entity linking — which embedding model

2,000 synonym queries against a 20,000-name gallery, identical sample per model.
The gallery holds node **names** only, so the queried synonym is never in the text
being matched. MRR by namespace:

| namespace | MiniLM | BioLORD | MedCPT | SapBERT |
|---|---|---|---|---|
| **overall** | 0.542 | 0.560 | 0.592 | **0.716** |
| HGNC (genes) | 0.19 | 0.19 | 0.23 | **0.39** |
| PR (proteins) | 0.09 | 0.14 | 0.26 | **0.42** |
| MONDO (disease) | 0.45 | 0.47 | 0.48 | **0.70** |
| CHEBI (chemicals) | 0.46 | 0.49 | 0.48 | **0.64** |
| HP (phenotype) | 0.76 | 0.88 | 0.79 | **0.93** |
| GO (gene function) | 0.73 | 0.80 | 0.80 | **0.92** |
| UBERON (anatomy) | 0.66 | 0.67 | 0.70 | **0.77** |
| OBA (attributes) | 0.91 | 0.92 | 0.93 | **0.98** |
| UPHENO | 0.93 | 0.80 | 0.91 | **0.93** |

SapBERT takes every namespace, and the gap it closes is concentrated in genes and
proteins — where a synonym is an alias like `PARK2` with no meaning on its
surface. The training objective, not being biomedical as such, is what separates
the models: SapBERT is the only one trained directly on concept synonymy.

*Caveat: MedCPT is an asymmetric query/document retriever and this test runs its
query encoder on both sides, so its number is a floor. It does not change the
ranking.*

## One pipeline, two models

The crawler uses **SapBERT** to find the entity and **MiniLM** to pick the
predicate. SapBERT is trained on term-to-term synonymy and packs whole *sentences*
into a narrow cosine band, scoring 63–72% on predicate selection; MiniLM — the
floor for entity linking — gets the same task at 100%. Different jobs, different
winners, and the steps are independent, so each uses the model that wins it.

## Hardware cost

Measured on a fanless M3 MacBook Air, 16 GB.

| step | cool | thermally throttled |
|---|---|---|
| SapBERT node embedding | ~400 docs/sec | ~110 docs/sec sustained |
| 300k node corpus | ~12 min | ~50 min |

Throttling begins after roughly the first 60,000 documents. External cooling
(case off, fan on bare aluminium) holds the throttled rate steady around 100/sec
rather than lifting it back toward 400 — a fanless chassis cannot be cooled past
the rate at which it sheds heat.

## What is not yet measured

Named here so the gaps are visible rather than implied:

- **Lexical retrieval (Solr/BM25).** No lexical arm exists, so no hybrid
  lexical-vector comparison is possible yet.
- **Ontology semantic similarity (Resnik / information content, OWLSim /
  semsimian / OAK).** Monarch's own signature line of work, and the mode of the
  five with no implementation here at all.
- **Translational KG embeddings** — TransE, RotatE, ComplEx, DistMult. Everything
  built so far is the random-walk branch of shallow graph representation.
- **Graph neural networks** — GraphSAGE, R-GCN, GAT.
- **Rank fusion** across the methods above.
- **Query-category breakdown** for entity linking (abbreviations, spelling
  variants, lay-language, paraphrase) — currently only graph-recorded synonyms,
  split by namespace.
- **Query latency and index size.** Throughput and corpus memory are measured;
  per-query latency and a built ANN index are not.
