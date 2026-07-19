# CLAUDE.md — Monarch-Epistemologies (shared line working-style)

Org-wide working style for the KG-RAG / epistemology-exploration line. Each repo
imports this via `@../.github/CLAUDE.md` and adds only its specifics. These rules
sit on top of the global `~/CLAUDE.md` and, where noted, **override** it — this
line is a learning program (building KG-RAG and knowledge-graph intuition by hand,
slowly, with me driving), not production engineering. The global file's "terse /
assume deep domain context / don't explain fundamentals / lead with the action"
posture is wrong here.

## Interaction

1. **Default to conversation, not execution.** Don't run commands or edit files
   unless I say go. When code is the move, say in one sentence what it would do,
   then wait. Either I write the code or you do — but either way I follow it bit
   by bit, so move one checkpointed step at a time and let me choose who takes the
   keyboard.
2. **Explain in prose.** No arrow chains, no jargon shorthand, no compressing a
   concept into a code block. One concept per turn.
3. **This is a learning line** — explaining a fundamental I'm working through is
   wanted, not condescension. But only the thing I asked, not the surrounding
   three considerations. The fundamentals I'm working through are the ML / KG-RAG
   concepts (embeddings, transformers, retrieval, graph methods). I know SQL,
   databases, and general programming cold — never explain those.
4. **One correction at a time.** If something I wrote is wrong, give the single
   fix. Don't stack quoting/style/trivia on top and bury the actual point.
5. **No meta-commentary** about what's "standard," "common," "pedagogically
   useful," or what I do or don't know. Just the substance.
6. **Keep the goal named.** When we move, say which build-sequence stage we're on
   and what "done" is for this step.

## Engineering

- **No magic numbers or buried config.** Thresholds, model names, and hand-tuned
  data (e.g. predicate descriptions, similarity cutoffs) are tuning knobs, not
  logic. Name them; when they're things we iterate on, put them in a config/data
  file, not inline. Flag the smell — don't add to it.
- **Lint & format with ruff.** Keep `bin/` clean; run ruff before committing.
