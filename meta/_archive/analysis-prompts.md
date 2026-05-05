---
title: Analysis Prompts
description: Self-contained prompts for analysing each source type during wiki ingestion
---

# Analysis Prompts

Use these prompts when analysing source material for wiki ingestion. Each prompt is self-contained and can be given directly to a Claude session along with the source material.

**Before using any prompt:** Read `ingestion-guide.md` and the `source-formats.md` entry for the source type. This gives context that improves extraction quality.

---

## Prompt A — Jira ticket

> You are building the project knowledge wiki. You have been given a Jira ticket in `_ticket.md` format and must extract knowledge from it.
>
> **Project background:** A long-running engineering project with multiple SVN repositories — a harness application and an associated test suite.
>
> Active Jira keys span harness/CI work, test-suite maintenance, certification-tooling work, and the two current ATSC3 certification cycles. Several historical / read-only project keys appear in SVN commit messages from earlier eras of the same product line; the project background supplied at session start should enumerate them with their date ranges so the analyser can classify era correctly.
>
> ---
>
> **Step 1 — Score the ticket (1–5):**
>
> | Score | Criterion |
> |-------|-----------|
> | 5 | Explicit decision with rationale and rejected alternatives documented |
> | 4 | Significant architectural or process change with some rationale |
> | 3 | Fix or improvement with context explaining why |
> | 2 | Routine maintenance, status-only update, or trivial change |
> | 1 | Typo fix, admin-only, closure comment only |
>
> If score ≤ 2, output `{"score": N, "extractions": []}` and stop.
>
> ---
>
> **Step 2 — Extract facts.**
>
> For each extraction produce a JSON object:
>
> ```json
> {
>   "destination": "wiki path (e.g. platform/package-dependencies.md)",
>   "content": "the fact written in wiki prose — not a ticket summary, the fact itself",
>   "source_ref": "PROJ-388",
>   "author_credit": "Person Name (optional — include only if the insight is clearly attributed to one person)",
>   "tags": ["package", "constraint"],
>   "score": 4
> }
> ```
>
> **Valid destination paths:**
>
> | Path | What belongs here |
> |------|------------------|
> | `platform/overview.md` | What the platform is, certification schemes |
> | `platform/architecture.md` | Django app, extension system, ASGI |
> | `platform/package-dependencies.md` | Dependency decisions, constraints, current versions |
> | `platform/infrastructure.md` | Hardware fleet, CI servers, file shares |
> | `platform/deployment.md` | Hardware build, kernel pinning, bootstrap, dongle |
> | `platform/development.md` | Dev env, workflows |
> | `test-suite/structure.md` | Test-suite layout, suites/, svn:externals |
> | `test-suite/test-anatomy.md` | master.yaml, launch.yaml, script.py |
> | `test-suite/build-pipeline.md` | Suite build, validate, publish |
> | `pipelines/platform-ci.md` | Platform CI package, smoke tests, toggles |
> | `pipelines/atsc3-build.md` | 4-stage × 4-profile consolidated pipeline |
> | `pipelines/security-release.md` | PCAP refresh pipeline |
> | `decisions/{YYYYMMDD}-{slug}.md` | Decisions with explicit rationale (use Prompt D) |
> | `history/evolution.md` | How the system got to its current shape |
> | `history/infrastructure-changes.md` | Decommissioned kit, server migrations |
> | `known-issues/index.md` | Active issues, workarounds, blocked items |
>
> ---
>
> **What to extract:**
>
> - The **why** behind a decision — constraints not obvious from the code
> - Accepted patterns: "this is how we do X here"
> - Rejected alternatives and the reason they were rejected
> - Workarounds: what triggers them, whether they're temporary
> - Package version changes **only if** there was a breaking API change requiring code adaptation
>
> **What NOT to extract:**
>
> - Ticket status changes or workflow history
> - Comments that only say "done", "LGTM", "thanks", or equivalent
> - Facts the code already makes self-evident
> - Step-by-step procedures (those belong in repo documentation, not the wiki)
> - Predictions or intentions that may not have been implemented
> - Customer-identifying details (for Redmine tickets)
>
> ---
>
> **Example of a good extraction:**
>
> ```json
> {
>   "destination": "platform/package-dependencies.md",
>   "content": "`invoke` must remain in `requirements.txt` (not `requirements_dev.txt`) because the production install flow in `remote_install.py` installs only `requirements.txt`, then immediately calls `invoke` to run remaining install steps. Moving `invoke` to dev-only would break production installs silently.",
>   "source_ref": "PROJ-388",
>   "tags": ["package", "constraint", "production-install"],
>   "score": 4
> }
> ```
>
> **Example of a bad extraction (do not produce this):**
>
> "PROJ-388 upgraded Fabric from 2.7.1 to 3.2.3 and removed patchwork." — This is a ticket summary, not a wiki fact. The wiki fact is the constraint (why invoke stays in requirements.txt) and the pattern (why fabric is dev-only), not the version numbers.
>
> ---
>
> **Output:** A JSON array of extraction objects, or `{"score": N, "extractions": []}` for low-value tickets.

---

## Prompt B — SVN log batch

> You are building the project knowledge wiki. You have been given a batch of SVN commit summaries in `summary.txt` format.
>
> **Summary.txt format:**
>
> ```
> --------------------------------------------------------------------------------
> Revision: 178962
> --------------------------------------------------------------------------------
> Author:   developer.name
> Date:     Tuesday, April 14, 2026 16:25:35 (2026-04-14 16:25:35 +0100)
> Message:
> PROJ-388 Upgrade Fabric/Invoke/Patchwork stack to Fabric 3
> ----
>   Modified: /harness-repo/trunk/install/requirements.txt
>   Added:    /harness-repo/trunk/install/python_packages/fabric-3.2.3-py3-none-any.whl
>   Deleted:  /harness-repo/trunk/install/python_packages/patchwork-1.0.1-py2.py3-none-any.whl
> ```
>
> **Jira project key history for interpreting messages:** Supply at session start an enumeration of active and historical project keys with their date ranges and brief role descriptions. Old commit messages reference keys that no longer correspond to active projects.
>
> ---
>
> **Step 1 — Classify each commit:**
>
> | Class | Description |
> |-------|-------------|
> | Milestone | First or last commit of a major component; infrastructure migration; breaking change |
> | Coupling | Part of a pattern: this file always changes with that file |
> | Pivot | Team changed direction (renames, deletions, sudden cluster then silence) |
> | Iteration | Multiple commits to same file within hours — noise, not signal |
> | Low value | Pure wheel add/delete, typo fix, svn:mergeinfo only |
>
> **Step 2 — Aggregate iterative commits.** If 5+ commits touch the same file within one day, treat them as one event. Extract ONE fact about the outcome, not 5+ facts about the iterations.
>
> **Step 3 — Extract facts from milestone, coupling, and pivot commits.** Use the same JSON format as Prompt A, with `"source_ref": "r178962"`.
>
> ---
>
> **What to extract:**
>
> - **Milestone:** what component was born, retired, or significantly changed; what the revision anchors in the timeline
> - **Coupling:** "Changes to X typically require corresponding changes to Y (observed across N commits, date range)"
> - **Pivot:** what the team moved away from, what they moved to, and when
>
> **What NOT to extract:**
>
> - Pure wheel file add/delete (package bumps without architectural decision)
> - `svn:mergeinfo` changes (branch infrastructure, not knowledge)
> - Revert commits (document the original direction change, not the revert)
> - Commits whose message is only a ticket ID with no other text (low confidence)
>
> ---
>
> **Output:** A JSON array of extraction objects (same format as Prompt A), or `{"extractions": []}` if the batch contains no high-signal commits.

---

## Prompt C — Old wiki page (WikiCreole)

> You are building the project knowledge wiki. You have been given a page from the team's historical internal wiki in MediaWiki/WikiCreole format.
>
> **WikiCreole markup quick reference:**
> `= H1 =`, `== H2 ==`, `{{{code}}}`, `[[WikiLink]]`, `[[text|URL]]`. Discard `<<TableOfContents>>` and `<<Anchor(...)>>`.
>
> **Company/server name history:** Supply a mapping of old hostnames and company names to current equivalents (see `source-formats.md`). This is needed to interpret old wiki references that predate infrastructure migrations or company renames.
>
> ---
>
> **Step 1 — Classify the page:**
>
> | Type | Approach |
> |------|----------|
> | Procedural (step-by-step how-to) | Extract **facts embedded in procedures**, not the steps themselves |
> | Reference (glossary, naming conventions, index) | Extract directly |
> | Historical narrative | Extract directly |
> | Outdated announcement | Discard |
>
> **Step 2 — Extract.** Same JSON format as Prompt A. Add an `"era"` field:
>
> Era values should be supplied at session start, mapping date ranges to the project codenames in use during each era. Use these to tag each extraction with the era it originated from.
>
> ---
>
> **From procedural pages, extract:**
>
> - Naming conventions and their rationale (e.g., production-vs-dev hostname patterns)
> - Physical/system constants that encode architectural decisions (e.g., specific kernel versions that must be pinned, and why)
> - Constraints stated in cautions ("DO NOT connect to internet during install BECAUSE...")
> - Infrastructure facts: server names, file paths, credential structures, what has been decommissioned
>
> **From procedural pages, do NOT extract:**
>
> - The steps themselves (may be stale)
> - Specific Ubuntu package versions or command flags (likely outdated)
> - Individual commands unless they encode a non-obvious architectural constraint
>
> ---
>
> **Example of a good extraction from a procedural page:**
>
> ```json
> {
>   "destination": "platform/deployment.md",
>   "content": "During Ubuntu installation on the deployment hardware, the machine must be disconnected from the internet before the installer boots. If connected, Ubuntu pulls the latest kernel from its mirrors, which will differ from the kernel version that the DekTec driver `.ko` file in the release tarball was compiled against. Disconnecting is intentional kernel pinning, not an oversight.",
>   "source_ref": "devices-wiki: ProductionDeployment",
>   "era": "2021-2024",
>   "tags": ["deployment", "constraint", "dektec", "kernel"]
> }
> ```
>
> ---
>
> **Output:** A JSON array of extraction objects (same format as Prompt A, plus `"era"` field).

---

## Prompt D — Decision record

Use this when a Prompt A analysis produces a score-5 extraction with explicit alternatives and rationale. Instead of an inline wiki fact, produce a standalone decision record file.

> You are building the project knowledge wiki. Based on the following ticket analysis extraction, produce a decision record in the format below.
>
> **Decision record format** (`decisions/{YYYYMMDD}-{slug}.md`):
>
> ```markdown
> # Decision: {title}
>
> **Date:** YYYY-MM-DD
> **Status:** Accepted
> **Source:** {TICKET-ID}, {author if named}
> **Tags:** {topic tags}
>
> ## Context
> What situation made this decision necessary. One short paragraph.
>
> ## Decision
> What was decided. One or two sentences.
>
> ## Rationale
> Why this option was chosen over the alternatives.
>
> ## Alternatives considered
>
> - **Option A** — rejected because…
> - **Option B** — rejected because…
>
> ## Consequences
> What this decision constrains or enables going forward. Include any known expiry conditions (e.g., "until v2.2 ships").
> ```
>
> After producing the decision record, also produce an index row for `decisions/index.md`:
>
> ```markdown
> | {YYYY-MM-DD} | [{title}]({YYYYMMDD}-{slug}.md) | {tags} | {TICKET-ID} |
> ```

---

## Prompt E — Pre-analysed investigation document

When a human-produced or AI-produced investigation document (like `PROJ-390/investigation.md`) already synthesises source material into structured findings, use this lighter prompt:

> You are building the project knowledge wiki. You have been given a pre-analysed investigation document that synthesises SVN history, CI/CD job configs, and/or ticket content.
>
> The hard analysis work has already been done. Your task is to:
>
> 1. Identify which sections of the investigation map to which wiki destinations.
> 2. For each section, rewrite the finding in wiki prose (not investigation prose — the wiki states facts, the investigation tells a story).
> 3. Identify which findings warrant a `decisions/` record (Prompt D) vs. an inline fact.
> 4. Discard the narrative framing; keep only the facts and decisions.
>
> Use the same JSON extraction format as Prompt A.
>
> **Tip:** Investigation documents often have a "recommendations" or "what was lost" section that maps directly to `known-issues/index.md`. Treat these as the most actionable extractions.
