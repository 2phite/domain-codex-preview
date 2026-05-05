---
title: Source Format Catalog
description: Target output formats the data layer must produce per source type, plus reference material for parsing each format
---

# Source Format Catalog

This document describes what the AI should **receive** from each source type — the target output format the MCP tools must produce. Where the existing scripts already match, no change is needed. Where they fall short, `data-layer.md` documents the gap and requirements.

**If the format reference tables here conflict with what the MCP tools actually produce, update this file to match reality — not the other way around.** This document describes the target; the tools are the implementation.

| Source | Format the AI receives | Signal | What goes into the wiki |
|--------|----------------------|--------|------------------------|
| Jira ticket | `_ticket.md` (markdown, Jira markup in body) | Very high | Decisions → `decisions/`, architectural facts → `platform/` or `pipelines/`, workarounds → `known-issues/` |
| Jira ticket list | Lightweight metadata: `{id, summary, status, priority, created}` | — | Used to build fetch queue only; not ingested directly |
| SVN commit summary | `summary.txt` structured text (see below) | Medium | Milestone revisions anchoring decisions; coupling signals; historical pivots |
| SVN diff | Unified diff | Low–medium | Fetched selectively when commit message is thin but changed files suggest importance |
| Old wiki page | WikiCreole markup (see below) | Medium–high | Strip markup → extract stable facts → `platform/deployment.md`, `history/` |
| Pre-analysed `investigation.md` | Markdown | Very high | Read directly, extract findings without re-analysis |
| Spec PDF | Markdown (pre-converted by the document-parser tooling) | Variable | `specs/{spec-name}/{chapter-slug}.md` one file per chapter |
| Redmine ticket | TBD — format to be defined when ingestion scripts are built | Medium | Customer-revealed gaps → `known-issues/`; compliance edge cases → `specs/` |
| Chat-platform snippet | (human-provided on request) | High when needed | Supporting evidence cited inside a `decisions/` record; not standalone entries |

---

## SVN log format reference

### summary.txt structure

```
================================================================================
SVN Revision Bundle
================================================================================
Repository: your-repo
URL:        https://svn.your-company.example.com/repos/your-repo/trunk
Revisions:  131425 131952 ...
================================================================================

--------------------------------------------------------------------------------
Revision: 178962
--------------------------------------------------------------------------------
Author:   developer.name
Date:     Tuesday, April 14, 2026 16:25:35 (2026-04-14 16:25:35 +0100)
Message:
PROJ-388 Upgrade Fabric/Invoke/Patchwork stack to Fabric 3
----
  Modified: /your-repo/trunk/install/requirements.txt
  Added:    /your-repo/trunk/install/python_packages/fabric-3.2.3-py3-none-any.whl
  Deleted:  /your-repo/trunk/install/python_packages/patchwork-1.0.1-py2.py3-none-any.whl
```

### High-signal file patterns

| File pattern | What it signals |
|---|---|
| `install/requirements*.txt` | Package decisions, dependency changes |
| `jenkins/*.sh` or `jenkins/*/pipeline.py` | CI/CD decisions, script retirement |
| `extensions/*/` changes | Extension architecture changes |
| Deletions of `*.sh` | Script retirement (document what was replaced and why) |
| `harness/*/models.py` | Data model changes |
| `harness/*/settings.py` | Configuration changes |

### Low-signal patterns (usually skip)

| File pattern | Why low signal |
|---|---|
| `install/python_packages/*.whl` add/delete only | Package bump; only document if breaking change |
| `install/constraints.txt` only | Regenerated automatically; not a decision |
| `svn:mergeinfo` changes | Branch infrastructure, not knowledge |
| Multiple commits to same file within 1 hour | Iteration noise; summarise as one fact |

### Jira project key history (for interpreting commit messages)

Build a table mapping each project key seen in the SVN log to its era, role, and current status (active / winding down / read-only). Old commit messages reference keys from prior eras of the same product line; without the mapping, era classification of an extracted fact is unreliable.

---

## Old wiki format reference

### WikiCreole markup → Markdown equivalents

| WikiCreole | Markdown | Note |
|---|---|---|
| `= Heading =` | `# Heading` | |
| `== Subheading ==` | `## Subheading` | |
| `{{{inline code}}}` | `` `inline code` `` | |
| multiline `{{{ ... }}}` | ` ``` ... ``` ` | |
| `[[WikiLink]]` | (internal link, often dead) | Treat as historical reference, not live link |
| `[[text\|URL]]` | `[text](URL)` | |
| `<<TableOfContents>>` | discard | |
| `<<Anchor(name)>>` | discard | |

### Company and infrastructure name history

Build a mapping table of old hostnames, domain names, and company names to their current equivalents. This is needed to interpret old wiki references correctly — old docs refer to decommissioned servers and prior company names that no longer appear in active infrastructure.

### Era classification for old wiki content

Assign an era tag to each extraction from old wiki pages. The era taxonomy is project-specific — define one tag per major product / infrastructure phase, with the date range and the project codename(s) active during it. Earliest era usually carries highest obsolescence risk.

---

## Redmine

Redmine is used as an external/customer-facing bug tracker. Access method TBD; format to be catalogued when ingestion scripts are built.

Key extraction difference from Jira: **omit all customer-identifying details**. Extract only the pattern (what class of issue was revealed) and the consequence for the wiki (gap in design, compliance edge case, documentation need).

---

## Chat-platform snippets

Not a primary source. The human provides relevant chat conversation excerpts when:
- A Jira ticket directly links to a chat thread
- The thread contains rationale not present in the ticket

Treat as supporting evidence. Cite in `decisions/` records as `<!-- src: chat, {context}, {approx date} -->`. Do not create standalone wiki entries from chat content alone.
