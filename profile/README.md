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

Full treatment: [Epistemologies of retrieval](https://monarch-epistemologies.github.io/retrieval_epistemologies/).

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

Rendered site: **<https://monarch-epistemologies.github.io/>**
(source: [monarch-epistemologies.github.io](https://github.com/Monarch-Epistemologies/monarch-epistemologies.github.io))

- [Retrieval epistemologies](https://monarch-epistemologies.github.io/retrieval_epistemologies/) — the five modes, and how each system knows.
- [Results](https://monarch-epistemologies.github.io/results/) — the running scoreboard: every method measured so far on one substrate and one answer key, and what is not yet measured.
- [Use cases](https://monarch-epistemologies.github.io/use_cases/) — computational phenotyping, cohort discovery, drug repurposing.
- [Research plan integration](https://monarch-epistemologies.github.io/research_plan_integration/) — the five-phase comparison, mapped against what is built, measured, or missing.
- [Positioning & future directions](https://monarch-epistemologies.github.io/future_directions/) — where this sits against the literature, and the next measurements worth making.
- [Related work — methods lineage](https://monarch-epistemologies.github.io/related_work/) — the LBNL / Haendel-group line these methods descend from.
- [Related work: Phenomics Assistant](https://monarch-epistemologies.github.io/related_work_phenomics_assistant/) — the external tool-calling comparison point.
