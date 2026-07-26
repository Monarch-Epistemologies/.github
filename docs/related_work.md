# Related work — the methods lineage behind this line

*An orientation and a reading list. This document is not a survey of the whole field; it
traces the specific line of work, mostly out of Lawrence Berkeley National Lab and Melissa
Haendel's group, that the methods in these projects descend from — text embedding over graph
node text, node2vec with typed walks, link-prediction retrieval, and ontology grounding. Read
the intro for orientation; treat the annotated entries as a queue, ordered roughly from "start
here" to "deeper."*

---

## Why these papers

This line is not inventing its retrieval methods from scratch. The three moves it
measures — (1) embed node text and retrieve by similarity, (2) walk typed edges and run
node2vec, (3) train a link-prediction classifier and retrieve by predicted edges — are the
house methods of the Monarch / LBNL ecosystem. That ecosystem has published both the
*infrastructure* that builds these graphs and the *evaluations* of the ML run over them, and
it has done so across many knowledge graphs, not just Monarch. So the literature does double
duty here: it is prior art for the methods, and it is evidence about how those methods behave
at scale — which is exactly what this work sets out to measure.

The work splits into two threads:

- **Graph ML** — embeddings, link prediction, node classification run *on* the graph. This is
  the direct ancestor of the node2vec / link-prediction retrieval in these projects.
- **LLM-based extraction** — pulling structured knowledge *out of* text and grounding it back
  onto ontologies. The mirror image of embedding-and-retrieval, and the RAG-adjacent thread.

A third body of work — Haendel's group — is less a separate method and more the *substrate*:
the ontologies, the data model, and the graph itself that both threads run on. It gets its own
section because you asked, and because the substrate decisions (Biolink Model, the phenotype
ontologies, cross-species linking) constrain what any retrieval method can even reach.

---

## Thread 1 — Graph ML: embeddings, link prediction, node classification

**KG-Hub — building and exchanging biological knowledge graphs.**
Caufield, Reese, Mungall, Haendel, et al., *Bioinformatics*, 2023.
The umbrella infrastructure. A modular extract-transform-load pattern that produces
Biolink-Model-compliant graphs from any OBO ontology, wired directly to graph-ML tooling for
automated node embeddings, link prediction, and node classification. Read this first: it
explains why the *same* embedding/link-prediction machinery is meant to run across a family of
graphs, and it names the other graphs (KG-COVID-19, KG-IDG, KG-Microbe, rare-disease KGs).
Start here for the big picture of where Monarch sits.
https://pubmed.ncbi.nlm.nih.gov/37389415/

**KG-COVID-19 — a framework to produce customized knowledge graphs for COVID-19 response.**
Reese, Mungall, et al., *Patterns*, 2021.
The worked example of the whole pipeline. They built a COVID-centric KG and used node2vec
embeddings plus trained link-prediction classifiers for drug repurposing. This is essentially
the section-6 method in KG-RAG-Monarch — typed walks → node2vec → link prediction — applied to a
different graph, and it's the cleanest single paper to see the method end-to-end.
https://www.sciencedirect.com/science/article/pii/S2666389920302038

**GRAPE (and its predecessor Embiggen) — the graph-ML library.**
Reese et al. (LBNL). Embiggen was the original node2vec implementation used in KG-COVID-19;
it was folded into GRAPE, the faster graph-processing / embedding library the ecosystem now
uses. If you want to know what "node2vec, in this world" actually means in code — the walk
generation, the hyperparameter search — this is the implementation your own node2vec step is
a hand-rolled cousin of. (Search "GRAPE graph processing embeddings Reese" for the current
paper/repo.)

**Application and evaluation of knowledge graph embeddings in biomedical data.**
*PeerJ Computer Science*, 2021.
A benchmark study: it takes embedding methods and *measures* them on biomedical KGs rather
than asserting they work. Relevant less for any single result than for the posture — the
"runtime-vs-quality numbers drive the decisions" stance the KG-RAG-Monarch README insists on has
direct precedent here. Read it when you want to see how the field frames an evaluation.
https://peerj.com/articles/cs-341/

---

## Thread 2 — LLM-based, ontology-grounded extraction

**SPIRES — Structured Prompt Interrogation and Recursive Extraction of Semantics.**
Caufield, Mungall, et al.
A zero-shot method: give an LLM a user-defined schema and free text, recursively interrogate
it for structure matching that schema, then *ground* every extracted entity against existing
ontologies to assign stable identifiers. This is the conceptual mirror of your retrieval
step — instead of embedding graph node text and retrieving, it pulls structure out of prose
and snaps it back onto the ontology. Read it to see the "other direction" of KG-RAG.
https://www.ncbi.nlm.nih.gov/pmc/articles/PMC10924283/

**OntoGPT — the tool.**
monarch-initiative/ontogpt (GitHub). The package SPIRES lives in; a set of LLM-plus-ontology
extraction tools. Worth cloning and running once to feel how grounding differs from raw LLM
extraction — that difference is the whole point of doing this over a knowledge graph rather
than over free text.
https://github.com/monarch-initiative/ontogpt

**OntoGPT for environmental evidence synthesis.**
*Environmental Evidence*, 2026.
Included precisely because it is *not* biomedical. It applies OntoGPT to ecology/environmental
literature and reports a blunt number (~65% average agreement with human reviewers, varying by
field type). Useful as a reality check on how ontology-grounded LLM extraction actually scores,
and as evidence the methods are meant to travel beyond Monarch's domain.
https://link.springer.com/article/10.1186/s13750-026-00381-0

---

## Thread 3 — Melissa Haendel's group: the substrate

Haendel's lab (TISLab — Translational and Integrative Sciences Lab, now at UNC Chapel Hill,
previously CU Anschutz / OHSU) is the source of the ontologies, the data model, and the graph
itself. Their contribution to "extracting knowledge from KGs" is upstream of any retrieval
method: they decide what the nodes and edges *are*. The recurring ideas to watch for across
these papers are the **Biolink Model** (the typed schema every edge conforms to — the reason
"typed walks" is even a meaningful phrase), **cross-species linking** via gene orthology and
phenotype similarity, and a stack of phenotype ontologies (**HPO, Mondo, uPheno, PHENIO**).

**The Monarch Initiative in 2024: an analytic platform integrating phenotypes, genes and
diseases across species.**
*Nucleic Acids Research*, 2024.
The current reference for the graph this line is built on. Describes the rebuilt platform, the
Biolink-conformant semantic layer (PHENIO), and the integration approach. This is the "what am
I actually retrieving over" paper — read it alongside your own §1 on the corpus boundary.
https://academic.oup.com/nar/article/52/D1/D938/7449493

**The Monarch Initiative in 2019 / 2017 (earlier platform papers).**
*Nucleic Acids Research*, 2019; earlier in *NAR* 2017.
The lineage of the platform. Read only if you want the history of how cross-species
phenotype-to-genotype integration was assembled; the 2024 paper supersedes them for current
structure.
https://academic.oup.com/nar/article/48/D1/D704/5614574

**Biolink Model.**
The data model Monarch (and KG-Hub) adopted: a standardized set of entity categories (gene,
disease, phenotype, …) and predicate types. This is the single most load-bearing choice for
anyone doing *typed* graph ML on Monarch — your typed-walk method is only definable because the
edges carry Biolink predicate types. Follow the citation from the 2024 Monarch paper, or the
biolink/biolink-model repo, for the specification.

**Exomiser / Genomiser — phenotype-driven variant prioritization.**
A long-running Monarch tool that ranks candidate disease variants by combining a variant score
with a *phenotype-similarity* score computed over the ontology graph. It predates embeddings
but is the group's original "retrieve by graph-structured similarity" system — useful contrast
for what similarity meant before it was a learned vector.

**monarchr — an R package for querying biomedical knowledge graphs.**
*Bioinformatics*, 2025.
Recent, practical: programmatic access to the Monarch KG. Worth a look if you want a second
implementation's view of how the graph is traversed and queried, independent of your Python
pipeline.
https://academic.oup.com/bioinformatics/article/41/10/btaf549/8266340

**Why we need all the organisms: an exploration of the Monarch knowledge graph to aid
mechanism discovery.**
arXiv, 2025.
A recent exploration paper arguing the cross-species edges are where the mechanistic payoff is.
Directly relevant to a retrieval system: it's an argument about *which* parts of the graph carry
the signal worth reaching, which is a question about what your retrieval should prioritize.
https://arxiv.org/pdf/2509.18050

---

## Closest sibling system — Phenomics Assistant

**Phenomics Assistant — an interface for LLM-based biomedical knowledge graph exploration.**
O'Neil, Schaper, Reese, Robinson, Haendel, Mungall, et al., bioRxiv, 2024.
The nearest thing to this line out of the same lab, and the sharpest contrast — it answers the
same question (how does an LLM get knowledge out of the Monarch graph) with an almost orthogonal
method. Phenomics Assistant is **agentic and query-time**: the LLM (GPT-4 era) is given the
Monarch API as callable tools and, at question time, decides which calls to make — a keyword
search to resolve an entity to an ID (e.g. "Cystic Fibrosis" → `MONDO:0009061`), then an
associations call to fetch neighbours, then follow-ups. Retrieval *is* the model chaining live
API calls; graph traversal *is* the model choosing the next hop. There is no vector index and no
precomputed embedding at its core — entity lookup is keyword-based, and the paper names "hybrid
keyword + embeddings" as future work it has not built.

This line is the opposite: **index-driven and precomputed**. Node text is embedded offline
and retrieved by similarity; graph structure enters through learned representations (node2vec
over typed walks, link prediction), not through the model deciding to traverse an edge. So the
two sit on a single axis — model-driven flexibility at hosted-LLM cost (Phenomics Assistant)
versus precomputed, measurable, offline retrieval (here). Two further points make it the sibling
worth reading closely:

- The dense-embedding retriever in KG-RAG-Monarch *is* the "hybrid keyword + embeddings" layer
  Phenomics Assistant flagged as missing — so this work is a plausible component of it, not only
  an alternative to it.
- Phenomics Assistant is a built instance of the full RAG loop (retrieval wired to generation)
  that this line deliberately stops short of, which makes it the natural evaluation harness for
  the answer-level and grounding-vs-hallucination questions in [future_directions.md](future_directions.md).

https://www.biorxiv.org/content/10.1101/2024.01.31.578275v1

Companion write-up (readable, non-paper): *Knowledge-backed AI with Monarch*, Monarch Initiative
on Medium.
https://monarchinit.medium.com/knowledge-backed-ai-with-monarch-a-match-made-in-heaven-a8296eec6b9f

*A note on evaluation, which sharpens the contrast.* Phenomics Assistant is a
prototype-demonstration system: the paper evaluates it through qualitative worked examples (the
Cystic Fibrosis walk-through) rather than a quantitative benchmark — no reported accuracy /
precision / recall against a question set, no quantified GPT-4-vs-GPT-3.5 comparison, no measured
hallucination rate — and the repository itself ships no test suite (no `tests/` directory or eval
scripts; its agent machinery lives in the separate `agent-smith-ai` and `oai-monarch-plugin`
repos). Caveat: the bioRxiv full text could not be retrieved directly, so this reads from the
abstract, the companion write-up, and the repo contents; a small in-body table cannot be fully
ruled out. The takeaway for positioning: the closest sibling system from the same lab is
*demonstrated, not measured* — which is exactly the gap this line's evidence-first, runtime-vs-quality
posture exists to fill.

---

## How to read this queue

If you want the fastest path to orientation: **KG-Hub** (the infrastructure and the family of
graphs), then **KG-COVID-19** (the method end-to-end on one graph), then **Monarch 2024** (the
graph you're actually on). That trio gives you the methods lineage and the substrate in three
papers. Everything else deepens one of those three: SPIRES/OntoGPT for the extraction
direction, the PeerJ evaluation for the measurement posture, and the Haendel-group ontology and
cross-species papers for why the graph is shaped the way it is.
