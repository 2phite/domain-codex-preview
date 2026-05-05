# ATSC3 Pilot

The first `domain-codex` pilot. The pilot project is an ATSC3 test platform — a Django-based test harness and test suite, plus the CI/CD infrastructure that builds and deploys them.

This directory is the project-specific record. Project-agnostic design and rationale live in the meta layer above; what's here is what's particular to this pilot.

## Status

> **2026-05-04** — Mechanism-level extraction validated on a small ticket pilot drawn from an Ubuntu-upgrade chain. The staged validation pass (heterogeneous + quantitative; see [`corpus.md`](../../corpus.md#validation-pass)) has not yet started. Full corpus ingest deferred until validation stage 4 confirms graph behaviour at scale.

## Corpus scope

All tickets across the active Jira projects covering harness, test-suite, and certification work, scoped to a multi-year horizon. Older read-only projects are out of scope unless the historical horizon is revisited.

## External standards in scope

- **ATSC 3.0 family** — A/300 series, A/360 (security), A/331 (signaling), A/344 (interactive content). Selective ingestion: clauses the harness actually exercises.
- **CTA NEXTGEN TV** test plans.
- **NAB NEXTGEN TV** test plans.

These are external immutable inputs the project negotiates against, distinct from internal design intent. See [`corpus.md`](../../corpus.md#external-specifications) for how they get ingested.

## Code repositories

- **Harness application repo** — the test platform itself. Indexed via `graphify`.
- **Test suite repo** — per-test scripts, assertions, test-vector data.

Both indexed via `graphify` separately from the LightRAG text store. Cross-store reasoning happens at the consumer.

## Stack

The pilot uses:

- **LightRAG** as the hybrid text index, hosted on a local GPU server.
- **qwen3** as the extractor model, served via LM Studio.
- **nomic-embed-text** as the embedder.
- **MinerU + RAGAnything** for non-text ingestion (PDFs, attachments, images), co-located on the GPU server.
- **graphify** for code, run per-repo on the working copy.
- Jira and SVN ingest tools (per-source), with output adapted to RAG-friendly form by a post-process step.
- Consumers call the server's HTTP retrieval API.

## Project-specific gatekeeper notes

> **Routine patterns that should usually be filtered out:**
> - "Add assertion for new test ID" tickets with a single small commit and no discussion.
> - Dependency-bump tickets with no failure context.
> - Auto-generated tickets from CI failures that closed without investigation.
>
> **Routine patterns that should usually be kept** despite looking routine:
> - Harness infrastructure tickets — even small ones often touch infrastructure with cross-cutting effect.
> - Any ticket with a non-trivial comment thread or attached design discussion.

These rules are starting points. Refine after validation stage 1 with concrete examples from the heterogeneous extraction-quality run.

## Open questions specific to this pilot

- **Partition decision.** The test-suite project is large enough that if validation stage 4 shows degradation, it partitions cleanly into its own sub-index. Decided after stage-4 data.
- **External-tracker corpus scope.** Customer-facing tickets exist in a separate tracker; per-project counts and date horizon not yet inventoried. Decide whether that ingestion is part of the first full ingest or a phase-2 addition.
- **Curation interface.** Teams adopting this pattern typically have a chat-platform presence; the `@bot` channel is the most likely first interface. Web form is second. Decision deferred until consumers.md's first-version proposal gets concrete.
