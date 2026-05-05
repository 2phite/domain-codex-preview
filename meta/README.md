# domain-codex / meta

The project knowledge hub. A domain expert ingests their project's chaotic source material — tickets, version-control history, code, specs, internal docs — into a hybrid graph+vector index. From that index, two artefacts are generated on demand: a queryable backend that AI consumers (assistants, chat bots, ticket-drafting webhooks) call directly, and a periodically-regenerated static wiki for humans reading cold.

This `meta/` directory is the project's design layer. It exists so the **next** project to adopt the pattern can read these docs and understand not just *what* to do but *why* this shape was chosen and which alternatives were rejected.

> **Status (2026-05-04):** Meta docs drafted; mechanism-level extraction validated on a small sample. Staged validation per [corpus.md](corpus.md#validation-pass) is next.

## Entry points

Read in this order if you're new:

1. **[why.md](why.md)** — what problem this project solves; what alternatives were rejected and why; the strategic intent.
2. **[architecture.md](architecture.md)** — the four-stage model: ingest → index → generate → curate.
3. **[corpus.md](corpus.md)** — what to ingest from each source; the gatekeeper filter; the staged validation pass; re-ingestion cadence.
4. **[consumers.md](consumers.md)** — live retrieval (bots) vs static wiki (humans); curation overlay; future interfaces.

Per-project pilots (concrete instances of the pattern):

- **[projects/atsc3-pilot/](projects/atsc3-pilot/)** — first pilot: an ATSC3 test platform.

Per-project tooling, infrastructure, and corpus specifics live under each project's directory; the meta layer stays project-agnostic.

## What this is not

- Not a wiki engine. The static wiki is a derived artefact, regenerated; not authored.
- Not a personal scratchpad — those live in each contributor's local notes. Knowledge with cross-project value migrates here via ingestion.
- Not specific to any one project. The first project pilot is the proving ground; the architecture is project-agnostic.

## Maintenance rule

These docs are live deliverables. Update them when:

- A retrieval mode is added or changed → [architecture.md](architecture.md), [consumers.md](consumers.md).
- A source format or gatekeeper rule changes → [corpus.md](corpus.md).
- A major decision is made or reversed → record rationale in the relevant doc; if it warrants persistence, promote to `decisions/`.
- A new project pilot starts → add a directory under `projects/`.

Do not defer these updates to a separate session. Stale meta = next project starts with wrong assumptions.
