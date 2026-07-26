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

## Start here

- **[Epistemologies of Retrieval](retrieval_epistemologies.md)** — the central
  argument. Names five modes of knowing (curated assertion, deductive inference,
  semantic similarity, text embedding, network embedding) and maps three Monarch
  systems onto them.
- **[Results](results.md)** — the running scoreboard. Every method measured so
  far, on one substrate and one answer key, plus an explicit list of what is not
  yet measured.

## The argument

- **[Use Cases](use_cases.md)** — concrete tasks with stakeholders, and which
  retrieval architecture each one needs. Argues that hypothesis and definition
  output shapes are what a curated tool menu structurally cannot produce.

## The plan

- **[Research Plan Integration](research_plan_integration.md)** — a five-phase
  comparison of text and graph representations, mapped against what is already
  built, measured, or missing.
- **[Positioning & Future Directions](future_directions.md)** — where this work
  sits against the literature, and the next measurements worth making.

## Background

- **[Related Work — Methods Lineage](related_work.md)** — the LBNL / Haendel-group
  line the methods here descend from, as an annotated reading queue.
- **[Related Work: Phenomics Assistant](related_work_phenomics_assistant.md)** —
  how Shawn O'Neil's Monarch chatbot reaches the graph, and why its tool-calling
  approach is the comparison point for this work.

## Projects

- [KG-RAG-EDS](https://github.com/Monarch-Epistemologies/KG-RAG-EDS) — v1, hand-built
  on a single disease. Text-embedding retrieval: embed node text and the question
  into vectors, anchor to a node, walk the graph with SQL.
- [KG-RAG-Monarch](https://github.com/Monarch-Epistemologies/KG-RAG-Monarch) — v2,
  the same shape scaled toward the whole Monarch graph, with three retrieval
  methods compared on a shared gold set. Its
  [design notebook](https://github.com/Monarch-Epistemologies/KG-RAG-Monarch/blob/main/doc/substack_draft.md)
  is the running record of why each decision was made.
