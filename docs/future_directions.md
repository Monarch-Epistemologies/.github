# Positioning and future directions

*Where this work sits against the literature collected in [related_work.md](related_work.md),
and the directions worth pursuing next. Every direction below is framed as a **measurement**,
not a feature — in keeping with the line's "evidence before infrastructure" posture. The
positioning is read from the literature plus the KG-RAG-Monarch README and design notebook;
where a claim runs past what is currently implemented, it is marked as a hypothesis to test,
not a description of what exists.*

---

## Where this work sits

Three coordinates locate this line against the three threads in the related-work reading
list.

**Against the graph-ML thread (KG-Hub, KG-COVID-19, GRAPE).** Those are production
infrastructure: cluster-scale ETL, an optimized embedding library, methods asserted to work and
then deployed. This line is the opposite corner — a hand-built, single-laptop
reconstruction of the same three retrieval methods (dense text embedding, node2vec over typed
walks, link-prediction retrieval) whose deliverable is not a benchmark score but the
**runtime-vs-quality cost of those methods on commodity Apple Silicon**. KG-COVID-19 answers
"does node2vec plus link prediction find drug-repurposing candidates." This work answers a
question that literature largely skips: what does it *cost*, patiently, without a cluster, and
where does that break. That is a real and underserved niche — the field publishes the method,
rarely the price on modest hardware.

**Against the extraction thread (SPIRES, OntoGPT).** Those run the arrow the other way — text
*into* grounded graph edges. This line retrieves *from* the graph. They are complementary halves
of a loop that neither side currently closes.

**Against Haendel's substrate (Biolink Model, the phenotype ontologies, cross-species edges).**
This line is a consumer of Biolink typing and the phenotype / cross-species edges, not a
producer. Its typed-walk method is entirely parasitic on Biolink predicate types existing —
worth stating plainly, because it means retrieval quality is bounded by the substrate's edge
semantics, not only by the embedding math.

**One-line position:** it is the measurement study the graph-ML thread skips — the same house
methods, instrumented for cost on a laptop, with intuition-building rather than deployment as
the goal.

---

## Future directions

Ordered by how well each fits the line's own evidence-first ethos — each is framed as a
measurement, not a feature. Directions 1 and 4 are recommended first: 1 gates the
interpretability of everything else, and 4 is KG-RAG-Monarch's stated reason for existing.

### 1. Build the evaluation anchor first

Right now "quality" in runtime-vs-quality has no denominator. The cheapest rigorous one is
already in reach: hold out a set of known Biolink edges (gene–disease, phenotype–gene) and score
each retrieval method as link prediction against them. That turns "quality" from a vibe into a
number and makes every subsequent comparison honest. Highest-leverage step — the other
directions are hard to interpret without it.

### 2. Fuse the three retrievers instead of only comparing them

KG-RAG-Monarch has dense text embeddings, graph structural embeddings, and link-prediction scores over
one corpus. That is a rank-fusion experiment the large-infrastructure papers rarely isolate,
because they commit to a single method. Measure whether fusion beats the best single method, and
by how much per unit of added runtime.

### 3. Test the "all the organisms" claim directly

The 2025 arXiv paper from the Monarch orbit argues the cross-species (orthology,
phenotype-similarity) edges carry the mechanistic signal. That is a falsifiable retrieval
hypothesis: does a retriever that traverses those edges surface answers that dense text embedding
alone misses? This line is well positioned to test it because it controls the walk types.

### 4. Find the scaling wall on purpose

The README's premise is that v3 (Docker / RunPod) is unjustified until a measured step defeats the
Mac. Node2vec walk generation over the full Monarch graph, or embedding the entire node-text
corpus, is the likely wall. Instrument it until there is a number that either justifies v3 or
shows the Mac holds. Not a detour — the thesis.

### 5. Close the loop with SPIRES / OntoGPT (later)

Once the evaluation anchor exists, extract fresh edges from literature with OntoGPT and measure
whether they improve retrieval against the held-out set. The natural next version, but premature
before direction 1.

### 6. Ablate retrieval against the generated answer, not only the retrieved set (last)

The final RAG question is whether better retrieval changes the LLM's answer, or whether ontology
grounding reduces hallucination relative to a plain LLM. Worth doing, but the noisiest
measurement — it belongs last.

Phenomics Assistant (see [related_work.md](related_work.md), "Closest sibling system") is the
natural harness for directions 5 and 6: it is a built instance of the full retrieval-to-generation
loop from the same lab, using model-driven API calls where this line uses precomputed embeddings.
That makes it both a baseline to measure against — does precomputed vector retrieval beat the
agent's keyword lookups on the held-out edge set — and the slot where this line's dense retriever
could be dropped in as the "hybrid keyword + embeddings" layer its authors named as missing.

---

## Recommended order

Do **1** (evaluation anchor) and **4** (find the scaling wall) first: 1 makes every other result
interpretable, and 4 is the measurement KG-RAG-Monarch exists to produce. Treat **2** (fusion)
and **3** (cross-species edges) as the interesting science that becomes measurable once 1 is in
place. Hold **5** (extraction loop) and **6** (answer-level ablation) until the anchor and the
wall are established.
