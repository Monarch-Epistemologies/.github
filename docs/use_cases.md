# Use cases: what KG-RAG over Monarch is actually for

## Why this doc exists

Shawn O'Neil's Phenomics Assistant (see below) is a working Monarch chatbot, but
it is architecturally *generic*: it answers whatever maps onto a curated menu of
endpoints and produces facts or ranked lists. There is no task in it with a
stakeholder — no one whose job gets done. This doc names concrete tasks, and for
each says which retrieval architecture it needs and why. The recurring point:
the tasks with real users demand output shapes — **hypotheses** and
**definitions** — that a curated tool menu structurally cannot produce.

## The comparison point: Shawn O'Neil's Phenomics Assistant

Canonical repo `monarch-initiative/phenomics-assistant` (his fork
`oneilsh/phenomics-explorer`). Three repos stacked:

1. **`phenomics-assistant`** — Streamlit chat UI.
2. **`agent-smith-ai`** — LLM orchestration. Its `UtilityAgent` lets the model,
   each turn, either answer or call a tool; on a tool call it runs the HTTP
   request, feeds the JSON back into the conversation, and loops.
3. **`oai-monarch-plugin`** — a FastAPI service exposing the Monarch REST API
   (v3) as a small set of LLM-friendly endpoints.

The retrieval mechanism is **LLM tool-calling over a curated menu**, not
embedding retrieval — there are no graph-node embeddings anywhere. The menu is
the files in `oai_monarch_plugin/routers/`: `disease_to_gene` /
`gene_to_disease`, `disease_to_phenotype` / `phenotype_to_disease`,
`gene_to_phenotype` / `phenotype_to_gene`, `phenotype_profile_search`, `entity`,
`search`. That is the biolink association categories among disease/gene/phenotype
enumerated in both directions, plus name→ID resolution and phenotype-profile
similarity. Each endpoint carries a one-sentence `description` and an
`operation_id`; those English sentences *are* the interface the model matches
against. (Code-level detail:
`related_work_phenomics_assistant.md`.)

What that architecture can and cannot produce:

- **Can:** a fact, or a ranked list, for a traversal someone already wrote an
  endpoint for. Precise — the model rarely mis-picks, because each option is a
  hand-written sentence.
- **Cannot:** a traversal that isn't on the menu, a multi-hop composed path, a
  novel hypothesis, or a reusable definition. New capability = new router.

## The axis that sorts use cases

Both architectures must solve the same two sub-problems Project 2 names —
**anchor** (words → a specific node) and **traverse** (follow the right edges).
The menu anchors with its `search` tool and traverses by tool-selection; this
project anchors with embeddings and traverses by predicate-embedding + SQL over
the edges table. The consequence that matters for use cases is **output shape**:

| output shape | example | menu can do it? |
|---|---|---|
| fact | "what gene causes X" | yes |
| ranked list | "diseases matching these phenotypes" | yes (`phenotype_profile_search`) |
| hypothesis | "what drug might treat X, and by what mechanism" | no — multi-hop, not enumerable |
| definition | "a computable cohort for X" | no — composed traversal + value logic |

The three use cases below are picked to land in the bottom two rows.

---

## Use case 1 — computational phenotyping: OMOP measurement → HPO → MONDO

Derive a patient's deep phenotype profile from structured EHR lab data, then rank
candidate diseases. This is Exomiser-style disease ranking driven by labs instead
of a genome.

```
OMOP measurement row              HPO term                     MONDO disease
(LOINC + value + ref range)  →    (phenotypic abnormality) →   (ranked candidates)
        hop 1                          hop 2
```

**Hop 1 is the hard one, and it is not `measurement_concept_id → HPO`.** The same
LOINC analyte maps to *opposite* HPO terms by direction: glucose at 60 vs 300
mg/dL is one concept_id but hypoglycemia (HP:0001943) vs hyperglycemia
(HP:0003074) — or neither, in range. The mapping key is a triple: `(LOINC
analyte, direction)`, where direction ∈ {low, normal, high} comes from comparing
`value_as_number` against `range_low`/`range_high` (or `operator_concept_id`, or
`value_as_concept_id` for qualitative results). Turning a continuous number into
a qualitative high/normal/low call — **value abstraction** — is the actual
computational-phenotyping primitive. Everything after it is lookup and graph.

**Hop 2 is Monarch's `phenotype_to_disease`.** The set of derived HPO terms is the
patient's phenotype profile; ranking MONDO diseases from it is exactly the
Phenomizer-style semantic similarity that `phenotype_profile_search` implements.

### Prior art does most of hop 1

- **LOINC2HPO** (`monarch-initiative/loinc2hpo`; Zhang & Robinson, *npj Digital
  Medicine* 2019). Maps `(LOINC + interpretation L/N/H)` → HPO for 2,923 common
  lab tests. The value-direction is built into the mapping — this is the piece
  that specializes in hop 1 as stated above.
- **OMOP2OBO** (`callahantiff/OMOP2OBO`; Callahan, *npj Digital Medicine* 2023,
  "Ontologizing health systems data at scale"). Concept-level crosswalk from the
  whole OMOP standardized vocabulary to OBO ontologies, built with concept
  alignment + embedding. Conditions → HPO **and** MONDO; measurements → HPO (and
  ChEBI/CL/etc.); ~92k conditions and ~10.7k measurements mapped.

The two are complementary. OMOP2OBO gives broad concept-level coverage and a
direct conditions→MONDO shortcut; LOINC2HPO gives the value-direction
interpretation that OMOP2OBO's concept-level measurement mappings don't encode.
For labs specifically, LOINC2HPO is the sharper hop-1 tool. (This is my reading
of their granularity; worth confirming against OMOP2OBO's measurement mapping
schema before relying on it.)

### PheKnowLator is a different kind of thing — clearing up the confusion

It is easy to file PheKnowLator as "a more general LOINC2HPO." It is not a
mapping resource at all. **OMOP2OBO** is the more-general EHR-concept → ontology
mapper — it does for conditions, drugs, and measurements across many OBO
ontologies what LOINC2HPO does for labs into HPO. That is the "general
LOINC2HPO" role.

**PheKnowLator** (`callahantiff/PheKnowLator`, Callahan et al., *Scientific
Data* 2024) is one layer down: a framework for *constructing* the knowledge
graph itself. You feed it a list of ontologies, a list of linked-open-data / non-
ontology sources, and edge specifications; it emits a semantically grounded,
OWL-reasonable KG (and property-graph / SPARQL forms). It builds graphs like the
one Monarch publishes; it does not map clinical data onto them.

So the three resources sit in a stack, all from adjacent groups:

- **PheKnowLator** — *builds* the graph (ontologies + data → KG). The layer
  Monarch's KGX dump is an output of; relevant to this project only if we ever
  wanted to construct our own graph instead of pulling Monarch's.
- **OMOP2OBO** — the *on-ramp* from EHR data to the graph's ontology nodes
  (OMOP concept → HPO/MONDO/ChEBI/…). The general mapper.
- **LOINC2HPO** — the *specialized on-ramp* for labs, carrying the value-
  direction hop-1 needs.

For this project's use cases the on-ramps (OMOP2OBO, LOINC2HPO) are what matter;
PheKnowLator is construction-side and out of scope unless the graph source
changes.

### Where each architecture lives — it's a hybrid

- **Value abstraction** — threshold logic over reference ranges. No ML. OMOP
  hands you per-row `range_low/high`, so it's data-driven, not hand-tuned
  numbers.
- **LOINC → HPO** — a curated crosswalk (LOINC2HPO / OMOP2OBO). The
  tool-calling-style hop. Embeddings only to propose an HPO term for an unmapped
  analyte, then validated.
- **HPO → MONDO** — the embedding + graph hop. Set-based similarity over the KG.
  The KG-RAG core.

Two curated hops feed one graph hop. This is a sharper "why not just use Shawn's
menu" than drug repurposing: his menu can rank a disease from an HPO set you hand
it, but it has no path *from OMOP data to that HPO set* — the abstraction step
lives entirely outside the graph.

### Caveats before this is real

- **Units.** Reference ranges are unit-specific; normalize `unit_concept_id`
  before the high/low call or the direction is wrong.
- **Missing ranges.** `range_low/high` are often NULL in real OMOP; the fallback
  (LOINC default ranges? population percentiles?) is a modeling decision.
- **Temporality.** One abnormal value ≠ a phenotype; many HPO terms imply
  persistent/recurrent. Timeline aggregation is a design choice.
- **Absence is signal.** A normal lab can rule a disease out; profile similarity
  uses presence — negative handling changes the ranking.

---

## Use case 2 — cohort discovery: the inverse of phenotyping

Run the same chain backwards and it becomes a computable cohort definition:

```
MONDO disease → expected HPO profile → LOINC analytes + directions → SQL over measurement
```

Forward (use case 1) = diagnosis/ranking: given a patient, which disease. Reverse
= **cohort definition**: given a disease, which patients — an OMOP phenotype
algorithm, authored by the KG instead of by hand. The KG supplies what a bare
concept-set author lacks: the disease's MONDO subtype hierarchy, its causative
genes, its constituent phenotypes, which then resolve (via LOINC2HPO /
OMOP2OBO, in reverse) to measurement predicates over the data.

Same edges, opposite traversal direction. That symmetry is the tie between the
two use cases: phenotyping and cohort discovery are one pipeline read two ways.
OMOP2OBO's direct conditions→MONDO mapping also gives a shortcut entry when the
patient already has coded conditions — but the lab-driven path reaches
phenotypes that condition codes never captured.

This is a definition-shaped output. The menu cannot produce it: it has no value
abstraction and no composed reverse traversal.

---

## Use case 3 — drug repurposing / mechanism hypothesis

A multi-hop walk that produces a hypothesis, not a lookup:

```
disease → gene → pathway → drug (with the path as the stated mechanism)
```

This is the clearest showcase of the embedding architecture's structural
advantage. The traversal is open-ended and not enumerable as endpoints, so
tool-calling can't reach it at all — there is no `disease_to_candidate_drug`
router, and there couldn't be a fixed one, because the useful path length and the
intermediate node types vary by question. Retrieving the *path* and handing it to
generation as the mechanism narrative is the point; the hypothesis is the path.

Not unprecedented in the Monarch orbit: **GRAPE**, **NEAT**, and **KG-Hub**
(Reese, Caufield, Mungall et al.) already apply network embedding and link
prediction on Monarch-family graphs to drug discovery, rare-disease link
prediction, and variant prioritization. The contribution here is not the
technique but running it as one arm of a like-for-like comparison across
retrieval epistemologies (see
[`retrieval_epistemologies.md`](retrieval_epistemologies.md)) — same EDS subgraph,
same questions as the other modes.

---

## Sources

- `github.com/monarch-initiative/phenomics-assistant`,
  `oai-monarch-plugin`, `agent-smith-ai`; `github.com/oneilsh/phenomics-explorer`
- LOINC2HPO — `github.com/monarch-initiative/loinc2hpo`; Zhang et al., *npj
  Digital Medicine* 2019, "Semantic integration of clinical laboratory tests from
  electronic health records for deep phenotyping and biomarker discovery"
  (nature.com/articles/s41746-019-0110-4)
- OMOP2OBO — `github.com/callahantiff/OMOP2OBO`; Callahan et al., *npj Digital
  Medicine* 2023, "Ontologizing health systems data at scale: making
  translational discovery a reality" (nature.com/articles/s41746-023-00830-x)
- PheKnowLator — `github.com/callahantiff/PheKnowLator`; Callahan et al.,
  *Scientific Data* 2024, "An open source knowledge graph ecosystem for the life
  sciences" (nature.com/articles/s41597-024-03171-w). A KG-*construction*
  framework, not a mapping resource — the general EHR→ontology mapper is OMOP2OBO.
- Network embedding / link prediction (use case 3): GRAPE (arXiv 2110.06196),
  NEAT, KG-Hub (Reese, Caufield, Mungall et al., arXiv 2302.10800)
