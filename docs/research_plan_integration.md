# Integrating the five-phase research plan with the existing line

*How "Comparing Text and Graph Representations for Biomedical Knowledge Retrieval"
maps onto what KG-RAG-EDS, KG-RAG-Monarch, and the org-level epistemology docs
already built and measured. Written to sit alongside
[`future_directions.md`](future_directions.md) and
[`related_work.md`](related_work.md): those two ask where this line goes next;
this one checks the plan's five phases against what already exists, phase by
phase, so effort goes to the real gaps instead of re-deriving what's already
measured.*

## The headline: the plan and this line are already the same project

The plan's framing — evaluate each retrieval stage independently, don't credit
an LLM-plus-KG pipeline for gains that belong to one component, decompose
"performance" into entity identification / graph retrieval / learned
representation / GNN / combined — is, almost sentence for sentence, the stance
[`retrieval_epistemologies.md`](retrieval_epistemologies.md)
already argues under a different vocabulary. That doc names five "modes of
knowing" (curated assertion, deduction, semantic similarity, text embedding,
network embedding) plus a sixth generative layer (the LLM that narrates the
answer) that sits on top and is explicitly held separate from retrieval
quality. The plan's "Role of LLMs" section — LLM query interpretation and
answer generation are a separate HCI/tool-use problem, not evidence for a
retrieval method — is the identical position this org's docs already take.

So the two don't need reconciling on philosophy. What the plan adds is a
five-*phase build sequence* (identify entity → traverse → shallow embed → GNN →
combine) where the existing docs had named five *modes* without committing to
an order. The phase numbering below is the plan's; each section says what's
built, what's measured, and what's genuinely missing.

## Phase 1 — Text-to-entity identification

**Built and measured.** This is exactly what `bin/embed_corpus.py`,
`bin/embed_models.py`, and `bin/eval_synonym_retrieval.py` already do: four
models compared (MiniLM, BioLORD, MedCPT, SapBERT) against namespaces that
cover the plan's three entity types and more (HGNC/genes, MONDO/disease,
HP/phenotype, plus PR, CHEBI, GO, UBERON, OBA, UPHENO). The gene-alias case the
plan calls out by name — "gene aliases" as a query category — is the literal
example the eval writeup uses to explain why SapBERT wins (`PARK2` as an alias
with no surface meaning). SapBERT wins every namespace; the gap it closes is
concentrated exactly in genes and proteins (HGNC 0.19→0.39 MRR, PR 0.09→0.42),
which is the plan's Phase 1 result already produced, just not yet organized
under the plan's category taxonomy.

Two things are closer to done than they look: `bin/eval_synonym_retrieval.py`
already computes Recall@1/5/10 (`K = [1, 5, 10]` in the source), it's just not
surfaced in `eval/README.md`, which reports only MRR. Pulling those numbers
into a table is a reporting task, not a build task.

**Missing, and these are the real Phase 1 gaps:**

- **Solr-based lexical retrieval.** Nothing in either repo runs a lexical/BM25
  index — dense embedding is the only retrieval channel tested. The plan's
  "hybrid lexical-vector method" step has no lexical arm to hybridize with yet.
- **The query-category taxonomy.** The plan wants queries split into exact
  labels, synonyms, aliases, abbreviations, spelling variants, lay-language,
  biomedical paraphrases, and phenotype descriptions, scored separately.
  `eval_synonym_retrieval.py` scores one category (graph-recorded synonyms) by
  namespace; it doesn't yet distinguish an alias from an abbreviation from a
  lay paraphrase. Building that taxonomy — probably as a labeled slice of
  `eval/synonym_queries.jsonl` plus a small hand-built lay-language/paraphrase
  set, since the graph doesn't hand those out for free the way synonyms do —
  is the sharpest Phase 1 gap.
- **Latency, memory, index size.** The substack notebook measures embedding
  *throughput* (documents/sec) and corpus memory footprint in detail (section
  2), but not query-time latency or a built index's on-disk size — those are
  costed qualitatively (the ANN-index discussion in section 4) rather than
  measured. Reusing the section-4 ANN work would make this closeable quickly.

## Phase 2 — Structured graph retrieval from a known node

**Built and measured, and this is the line's strongest result.** `bin/crawl.py`
(anchor → predicate-pick → disambiguate → traverse), `bin/traverse.py`, and
`bin/classify_predicate.py` are exactly the plan's "explicit one-hop... and
relation-specific queries," scored by `bin/eval_crawl.py` against
`eval/gold_monarch.jsonl`. It's the best-performing method on the traversal
gold (0.79 overall recall against 0.02 for node-text and 0.57 for triple-text),
and — matching the plan's evaluation requirement precisely — the failure modes
are already decomposed into the same four buckets the plan names: anchor
misses (wrong starting node), predicate misses (one bug found and fixed —
`expressed_in`'s description contained the word "gene" and mis-routed gene
questions), genuine multi-hop/subtype ambiguity (Meier-Gorlin's several
gene-distinct subtypes), and ranking within a hop. See `eval/README.md` and
`doc/substack_draft.md` §5 for the full breakdown.

**Missing:**

- **Ontology-aware semantic similarity (mode 3 in the org framing — Resnik /
  information-content over MONDO or HPO).** This is named conceptually in
  `retrieval_epistemologies.md` and attributed to Monarch's own
  OwlSim/Phenomizer/semsimian/OAK lineage, but nothing in either repo runs it.
  It is the one mode of the org's five that has zero code here. Phase 2's
  "ontology-aware semantic similarity" and "information-content measures such
  as Resnik similarity" bullets are asking for exactly this, and it would slot
  in as a fourth crawler-adjacent baseline alongside traversal.
- **Simple structural/degree ranking as an explicit baseline.** The `shape_*.py`
  scripts (`shape_hubs.py`, `shape_relevance.py`, `shape_sparsity.py`, etc.)
  measure graph shape — hub degree, sparsity, relevance cuts — as inputs to the
  *corpus-boundary* decision (substack §1), not as a retrieval baseline in
  their own right. Repurposing degree as a ranking signal for the plan's
  "simple structural ranking methods" bullet would be a small, well-supported
  addition given how much hub-degree data already exists.

## Phase 3 — Shallow graph representations

**Built and measured, but only half the method family the plan names.**
`bin/embed_network.py` implements node2vec-style random-walk embedding via
gensim skip-gram, with two walk schemes: `uniform` (equivalent to DeepWalk —
the p/q return/in-out bias that would make it node2vec proper is explicitly
deferred, noted in `config/network_embed.yaml`) and `metapath` (typed
disease-phenotype/gene/drug walks). `bin/eval_network.py` scores it on the same
gold as Phase 2 (uniform: 0.21 overall recall at N=20; typed: 0.40, roughly
doubling it), and `bin/build_link_gold.py` + `bin/eval_link_prediction.py` run
the harder, more honest test the plan's "link prediction" bullet asks for:
400 held-out disease→gene/phenotype edges, retrained on the reduced graph, hit@k
scored genuinely out-of-sample (disease→gene 0.51 hit@250, disease→phenotype
0.09 — the same gene-favoring inversion seen everywhere else in this line).
This is a well-instrumented, already-published result for the "disease-to-gene
retrieval," "phenotype-to-disease retrieval," and "link prediction" bullets.

**Missing:** the translational/factorization KG-embedding family the plan
names by name — **TransE, RotatE, ComplEx, DistMult** — has no code and no
dependency (`requirements-embed.txt` pulls in `gensim`, not `pykeen` or a
comparable KGE library). Everything built so far is the random-walk
(node2vec/DeepWalk) branch of "shallow graph representations," not the
translational-embedding branch. Adding one of `pykeen` or `torchkge` and
running TransE/RotatE/ComplEx/DistMult against the exact same held-out
link-prediction gold already built (`data/graph_lp.duckdb`) is close to a
drop-in extension — the gold, the corpus, and the hit@k scorer already exist;
only the embedding method itself is missing. "Ranking candidate genes from
phenotype descriptions" (as opposed to from a resolved phenotype node) also
isn't separately tested — the current gold starts from a resolved disease/gene
node, not free-text phenotype descriptions, so this bullet needs a
phenotype-description-to-candidate-gene gold that doesn't exist yet.

## Phase 4 — Graph neural networks

**Not started; this is the plan's real frontier relative to the existing
line.** No GraphSAGE, R-GCN, GAT, or heterogeneous-graph-transformer code
exists anywhere in either repo, and no GNN library (`torch_geometric`, `dgl`)
is a dependency. `torch` itself is already a `KG-RAG-Monarch` dependency
(listed in `requirements.txt`, presumably for the sentence-transformer models),
so adding `torch_geometric` is a small dependency change, not a new stack.

What *does* already exist and is directly reusable: the plan's own
recommendation to start with "a restricted human gene–disease–phenotype
subgraph" is close to what `bin/extract_subgraph.py` already produces (the
human-relevant namespace cut: 299,950 nodes / 4,097,434 edges, or the
coarser taxon-label cut at 442,307 nodes / 4,923,997 edges — see
`doc/substack_draft.md` §1 for the full derivation and why the namespace cut,
not the taxon-label one, is the correct boundary). It would likely need a
further cut down to just the disease/gene/phenotype node types and their
direct predicates to match the plan's "restricted" scope, but the hard
work of deciding *where* the human-relevant boundary sits, and confirming it
doesn't secretly favor one retrieval method over another (the "method-neutral
boundary" argument in §1), is already done and directly inheritable. The same
held-out link-prediction gold from Phase 3 (`data/graph_lp.duckdb`,
`eval/gold_link_prediction.jsonl`) is the natural first evaluation harness for
a GNN, letting Phase 4 be scored against Phase 3's shallow-embedding numbers on
day one rather than needing a new gold built from scratch.

## Phase 5 — Combined retrieval

**Named as the next priority, not yet run.**
[`future_directions.md`](future_directions.md) direction 2 — "fuse the three
retrievers instead of only comparing them," explicitly framed as "a rank-fusion
experiment the large-infrastructure papers rarely isolate" — is this plan's
Phase 5, already queued as the second-highest-priority future direction (after
building the evaluation anchor, which is also already done: `eval/gold_monarch.jsonl`
plus `eval_score.py`/`eval_crawl.py`/`eval_network.py` *is* that anchor). The
raw material for the plan's four named combinations already exists in some
form:

- *text embeddings for entity ID → graph traversal* — this is literally what
  the crawler already does internally (SapBERT anchor → predicate-pick →
  traverse); the "combination" framing would mean varying the anchor step and
  measuring the traversal's sensitivity to it, which the anchor-recall numbers
  already partially expose.
- *Solr candidates → vector rerank* — blocked on the Phase 1 Solr gap above.
- *text retrieval → GNN-based reranking* — blocked on the Phase 4 GNN gap.
- *graph traversal → embedding rerank* — the one combination closest to
  reachable today, since traversal and typed network embedding are both built
  and scored on the same gold; a fusion experiment here needs no new
  components, only a rank-combination script.

## What this means for sequencing

Phases 1–3 are not blank slates; they're partially measured, with specific,
named holes (Solr lexical retrieval, the query-category taxonomy, latency/size
metrics, ontology semantic similarity, and the TransE/RotatE/ComplEx/DistMult
family). Phase 4 is genuinely greenfield. Phase 5 is queued in the line's own
future-directions list and has one combination (traversal + typed network
embedding) buildable immediately from existing artifacts.

If the plan is adopted as the org's forward roadmap, the lowest-effort,
highest-value next steps, in the order the existing evidence-first posture of
this line would argue for, are: surface the Recall@1/5/10 numbers
`eval_synonym_retrieval.py` already computes; build the query-category
taxonomy for Phase 1; add a Resnik/OAK baseline for Phase 2; and run the
traversal + typed-network-embedding fusion for Phase 5, since it requires no
new modeling, only a rank-combination pass over results this line has already
produced. TransE-family embeddings, a Solr index, and any GNN are the three
genuinely new build efforts the plan calls for that nothing here has started.
