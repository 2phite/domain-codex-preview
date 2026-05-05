# [Preview] domain-codex

A repeatable process for building a structured project knowledge hub from chaotic engineering source material — tickets, version-control history, code, specs, internal docs.

A domain expert ingests their project's raw record into a hybrid graph+vector index. From that index, two artefacts are generated on demand: a queryable backend that AI consumers (assistants, chat bots, ticket-drafting webhooks) call directly, and a periodically-regenerated static wiki for humans reading cold.

## Where to read

The design layer lives in [`meta/`](meta/):

- [`meta/README.md`](meta/README.md) — entry point; reading order; maintenance rule.
- [`meta/why.md`](meta/why.md) — strategic intent; rejected alternatives.
- [`meta/architecture.md`](meta/architecture.md) — the four-stage model: ingest → index → generate → curate.
- [`meta/corpus.md`](meta/corpus.md) — what to ingest; the gatekeeper; the staged validation pass.
- [`meta/consumers.md`](meta/consumers.md) — live retrieval (bots) vs static wiki (humans); curation overlay.

Per-project pilots live under [`meta/projects/`](meta/projects/). The first pilot is [`meta/projects/atsc3-pilot/`](meta/projects/atsc3-pilot/) — an ATSC3 test platform.

## What this repo is not

The `meta/` design docs are the deliverable. This repo is **not**:

- a wiki engine — the static wiki is regenerated from the index on demand, not authored here;
- a personal scratchpad — those live in each contributor's local notes;
- specific to any one project — the architecture is project-agnostic; per-project specifics belong under `meta/projects/<name>/`.

Per-project pilot artefacts (extraction scripts, index stores, virtualenvs) are not committed here.
