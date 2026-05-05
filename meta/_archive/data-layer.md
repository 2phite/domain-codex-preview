---
title: Data Layer — Tooling Design
description: CLI tool design for wiki ingestion; gap analysis against existing scripts; SVN tool requirements
---

# Data Layer — Tooling Design

This document covers the data infrastructure for the wiki build project: what the existing scripts do, where they fall short for autonomous AI ingestion, and how the new CLI tool addresses those gaps.

**Status:**
- Jira ingest tool — implemented; lives in a separate tooling repo
- SVN ingest tool — implemented (full Python rewrite of an earlier shell script)
- Wiki tools — not required; Claude uses native file tools

**Tool location:** The CLI tools are maintained in a separate tooling repo, not in this repo. They are available in any Claude Code session via skill definitions.

**Architecture decision (D12):** The data layer uses a CLI tool, not an MCP server. Claude Code calls the Jira ingest tool via Bash exactly as the user does. This eliminates protocol overhead, schema bloat, and server reliability issues. See `architecture-decisions.md` D12.

---

## The core problem: ad hoc vs. autonomous

The existing scripts were designed for **human-driven ad hoc use**: the user knows which tickets or revisions they want, runs the script manually, and reads the output.

Wiki ingestion requires **AI-driven autonomous discovery**: Claude does not know upfront which tickets or revisions exist. It needs to query for them, decide which are worth fetching in full, fetch them, and process them — without a human selecting items upfront.

Under the CLI architecture, "autonomous" means Claude calls the Jira ingest tool via Bash to list tickets, reads the printed table, then fetches the ones it selects. The same tool, called by either human or AI.

---

## Existing script audit

### Earlier per-ticket fetcher scripts

**What they did well:**
- Fetched a single ticket via Jira REST API v2 (details + comments + remote links)
- Produced clean `_ticket.md` markdown output
- Auth via keyring; no hardcoded credentials
- Organized output by quarter (`YYYY-Q#/TICKET-ID/_ticket.md`)

**What they lacked:**
- No batch query mode — could not list tickets by project or date range
- No attachment support (text docs and images uploaded to tickets not fetched)
- Per-quarter directory layout made programmatic access awkward
- No ingestion ledger

These scripts informed the design of the Jira ingest tool and are now superseded for wiki ingestion purposes. A separate activity-stream script is unaffected.

---

### Earlier SVN shell script

**What it does well:**
- Three modes: direct revision list, keyword search, path history
- Produces `summary.txt` (compact, structured, excellent for AI consumption) + `diffs/r*.diff`
- Binary file detection and skipping
- Path history mode embeds full commit metadata into each diff section

**What it cannot do for autonomous ingestion:**
- No date range query mode: `svn log -r {2023-01-01}:{2023-12-31}` is not exposed
- No cross-path query: "all commits touching `jenkins/` between dates" requires combining date range + path filter
- No cross-repo query: each repository must be queried separately

These gaps are addressed by the SVN ingest tool, which supersedes the earlier shell script.

---

## CLI tool: Jira ingest tool

### Location and setup

The tool lives in a separate tooling repo with the structure below. Set up once per machine.

**Setup (once, on a new machine):** create a virtualenv, install requirements, copy `.env.example` to `.env`, and fill in `JIRA_BASE_URL`, `JIRA_USERNAME`, `JIRA_PASSWORD`, `JIRA_OUTPUT_DIR`.

**Invocation** (via a `jf` bash alias wrapping the main script):
```bash
jf PROJ-123
jf PROJ-123 PROJ-124 ALT-456
jf --keys-file tickets.txt
jf --list PROJ --from 2025-01-01 --to 2025-12-31
jf --list PROJ --status Done --limit 50
jf --jql "project = PROJ AND assignee = currentUser()"
jf --ledger
jf --ledger PROJ
jf --mark-synced PROJ-123 PROJ-124       # set wiki_synced_at to now
jf --unmark-synced PROJ-123              # clear wiki_synced_at
jf --read PROJ-123                       # print path to cached _ticket.md, fetch if missing
jf --read PROJ-123 --max-age 1d          # also refetch if local copy >24h old
jf PROJ-123 --output /custom/path        # override output dir (used by Claude)
```

### Configuration (`.env`)

| Variable | Required | Purpose |
|---|---|---|
| `JIRA_BASE_URL` | yes | Jira server URL (e.g. `https://jira.your-company.example.com`) |
| `JIRA_OUTPUT_DIR` | yes | Root directory for ticket files and ledger |
| `JIRA_USERNAME` | — | Jira username |
| `JIRA_PASSWORD` | — | Jira password (falls back to keyring if unset) |
| `LLM_ENABLED` | `false` | Enable LLM enrichment |
| `LLM_PROVIDER` | `ollama` | `ollama` \| `anthropic` \| `openai-compat` |
| `LLM_BASE_URL` | `http://localhost:11434/v1` | LLM endpoint (not used for `anthropic`) |
| `LLM_MODEL` | `llama3.2` | Model name |
| `LLM_API_KEY` | `ollama` | API key (blank or `ollama` for local) |

### Output structure

Tickets are saved flat — no quarter subdirectories:
```
{JIRA_OUTPUT_DIR}/
├── PROJ-123/
│   ├── _ticket.md
│   └── attachments/       # created only when attachments exist
│       ├── investigation.md   # text attachments inlined in _ticket.md
│       └── screenshot.png
├── PROJ-124/
│   └── _ticket.md
└── ledger.csv
```

### Ticket history

Every fetch includes the full Jira changelog (`expand=changelog`). The generated `_ticket.md` contains:

- **Previously known as:** callout in the header if the ticket key was ever renamed or moved (e.g. `OLD-50 → PROJ-123`). Multiple renames produce the full chain. This is critical for knowledge-chain integrity — old commit messages and comments may reference the original key.
- **Change History table:** chronological record of field changes (key renames, status transitions, assignee changes, project moves, resolution changes).
- **Description edits subsection:** unified diff for each time the description was edited, showing exactly what changed and when.
- **Edited comment annotation:** if a comment was edited after posting, its header shows `_(edited DATE)_`. The previous comment text is not available via the Jira Server REST API.

### Attachment handling

All attachments are saved to `attachments/` and listed as bullet items in `_ticket.md` (name, MIME type, size). No content is inlined — keeping `_ticket.md` compact for AI ingestion.

If LLM is enabled, text attachments and images also get a markdown summary written to `attachments/{stem}_description.md`. The ingester reads these description files rather than the raw attachment content.

### LLM enrichment (optional)

When `LLM_ENABLED=true`:
- **Image attachments** → LLM generates a 2–4 sentence technical description
- **Ledger entries** → LLM generates a one-line summary and hashtags

Supports Ollama (local, free), Anthropic Claude, and any OpenAI-compatible endpoint. LLM enrichment is best-effort — if the call fails, the fetch completes without it.

---

## Ingestion ledger

`{JIRA_OUTPUT_DIR}/ledger.csv` records every ticket fetched via this tool. It is **not version-controlled**.

| Column | Purpose |
|---|---|
| `key` | Ticket key |
| `summary` | Ticket summary at time of fetch |
| `status` | Ticket status at time of fetch |
| `fetched_at` | Timestamp of local fetch |
| `ticket_updated_at` | Jira's `updated` timestamp at time of fetch |
| `llm_summary` | LLM-generated one-line summary (if enabled) |
| `llm_tags` | LLM-generated hashtags (if enabled) |
| `wiki_synced_at` | Last time wiki was updated from this ticket |

View the ledger via the tool's `--ledger` flag.

The `F` (freshness) column in the display shows `!` if a ticket has never been synced to the wiki, and `~` if the ticket was updated on Jira after the last wiki sync (`ticket_updated_at > wiki_synced_at`).

**Three timestamps, three independent events:**
- `ticket_updated_at` — when the ticket last changed at the source (Jira)
- `fetched_at` — when we last grabbed a current copy locally
- `wiki_synced_at` — when wiki pages were last updated from this ticket

Collapsing any pair loses signal. The `llm_summary` / `llm_tags` ledger fields are descriptive metadata about the ticket and **do not** count as wiki sync — they describe the ticket, not the wiki. `wiki_synced_at` advances only when actual wiki pages are written.

**Updating sync state:** After processing tickets into wiki pages, run the tool's `--mark-synced` command to set `wiki_synced_at` to now. Use `--unmark-synced` to clear (e.g. to force re-processing on the next pass). Both commands skip keys that aren't in the ledger (with a warning to stderr) — they never create phantom rows.

---

## Wiki tools

No separate wiki tools are needed. Claude Code has native Read, Write, Edit, and Glob tools that provide direct filesystem access to the wiki directory. The wiki manifest (`manifest.json`) is a plain JSON file that Claude reads with the Read tool.

This is strictly simpler than building MCP wiki tools and avoids the protocol round-trip overhead.

---

## SVN ingest tool

The SVN ingest tool is a full Python rewrite of an earlier shell script. Lives in the tooling repo. Invoked via an `sb` bash alias. Auto-detects the SVN working copy root by walking up from cwd (like git).

**Mode inference from positional args:**

| Invocation | Mode |
|---|---|
| (no args) | Recent activity (`--days` window, default 100) |
| `12345` | Single revision |
| `12340 12350` | Revision range, inclusive |
| `12340,12345,12348` | Explicit revision list |
| `PROJ-123` | Keyword search in log messages |
| `2025-01-01` | From date to today |
| `2025-01-01 2025-06-30` | Date range |

**Modifiers:** `--path PATH` (subtree filter), `--no-diff` (summary only), `--days N`, `--limit N`, `--output DIR`, `--history PATH` (file/dir history mode).

**For autonomous AI ingestion, the two-pass pattern:**
1. Date range with `--no-diff` — scan all commit messages, identify relevant revisions.
2. Single revision — fetch full diff for each selected revision.

JSON output was considered but not implemented — `summary.txt` is directly readable by Claude with no parsing overhead.

---

## Design principles

1. **CLI over MCP** — Claude calls the ingest tools via Bash exactly as a human would. No protocol layer, no schema loading, no server process.
2. **Query before fetch** — `--list` and `--jql` return lightweight metadata. Claude reviews the list and selects tickets to fetch. Nothing is saved until `jf KEY` is explicitly called.
3. **The AI decides what to ingest** — the tools are neutral. They do not pre-filter by relevance or score items.
4. **Idempotent** — fetching the same ticket twice overwrites `_ticket.md` and updates the ledger timestamp. Safe to re-run.
5. **Read-only for remote systems** — no tool modifies Jira or SVN. Writes go to the local filesystem only.
6. **Fail loudly on auth errors** — missing or wrong credentials print a clear error with setup instructions.
