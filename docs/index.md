# Monarch-Epistemologies

*Modes of machine knowing, explored on one biomedical knowledge graph.*

This organization is a hands-on exploration of how different retrieval and
reasoning architectures "know" things — all grounded in a single substrate, the
[Monarch Initiative](https://monarchinitiative.org) knowledge graph. Each project
instantiates a different mode of knowing and is a complete, working system that
answers questions about disease, gene, and phenotype.

The same graph can be queried under very different epistemologies. What differs
is what counts as "this fact is relevant to the question," and whether that
grounding is symbolic and inspectable or learned and opaque.

## The documents

- **[Epistemologies of Retrieval](retrieval_epistemologies.md)** — the central
  argument. Names five modes of knowing (curated assertion, deductive inference,
  semantic similarity, text embedding, network embedding) and maps three Monarch
  systems onto them.
- **[Use Cases](use_cases.md)** — concrete tasks with stakeholders, and which
  retrieval architecture each one needs. Argues that hypothesis and definition
  output shapes are what a curated tool menu structurally cannot produce.
- **[Related Work: Phenomics Assistant](related_work_phenomics_assistant.md)** —
  how Shawn O'Neil's Monarch chatbot reaches the graph, and why its tool-calling
  approach is the comparison point for this work.

## Projects

- [KG-RAG-EDS](https://github.com/Monarch-Epistemologies/KG-RAG-EDS) — text-embedding
  retrieval: embed node text and the question into vectors, anchor to a node, walk
  the graph with SQL.
- [KG-RAG-Monarch](https://github.com/Monarch-Epistemologies/KG-RAG-Monarch) — the
  second-generation system.
