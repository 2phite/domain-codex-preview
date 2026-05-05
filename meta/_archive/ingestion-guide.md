---
title: Wiki Build — Ingestion Guide
description: Primary entry point for any session performing wiki build or maintenance work
---

# Wiki Build — Ingestion Guide

This document describes how to build and maintain the ATSC3 pilot knowledge wiki from source material. Read this first in any build session, then read `analysis-prompts.md`.

**Related files:**
- `analysis-prompts.md` — prompts to use when analysing each source type
- `source-formats.md` — target output formats the data layer must produce; what the AI will receive
- `data-layer.md` — analysis of existing scripts, gap analysis, and MCP tool requirements
- `architecture-decisions.md` — rationale for the key design decisions

---

## What this wiki is

A machine- and human-readable knowledge base covering the project's engineering platform. It is built from Jira tickets, SVN commit history, internal wiki pages, official spec documents, and external-tracker tickets. It serves:

- Developers doing day-to-day work (human-readable markdown)
- AI assistants providing expert guidance (machine-readable via `manifest.json`)
- New team members ramping up
- Project managers and tech leads making decisions

The wiki is **knowledge-extracted**, not an archive. Source material is not stored; only knowledge derived from it is stored. Every extracted fact carries an inline source annotation (`<!-- src: PROJ-388 -->`) for traceability.

---

## Wiki directory structure

```
wiki/
├── manifest.json                    # machine-readable index: sections, last-updated
├── README.md                        # human and AI consumer guide
│
├── platform/
│   ├── overview.md                  # what the platform is, two-repo design, certification schemes
│   ├── architecture.md              # Django app, extension system, TAPI, ASGI stack
│   ├── package-dependencies.md      # 3-layer deps, current versions, update procedure
│   ├── infrastructure.md            # hardware fleet, CI nodes, file shares
│   ├── deployment.md                # hardware build, kernel pinning, bootstrap, naming, dongle
│   └── development.md               # dev env, workflows, env vars, run_checks.py
│
├── test-suite/
│   ├── structure.md                 # master_suite, suites/, assertions, svn:externals
│   ├── test-anatomy.md              # master.yaml, launch.yaml, script.py, features
│   ├── build-pipeline.md            # suite-build script, validate, publish to CTA Portal
│   └── spec-crosslinks.md           # requirement → test cases (built programmatically, not AI)
│
├── specs/
│   └── atsc3/
│       ├── {chapter-slug}.md        # one file per spec chapter
│       └── normative-index.md       # all "shall/must" statements in one searchable list
│
├── decisions/
│   ├── index.md                     # table: date | title | tags | source
│   └── {YYYYMMDD}-{slug}.md         # one per significant decision (ADR format)
│
├── history/
│   ├── project-names.md             # historical project codename chain
│   ├── company-names.md             # historical company name chain
│   ├── infrastructure-changes.md    # old hardware, old CI servers, decommissioned kit
│   └── evolution.md                 # system narrative: what changed and why, by era
│
├── known-issues/
│   └── index.md                     # active issues, workarounds, blocked items
│
├── pipelines/
│   ├── platform-ci.md               # platform CI package, smoke tests, toggles
│   ├── atsc3-build.md               # 4-stage × 4-profile design (PROJ-1057)
│   └── security-release.md          # PCAP refresh, canonical pipeline pattern
│
└── meta/                            # this directory — build system docs, not wiki content
    ├── ingestion-guide.md           # this file
    ├── analysis-prompts.md
    ├── source-formats.md
    └── architecture-decisions.md
```

`spec-crosslinks.md` is built by a script traversing `master_suite/` — no AI required. See `architecture-decisions.md` D6.

---

## Build phase sequence

The initial build has two top-level phases: first build the data layer, then build the wiki content. Each content phase can be a separate Claude session.

### Phase 0: Data layer (shared responsibility)

Before any wiki content can be built autonomously, the CLI tooling must be set up. See `data-layer.md` for the full design. The data layer uses CLI tools called via Bash — not an MCP server (see `architecture-decisions.md` D12).

**The data layer is considered ready when:**
- The Jira ingest tool can fetch a ticket with full content and attachments ✓
- The Jira ingest tool can list tickets by project and date range ✓
- The ingestion ledger is accessible ✓
- The SVN ingest tool can query commits by keyword, revision, and date range ✓

**Wiki tools are not needed.** Claude uses its native Read, Write, Edit, and Glob tools to access the wiki filesystem directly.

**The ingest tools live in a separate tooling repo**, auto-loaded as Claude Code skills.

**Setup checklist (one-time, new machine):** install the tooling repo's dependencies, copy `.env.example` to `.env`, and fill in `JIRA_BASE_URL`, `JIRA_OUTPUT_DIR`, `JIRA_USERNAME`, `JIRA_PASSWORD`.

Until phase 0 tooling is set up, wiki sessions operate in **assisted mode**: the user pre-selects and provides source material manually, and the AI processes what it is given.

### Phase 1: Foundation (one session)

1. **`meta/` review** — read and validate all meta files before processing any sources. Prompt quality determines consistency across all subsequent sessions.
2. **Old wikis** — short corpus but high signal for historical context and physical infrastructure facts not captured in Jira. Establishes the pre-Jira baseline that makes Jira decisions interpretable.

### Phase 2: Jira ingestion (multiple sessions, newest-to-oldest)

3. **Primary project tickets** — densest source of architectural decisions. Process in batches of 20–50 tickets per session, newest first.
4. **Test-suite project tickets** — test suite and pipeline evolution.
5. **Adjacent projects** — smaller, more recent projects.

### Phase 3: SVN ingestion (supplementary)

6. **SVN logs** — anchors decisions to revision numbers, fills gaps where ticket messages are thin. Process after Jira so that SVN analysis adds to documented decisions rather than creating redundant entries.

### Phase 4: Spec content (can run in parallel with phases 2–3)

7. **Spec docs** — mechanical conversion of official spec PDFs to markdown. One file per chapter.

### Phase 5: Programmatic (script-only, no AI)

8. **`spec-crosslinks.md`** — built by script traversing `master_suite/`. Run after spec docs and `master_suite` are stable. No AI session needed.

---

## How a build session works

1. Read `meta/ingestion-guide.md` (this file) and `meta/analysis-prompts.md`.
2. Receive or fetch a batch of source material (e.g., 30 Jira tickets via `jira_list`).
3. For each item: apply the appropriate analysis prompt → produce structured extractions.
4. Review extractions: check destination paths, discard low-confidence facts.
5. Merge extractions into wiki pages. Deduplicate against existing content.
6. Update `manifest.json` with new or modified pages.
7. Report to the human: what was processed, what was written, what was discarded.

**Do not batch-write without review.** The human reviews what changed between sessions.

The intermediate extraction (JSON or investigation.md) is itself a deliverable — reviewable before any wiki page is touched.

---

## Data layer

The data layer is the set of MCP tools that give a Claude session autonomous access to source material and wiki storage. It is a significant design and implementation effort in its own right — see `data-layer.md` for the full analysis.

**Summary of what the data layer must provide:**

| Capability | How | Status |
|---|---|---|
| List Jira tickets by project + date range | Jira ingest tool (`--list`) | Done |
| Fetch a single Jira ticket with attachments | Jira ingest tool | Done |
| Query SVN commits by date range / path | SVN ingest tool | Done |
| Fetch diff for a single revision | SVN ingest tool | Done |
| Fetch commit history for a file or directory | SVN ingest tool (`--history`) | Done |
| Read / write wiki pages | Claude native Read / Write / Edit tools | Done (no tooling needed) |

Until the data layer is complete, sessions run in **assisted mode**: the user pre-selects and provides source material manually. The analysis prompts in `analysis-prompts.md` work equally well in assisted mode — the only difference is who fetches the source material.

---

## Quality bar

A wiki entry is good if:
- Removing the source annotation, a future reader still understands the constraint or decision
- It answers **why**, not just what
- It will still be true in 6 months (or is explicitly flagged as temporary)
- It is not a step-by-step procedure (those belong in the repo)

A wiki entry is bad if:
- It repeats what the code already says
- It is a status update ("ticket closed", "looks good")
- It is a procedure that may go stale
- It does not explain why the thing is the way it is
