# Epistemologies of retrieval: how three Monarch systems "know"

All three systems below answer questions from the *same* curated Monarch graph,
and all three narrate the final answer with an LLM. They differ in the
epistemology of the **retrieval** step — what grounds the claim "this fact is
relevant to your question," and whether that grounding is symbolic and
inspectable or learned and opaque. This doc names the modes of knowing in play
and maps each system onto them.

## The three systems

- **Phenomics Assistant** — Shawn O'Neil's tool-calling chatbot
  (`monarch-initiative/phenomics-assistant`). An LLM selects among a curated menu
  of REST endpoints that query the Monarch API. Details:
  [`related_work_phenomics_assistant.md`](related_work_phenomics_assistant.md).
- **KG-RAG-EDS (text-embedding retrieval)** — this project. Embeds node text and
  the question into vectors, anchors the question to a node and picks predicates
  by vector proximity, then walks the graph with SQL. Details:
  [`project_2_KG_RAG.md`](https://github.com/Monarch-Epistemologies/KG-RAG-EDS/blob/main/doc/project_2_KG_RAG.md).
- **Network-embedding retrieval** — the design in the shelved reasoning-methods
  comparison project (README → Related). Embed the graph's *topology* (node2vec /
  TransE / GraphSAGE family) into vectors and retrieve or infer by proximity in
  that learned structural space.

## Five modes of knowing

1. **Curated assertion.** The graph's edges: triples with provenance — a curator
   or an ingested database committed "EDS has phenotype joint hypermobility,"
   with an evidence code and a citation. Justified, discrete, human-auditable.
   You can ask *why is this here* and get an answer. The OBO / Semantic Web
   tradition of knowledge as explicit cited claim.

2. **Deductive inference.** Because the graph is ontologically grounded (MONDO
   subclasses, the HPO DAG), some facts are entailed rather than asserted: a
   phenotype of an EDS subtype inherits up the is-a hierarchy by logic.
   Knowing-by-consequence — deterministic, explainable, and it yields facts
   nobody wrote down.

3. **Semantic similarity** (your Mode 3). Similarity between two entities (e.g.
   phenotype sets) computed from the *ontology's own structure* — information
   content of their common ancestors in the DAG (OwlSim / Phenomizer /
   semsimian). Symbolic, deterministic, and *explainable*: you can name the
   shared ancestor that makes two things count as close. Note the term is
   overloaded — in general NLP "semantic similarity" usually means embedding
   cosine, which is mode 4, the opposite epistemology. Here it means the
   ontology-structural measure.

4. **Text / semantic embedding** (your Mode 4). Embed node *text* — labels,
   synonyms, descriptions — into vectors with a language model; "about" means
   geometrically near. Learned and distributional; continuous and graded (a
   cosine score); *not* explainable the way modes 1–3 are — you can only say "the
   model placed them near."

5. **Network embedding.** Embed the graph's *connectivity* into vectors
   (node2vec's random walks, TransE-style translational KG embeddings, GraphSAGE
   message-passing). Like mode 4 it is learned, continuous, and opaque — but the
   signal is topology, not text, and its defining move is different: it is
   **inductive**. It predicts links the graph never asserted.

Two adjacencies worth holding onto:

- **3 vs 5 — same source, different epistemology.** Both derive from graph
  structure, but semantic similarity measures closeness along *explicit* ontology
  paths and can name the common ancestor; network embedding infers linkage from
  the *whole* connectivity pattern and cannot say why. Symbolic-explainable vs
  learned-opaque, from the same raw material.
- **4 vs 5 — same machinery, different source.** Both are learned vector spaces
  compared by proximity; they differ only in whether the training signal is node
  text or graph topology.

Sitting on top of all five is a sixth, generative layer: the **LLM that writes
the answer**, which is distributional and can hallucinate. RAG exists precisely
to constrain that layer by feeding it retrieved facts — trading the model's
fluent-but-ungrounded knowing for grounded-but-narrower knowing. Every system
here does that; they differ in how retrieval finds what to feed it.

## These modes already exist in the Monarch orbit

None of the five is invented here. Each has a Monarch-adjacent implementation
already, so the honest claim for this org is the *comparison* — the modes side by
side, on one substrate, by hand — not the modes themselves.

- **Mode 1, curated assertion** — the Monarch KG itself, reached through the
  Monarch API, `monarch-py`, and KGX dumps. Phenomics Assistant retrieves it by
  tool-calling.
- **Mode 2, deduction** — MONDO / HPO / uPheno are OWL, reasoned with ROBOT / ELK
  in the ontology release pipelines; Monarch precomputes subclass closures so
  association queries inherit up the hierarchy.
- **Mode 3, semantic similarity** — Monarch's signature line of work: OwlSim →
  Phenomizer → Exomiser's phenotype scoring → **semsimian** (the Rust
  reimplementation in the current app) and **OAK** (Ontology Access Kit).
  Phenomics Assistant exposes it as `phenotype_profile_search`.
- **Mode 4, text embedding** — **CurateGPT** (Mungall et al.) indexes ontologies
  in a vector DB for RAG; **DRAGON-AI** applies RAG to ontology generation;
  **OntoGPT / SPIRES** grounds free text to ontology IDs. This is the mode the
  KG-RAG line sits in, and CurateGPT overlaps directly with its text channel.
- **Mode 5, network embedding** — **GRAPE** (node2vec at scale, benchmarked on a
  PheKnowLator-built biomedical graph), **NEAT** (Network Embedding All the
  Things), and **KG-Hub** (Reese, Caufield, Mungall et al.), applied to link
  prediction, drug discovery, rare-disease link prediction, and variant
  prioritization.

What none of these projects does is run *all* the modes against the *same*
subgraph on the *same* questions and compare how each one knows — which is exactly
the reasoning-methods comparison this line is building toward. The contribution is
the like-for-like comparison, not any single technique.

## Where each system puts the neural seam

**Phenomics Assistant — neural controller over a symbolic query engine.** Once
the LLM emits `get_disease_phenotype_associations(disease_id=MONDO:…)`, the
endpoint runs a fixed filter (`category=biolink:…, entity=<id>`) and returns that
category's associations exactly and exhaustively — pure mode 1, deterministic and
complete. The only distributional decision is *which* endpoint and *which* id to
pass — LLM menu-selection over a small discrete space. Entity-linking is handed to
Monarch's `search`, which is lexical IR (string/synonym matching with ranking),
not embeddings; and `phenotype_profile_search` exposes mode 3. No learned vector
retrieval anywhere. Characteristic failure: a **routing error** — wrong endpoint
or wrong id — discrete and often catchable. Reach: only pre-enumerated paths.

**KG-RAG-EDS — neural retrieval, symbolic traversal.** Mode 4 at the entry (anchor
and predicate chosen by text-embedding proximity) feeds mode 1 on the graph (SQL
over the edges table), then the LLM narrates. The uncertainty moves from "which
of nine buttons" to "which of thousands of nearby vectors," and it is graded, not
discrete. Characteristic failure: an **anchoring error** — landing on the wrong
node because it was geometrically near — which is exactly what the degree-based
disambiguation in project 2 patches. Reach: any path the graph asserts, without
hand-writing an endpoint. Note the traversal is still mode 1 — once on the graph
it only ever returns *asserted* edges.

**Network-embedding retrieval — learned structure throughout.** Retrieval and
reasoning happen in a vector space induced from the graph's topology (mode 5).
Knowing-by-latent-structure: two nodes are treated as related if the geometry
predicts a link, whether or not one is asserted. This is the only system here
whose retrieval can produce a claim that is *not already in the graph*.

## Truth conditions and failure modes

| mode | a claim is "known" when… | characteristic failure |
|---|---|---|
| 1 curated assertion | someone asserted it with evidence | omission — curation gaps, staleness |
| 2 deduction | it is entailed by the axioms | wrong / oversimplified axioms |
| 3 semantic similarity | it is structurally close in the ontology | ontology structure ≠ biology |
| 4 text embedding | language usage places the terms near | near-in-text ≠ same entity (anchor error) |
| 5 network embedding | the topology makes the link plausible | spurious predicted edges (false positives) |

## The epistemology of network embedding, expanded

Modes 1–4 all, in the end, *return knowledge that already exists* in the graph —
asserted (1), entailed (2), or found by a proximity measure that ranks existing
nodes/edges (3, 4). Retrieval surfaces what is there. Network embedding breaks
that frame: it generalizes from the observed distribution of edges to the ones
*likely to exist but unwritten*. The epistemic posture shifts from "what is
asserted / entailed / close" to "what the latent geometry predicts should hold."

That is genuinely inductive knowledge — closer to hypothesis than to lookup. Its
costs are the costs of induction: it is not explainable (no common ancestor to
point at, only training dynamics), and it will predict edges that are plausible
but false, so its output is *candidate* knowledge that needs validation, not
answers. Its payoff is the thing no other mode offers — it can propose links the
curators never made.

That maps directly onto the use cases (see
[`use_cases.md`](use_cases.md)):

- **Lookups, and diagnosis from a known phenotype profile** — modes 1 and 3.
  Phenomics Assistant is well-matched; nothing here needs learned retrieval.
- **Computational phenotyping and cohort discovery** — curated crosswalks
  (LOINC2HPO / OMOP2OBO, mode-1-flavored) plus mode 3/4 to fill mapping gaps,
  then mode-1 traversal. KG-RAG-EDS's seam fits.
- **Drug repurposing / mechanism hypothesis** — wants mode 5. The task is to
  propose a *disease → … → drug* path that is plausible but *not asserted*. This
  is why a tool-calling menu (mode 1 only) structurally cannot do it — and, more
  subtly, why even KG-RAG-EDS only partially can: its entry is learned (mode 4)
  but its traversal is mode 1, so it can only walk edges that already exist. To
  generate a genuinely new candidate link you need inductive structure — network
  embedding or knowledge-graph completion. This is the epistemic reason the
  shelved reasoning-methods project and use case 3 belong together.

## Sources

- Phenomics Assistant: `github.com/monarch-initiative/phenomics-assistant`
- Semantic similarity (mode 3): OwlSim / Phenomizer / semsimian / OAK (Ontology
  Access Kit) — Monarch's ontology-based phenotype comparison; Exomiser applies it
- Text embedding (mode 4): CurateGPT (Mungall et al., arXiv 2411.00046);
  DRAGON-AI (arXiv 2312.10904); OntoGPT / SPIRES
- Network embedding (mode 5): GRAPE (arXiv 2110.06196), NEAT, KG-Hub (Reese,
  Caufield, Mungall et al., arXiv 2302.10800) as Monarch-orbit implementations;
  node2vec (Grover & Leskovec 2016), TransE (Bordes et al. 2013), GraphSAGE
  (Hamilton et al. 2017) as the underlying methods
