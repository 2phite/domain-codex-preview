# domain-codex (archive)

A process for building and maintaining a structured knowledge wiki from engineering source material — Jira tickets, SVN/Git commit history, internal wiki pages, spec documents, and conversation snippets.

Designed to be repeatable across projects. The `meta/` directory is the process; the wiki content it produces is project-specific and lives elsewhere.

## What it produces

A directory of markdown files indexed by `manifest.json`, serving:

- Developers doing day-to-day work (human-readable markdown)
- AI assistants providing expert guidance (machine-readable via manifest)
- New team members ramping up
- Project managers and tech leads making decisions

The wiki stores **extracted knowledge only** — not raw source material. Every fact carries an inline source annotation (`<!-- src: TICKET-ID -->`) for traceability.

## How to start a build session

1. Read [`meta/ingestion-guide.md`](meta/ingestion-guide.md) — process overview, directory structure, build phases
2. Read [`meta/analysis-prompts.md`](meta/analysis-prompts.md) — self-contained prompts for each source type
3. Provide or fetch source material; apply the appropriate prompt; review extractions; merge into wiki pages

## What a new project must supply

To adapt this process to a different codebase:

1. A brief description of the project and its repos
2. The issue tracker project keys and any naming history
3. Access to source material (issue tracker, VCS log, old wikis)
4. Confirmation of the wiki directory structure (or deviations from the standard layout in `ingestion-guide.md`)

## Files

| File | Purpose |
|------|---------|
| [`meta/ingestion-guide.md`](meta/ingestion-guide.md) | Primary entry point; process overview and build phases |
| [`meta/analysis-prompts.md`](meta/analysis-prompts.md) | Self-contained prompts for Jira, SVN, old wikis, decision records |
| [`meta/source-formats.md`](meta/source-formats.md) | Target output formats the data layer must produce per source type |
| [`meta/data-layer.md`](meta/data-layer.md) | CLI tool design for autonomous AI ingestion (tools live in a separate tooling repo) |
| [`meta/architecture-decisions.md`](meta/architecture-decisions.md) | Design decisions and rationale |
