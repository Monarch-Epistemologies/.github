# Monarch-Epistemologies

*Modes of machine knowing, explored on one biomedical knowledge graph.*

This organization is a hands-on exploration of **how different retrieval and
reasoning architectures "know" things** — all grounded in a single substrate, the
[Monarch Initiative](https://monarchinitiative.org) knowledge graph. It is a
learning program built slowly by hand, not a product. Each project instantiates a
different *mode* of knowing and is a complete, working system that answers
questions about disease, gene, and phenotype.

## The thesis

The same graph can be queried under very different epistemologies. What differs
is what counts as "this fact is relevant to the question," and whether that
grounding is symbolic and inspectable or learned and opaque:

- **Curated assertion** — the graph's cited edges.
- **Deductive inference** — what the ontology entails.
- **Semantic similarity** — closeness in ontology structure (OwlSim / Phenomizer).
- **Text embedding** — closeness of node *text* in a learned vector space.
- **Network embedding** — linkage predicted from graph *topology*.

Full treatment: [docs/retrieval_epistemologies.md](../docs/retrieval_epistemologies.md).

The single substrate is the boundary, not an incidental detail. Holding the graph
fixed is what makes the modes comparable. Exploring a different graph — KaBOB, say
— would be a separate organization rather than another project here.

## Projects (entrypoints)

Each project is a working system and an instance of one or more modes. KG-RAG is
the entrypoint into the wider comparison, not the whole of it.

- **KG-RAG-EDS** — v1, the hand-built teaching version: one disease
  (Ehlers-Danlos), every step done slowly by hand. Text embedding.
- **KG-RAG-Monarch** — v2, the same text-embedding pipeline scaled toward the
  broader Monarch graph, native on Apple Silicon.
- *Semantic-similarity and network-embedding projects — to come.*

## Not from scratch

Most of these modes already have implementations in the Monarch orbit — semantic
similarity (OwlSim, semsimian, OAK), text-embedding RAG (CurateGPT, OntoGPT),
network embedding and link prediction (GRAPE, NEAT, KG-Hub), and tool-calling over
curated endpoints (Phenomics Assistant). What is new here is putting them side by
side, on one substrate, by hand — the *comparison*, not the modes.

## Docs

- [Retrieval epistemologies](../docs/retrieval_epistemologies.md) — the five modes, and how each system knows.
- [Use cases](../docs/use_cases.md) — computational phenotyping, cohort discovery, drug repurposing.
- [Related work: Phenomics Assistant](../docs/related_work_phenomics_assistant.md) — the external tool-calling comparison point.
