# Related work: Phenomics Assistant (Monarch chatbot)

## What it is

Shawn O'Neil's Monarch chatbot. The canonical repo is
`monarch-initiative/phenomics-assistant` (his working fork is
`oneilsh/phenomics-explorer`, described as an "LLM retrieval augmented
generation agent for the Monarch Knowledge graph"). Same end goal as this
project — natural-language questions answered against the Monarch KG — but it
reaches the graph a different way, and that difference is the reason to write
this down.

It's built from three repos stacked on each other:

1. **`phenomics-assistant`** — a Streamlit chat UI.
2. **`agent-smith-ai`** — the LLM orchestration layer. Its `UtilityAgent`
   class holds the conversation, and on each turn lets the model either answer
   or call a tool. When the model calls a tool it yields a message with
   `is_function_call=True`, runs the call, feeds the JSON result back into the
   conversation, and loops until the model produces prose.
3. **`oai-monarch-plugin`** — a FastAPI service that wraps the Monarch REST
   API (v3) as a handful of LLM-friendly endpoints.

The retrieval mechanism is **LLM tool-calling over a curated menu of REST
endpoints**, not embedding-based retrieval. There are no graph-node embeddings
anywhere in the system.

## The tool surface is the graph's association types, enumerated by hand

The whole retrieval interface is the list of files in
`oai-monarch-plugin/src/oai_monarch_plugin/routers/`:

- `disease_to_gene` / `gene_to_disease`
- `disease_to_phenotype` / `phenotype_to_disease`
- `gene_to_phenotype` / `phenotype_to_gene`
- `phenotype_profile_search` — given a set of phenotypes, find best-matching
  diseases/genes (semantic phenotype similarity, Monarch's own service)
- `entity` — look up one node by its ontology ID
- `search` — resolve a name ("cystic fibrosis") to an ontology ID
  (`MONDO:...`)

That list is the biolink association categories among the three node types
Monarch centers on — disease, gene, phenotype — written out in both
directions. A person decided which traversals were worth exposing and wrote one
endpoint per traversal. Adding a new kind of traversal means writing a new
router.

## What the model actually sees for one tool

The `disease_to_phenotype` router, verbatim on the parts that matter:

```python
@router.get(
    "/disease-phenotypes",
    description="Get a list of phenotypes associated with a disease",
    summary="Get a list of phenotypes associated with a disease",
    operation_id="get_disease_phenotype_associations",
)
async def get_disease_phenotype_associations(
    disease_id: str = Query(..., description="The ontology identifier of the disease.",
                            example="MONDO:0009061"),
    limit: Optional[int] = Query(10, ...),
    offset: Optional[int] = Query(0, ...),
) -> PhenotypeAssociations:
    ...
    genericAssociations = await get_association_all(
        category="biolink:DiseaseToPhenotypicFeatureAssociation",
        entity=disease_id, limit=limit, offset=offset,
    )
```

FastAPI turns this into an OpenAPI spec; `agent-smith-ai` reads that spec and
offers each endpoint to the model as a callable function (the framework
requires every callable endpoint to carry a `description` and an
`operationId`). So the model's entire knowledge of this tool is three things:
the name `get_disease_phenotype_associations`, the English sentence "Get a list
of phenotypes associated with a disease", and the typed parameter `disease_id`
with the example `MONDO:0009061`. The natural-language descriptions *are* the
retrieval interface — the model matches question to tool by reading them.

Under the hood the endpoint is one Monarch API call, pinned to a fixed biolink
category.

## The full loop, on a real question

"What symptoms does cystic fibrosis cause?"

1. Model calls `search` to turn "cystic fibrosis" into `MONDO:...`. (Name → ID
   is its own tool, and must run first — the association endpoints take IDs,
   not names.)
2. Model reads its menu, matches intent to
   `get_disease_phenotype_associations`, calls it with that ID.
3. `agent-smith-ai` runs the HTTP call, feeds the JSON back, model narrates.

## The contrast with this project

Both designs have to solve the same two problems Project 2 names — get from the
words of the question onto a specific node (**anchor**), then follow the right
edges (**traverse**). They solve both differently.

**Anchor.** Phenomics Assistant anchors with a dedicated `search` tool: the
model calls Monarch's own name-resolution endpoint and gets back an ontology
ID. Project 2 anchors with embeddings — embed the question, compare to node
text in vector space — because it has no curated resolver to lean on. His
system offloads the hard entity-linking problem to a Monarch service; ours does
it in-house and hits the disambiguation trouble recorded in the Project 2 notes
(the degree-based tie-break).

**Traverse.** This is the sharp divergence.

- **His way:** a human enumerated the useful traversals as ~9 named endpoints,
  and the LLM picks among them by reading English descriptions. Retrieval is
  *tool selection over a curated menu*. Precise — each option is a hand-written
  sentence, so the model rarely picks wrong — but bounded: the model can only
  do what the menu allows, and new traversals need new code.
- **Our way:** embed the predicate set and pick edges by similarity, then walk
  them with plain SQL over the edges table. Any traversal the graph supports is
  reachable without hand-writing an endpoint for it — at the cost of the
  matching being fuzzy rather than a curated sentence.

So the two systems trade opposite ways. His buys precision and pays in
coverage and hand-maintenance; ours buys coverage and generality and pays in
disambiguation difficulty. Neither is "the" KG-RAG design — they sit at
opposite ends of how a natural-language question reaches an edge in the graph:
curated tool-calling versus embedding retrieval.

## Sources

- `github.com/monarch-initiative/phenomics-assistant`
- `github.com/monarch-initiative/oai-monarch-plugin`
- `github.com/monarch-initiative/agent-smith-ai`
- `github.com/oneilsh/phenomics-explorer` (O'Neil's fork)
