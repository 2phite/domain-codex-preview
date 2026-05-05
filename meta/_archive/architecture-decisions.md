---
title: Wiki Architecture — Design Decisions
description: Decisions made during the initial design discussion; rationale for the key architectural choices
---

# Wiki Architecture — Design Decisions

Recorded from the design discussion between Fai (Ting Fai Lam) and Claude Code, 2026-04-24. The ATSC3 pilot project is the first instance of this pattern; the decisions here are intended to be repeatable for other projects.

---

## D1: Use a structured markdown directory with a manifest, not a single file or a database

**Decision:** The wiki is a directory of markdown files indexed by `manifest.json`.

**Rationale:**
- A single `.md` file does not scale to thousands of tickets' worth of knowledge — a `/lint` command would scan the entire document on every invocation
- Markdown files can be served as static files; no bespoke server is needed at any stage
- `manifest.json` provides the routing layer for AI consumers: fetch manifest → identify relevant sections → fetch only those
- The LLM wiki pattern (already established for this project's `wiki.md`) is based on markdown; this extends it without introducing a new paradigm

**Alternatives rejected:**
- **Single `.md` file**: proven for personal reference but does not scale; AI consumers would need to read everything on each invocation
- **RAG / vector database**: more powerful for fuzzy retrieval but loses human readability; adds infrastructure dependency; overkill before consumption patterns are known
- **Graph knowledge base**: same objections as RAG, plus higher tooling and maintenance cost

---

## D2: Start local-only; decouple hosting from wiki structure

**Decision:** The wiki is built and consumed as local files first. Hosting (static file server or lightweight API) is added later without requiring structural changes.

**Rationale:**
- The existing `wiki.md` proves local-first works in practice
- Hosting decisions should be made after consumption patterns are understood
- The API layer is thin: serve files, return `manifest.json`. No structural changes needed when hosting is added
- Avoids premature infrastructure commitment (internal server details, access control, network reach)

---

## D3: The wiki stores extracted knowledge only, not source material

**Decision:** Jira tickets, SVN logs, and old wiki pages are source material. The wiki contains only knowledge derived from them, with inline source annotations.

**Rationale:**
- Raw source data (thousands of tickets) would make the wiki too large and too noisy for AI consumption
- Jira is the authoritative source for tickets; the wiki does not need to duplicate it
- Traceability is preserved by inline annotations (`<!-- src: PROJ-388 -->`), which are co-located with the fact and cannot desync

**Consequences:** Every extraction must be rewritten as a wiki fact, not a ticket summary. This is the main quality gate.

---

## D4: Inline source annotations for traceability (not a separate traceability document)

**Decision:** Each wiki fact carries `<!-- src: TICKET-ID -->` embedded in the markdown. There is no separate traceability file or database.

**Rationale:**
- Co-location prevents desync: if a fact is edited or deleted, its annotation moves with it
- Lightweight: no additional data structure to maintain
- AI consumers can follow the source reference to retrieve more detail when needed
- Human reviewers can grep for `<!-- src:` to audit provenance across the wiki

---

## D5: Two-pass ingestion model (analyse → intermediate artifact → merge)

**Decision:** Ingestion always has two phases: (1) analyse source material and produce structured extractions, (2) merge extractions into wiki pages. The intermediate artifact is always reviewable before any wiki page is touched.

**Rationale:**
- Separates the extraction judgment (what is worth keeping and where it belongs) from the writing act (how to express it in wiki prose)
- The intermediate extraction (JSON array or `investigation.md`-style document) is the human review checkpoint
- The `PROJ-390/investigation.md` document, produced by analysing SVN + Jenkins data, demonstrated this pattern works: it is structured, readable, and clearly separates findings from raw data

**The investigation.md pattern:** For a complex SVN bundle or cluster of related tickets, produce a structured markdown analysis document first. Selectively merge its findings into the wiki. The investigation document itself may be worth preserving as a reference alongside the wiki (not inside it).

---

## D6: spec-crosslinks.md is built programmatically, not by AI

**Decision:** The requirement → test case cross-link table is produced by a script traversing `master_suite/`, not by AI analysis.

**Rationale:**
- The chain is fully structured and machine-readable: `master.yaml` `assertions:` list → assertion YAML `spec-references:` section → spec section IDs
- A script inverts this mapping exactly, cheaply, and without hallucination risk
- AI analysis is reserved for tasks requiring judgment (understanding why, what matters, historical context); mechanical traversal is not that task

**Implementation:** Script reads every `master_suite/tests/*/master.yaml`, collects assertion IDs, reads each assertion YAML's `spec-references`, and inverts the mapping to produce requirement → `[test-IDs]`.

---

## D7: Old wiki content is classified before extraction; procedures yield facts, not steps

**Decision:** Old wiki pages are classified as procedural / reference / historical / outdated before any extraction. Procedural pages yield the facts embedded in them, not the procedures themselves.

**Rationale:**
- Old procedural pages (e.g., NUC setup instructions) may be months or years out of date, but they contain embedded facts that are still true: naming conventions, kernel pinning rationale, physical paths, infrastructure history
- Extracting the steps would populate the wiki with potentially stale instructions
- The distinction "stable fact embedded in a procedure" vs "the procedure itself" is the key judgment call for old wiki ingestion

**Example:** The Ubuntu installation instructions contain the fact "disconnect the NUC from the internet before installing because Ubuntu will otherwise pull a newer kernel incompatible with the DekTec driver". The steps around it are potentially stale; the constraint is permanently true.

---

## D8: Build sequence is determined by signal quality and dependency order

**Decision:** meta/ → old wikis → primary project newest-first → adjacent projects → SVN logs → spec docs → spec-crosslinks.

**Rationale:**
- `meta/` first: establishes analysis prompts before any source is processed, ensuring consistency across sessions
- Old wikis before Jira: establishes the pre-Jira historical baseline; without it, early Jira decisions lack context
- Primary project newest-to-oldest: maximises early signal (most recent tickets have the highest relevance to current architecture)
- SVN after Jira: SVN anchors revisions to decisions already documented, rather than producing redundant extractions
- Spec cross-links last: depend on both spec docs (already converted) and `master_suite` (stable)

---

## D9: MCP API surface is minimal for the initial build phase

**Decision:** Seven MCP tools covering Jira ticket fetch, Jira list, SVN log summary, SVN diff, wiki read, wiki write, and wiki manifest.

**Rationale:**
- The existing Jira mirror script and SVN tooling map directly to these tools with minimal adaptation
- Automated ingestion (phase 2, living wiki) may add watch/trigger/diff tools, but adding them before they're needed would be premature
- `wiki_manifest` is the tool that makes AI consumers navigable without reading everything; it is the most architecturally important of the seven

---

## D10: Analysis prompts are the consistency mechanism across sessions

**Decision:** `meta/analysis-prompts.md` is the single source of truth for what to extract and how. Every build session reads it before processing any source material.

**Rationale:**
- Different Claude sessions would otherwise make different judgments about what is wiki-worthy, creating inconsistent extraction quality
- Prompts encode the quality bar so it can be applied consistently without human review of every individual extraction
- Prompts can be versioned and improved between phases; improvements apply to all future sessions, not retroactively to past sessions

---

## D11: This process is open and repeatable by design

**Decision:** The `meta/` directory documents not just the what (wiki structure, prompts) but the why (this file). Another Claude Code user from a different project should be able to read `meta/` and reproduce the process for their own codebase.

**What "repeatable" requires:**
- `ingestion-guide.md` describes the process without assuming knowledge of the pilot project
- `analysis-prompts.md` prompts are self-contained — each includes the project background a new session needs
- `architecture-decisions.md` (this file) explains the choices so they can be adapted, not just copied
- The Jira ingest tool (in a separate tooling repo) is a template: swap in the project's Jira URL and credentials

**What the human must provide to start a new project's wiki:**
1. A brief description of the project and its repos
2. The Jira project keys and any naming history
3. Access to source material (Jira mirror, SVN, old wikis)
4. Confirmation of the wiki directory structure (or deviations from the standard)

With those four things, a Claude Code session can read `meta/` and begin ingesting autonomously.

---

## D12: CLI over MCP for the data layer

**Decision:** The data layer tools are CLI scripts called via Bash, not MCP servers.

**Rationale:**
- Claude Code has direct Bash tool access; calling the Jira ingest tool is identical whether the caller is a human or Claude
- MCP requires loading all tool schemas into the context window on every invocation — "schema bloat" costs tokens even when only one tool is used; benchmarks show CLI is up to 32× cheaper per operation for local workflows
- MCP server reliability over TCP shows ~72% completion rates due to timeout issues; CLI via subprocess is 100%
- Claude has strong priors on shell invocation patterns from training; MCP's custom schemas add cognitive load that competes with the actual ingestion task
- The wiki tools (`wiki_read`, `wiki_write`, `wiki_manifest`) are unnecessary entirely: Claude's native Read, Write, Edit, and Glob tools already provide direct filesystem access

**When MCP would be the right choice instead:**
- Multi-user or shared infrastructure requiring per-user OAuth and audit trails
- External SaaS systems without a mature CLI
- Non-developer users who cannot install or configure local tools

None of those conditions apply here: single developer, internal Jira server, Claude Code with full Bash access.

**Consequences:**
- The Jira ingest tool is the only data-layer tool that needs to be built for Jira ingestion; it lives in a separate tooling repo, auto-loaded as a Claude Code skill
- SVN ingestion uses the SVN ingest tool (Python rewrite of an older shell script), also in the tooling repo
- No MCP server process to maintain, start, or authenticate
- A new session acquires Jira access by having the ingest tool on PATH and credentials in `.env` — no MCP configuration step
