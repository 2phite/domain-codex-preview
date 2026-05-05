# Consumers

> How the index becomes value. Two derived artefacts, one curation contribution path, several consumer surfaces. The architecture only matters if a consumer reaches it.

The *why* of two artefacts (live retrieval vs static wiki) is in [why.md](why.md#why-two-outputs). This doc is what each consumer actually looks like, what shape its output takes, and how curation gets *into* the system rather than just *out*.

## The async-only constraint

Hybrid retrieval runs in the tens of seconds at corpus scale, and is **prefill-bound at large context windows** — the dominant cost is feeding retrieved context into the model, not generating the answer. Decode-side accelerators don't help; sub-10s hybrid is structurally unavailable on commodity single-GPU hardware (see [`architecture.md`](architecture.md#async-only-consumer-constraint)).

Real-time chat is not a viable consumer shape for this system. Designing around the async constraint is part of the architecture, not a workaround.

The viable consumer shapes are all asynchronous in some form: the user submits a question and reads the answer when ready, or the consumer runs unattended on a webhook or schedule.

## Live retrieval consumers

These query the index directly. Each has its own latency budget, scope, and output format.

### IDE assistant sessions

The pattern that already works. A developer in their IDE asks a question; the assistant calls the retrieval API plus query-time tools (history fetch, ticket fetch); the answer arrives in seconds-to-a-minute. The user is already adapted to multi-second turns.

Latency budget: generous. Scope: whatever the developer's question implies. Output: conversational, citation-bearing.

This is also the **lowest-friction consumer to validate**. New retrieval prompts, new index versions, gatekeeper changes — all get exercised here first because the developer running the session is also the person who can tell whether the answer was useful.

### Ticket-drafting webhook

A new ticket arrives. A webhook fires, the consumer reads the ticket, queries the index for context, calls query-time tools for specifics, and produces a **draft reply** for human review. The reply goes into a comment marked as bot-authored, or into a sibling document for the assignee to edit before posting.

Latency budget: minutes. The webhook is asynchronous by design — no human is waiting on the response.

**Quality target:** the draft should look like an investigation document — synthesising across tickets (related items, prior decisions), code (structural queries on the affected modules), version-control history (recent changes touching the relevant files), and specs (relevant clauses if applicable). Citations are mandatory; the reviewer needs to verify, not trust.

This is the consumer where the **investigation-style synthesis** matters most. Engineers already produce this kind of analysis manually for hard tickets; the webhook industrialises that work. If reviewers consistently rewrite the draft from scratch instead of editing it, retrieval is shallow and the system isn't earning its keep — that's the warning sign named in [why.md](why.md#warning-signs-during-operation).

### Wiki regenerator

A scheduled batch process re-prompts the generator over the current index, producing the static wiki (next section). Runs on demand or on a weekly/monthly cadence. No latency budget — minutes-to-hours is fine.

### Chat-platform agent (future)

A bot that answers `@`-mentions in team chat threads. Async by necessity — even if the user is waiting, the bot replies in 30–90s. Useful for "have we hit this before" / "where is X documented" / "who knows about Y" questions.

Constrained the same as the IDE consumer — latency-tolerant, scope-flexible — but with a much wider audience that hasn't self-selected for technical patience. The bot needs to handle "I don't know" gracefully and cite sources prominently.

### What's *not* a viable consumer

- **A real-time chat that responds in under 5 seconds.** The prefill bottleneck rules it out.
- **Anything blocking a user-facing action.** A login flow, a ticket-creation form, a deploy gate. The latency floor is too high.
- **A "natural language API" exposed to external services with SLA.** Not unless the consumer accepts the latency and its variance.

## The static wiki

A periodically-regenerated, navigable, narrative artefact for humans reading the project cold. It is the consumer for *new joiners, managers, and people outside the immediate team*.

### Shape

- **Per-project.** One static wiki per project pilot (see `projects/<name>/`). Cross-project knowledge stays in the meta layer here.
- **Structured as a small number of long pages**, not many short ones. Long pages let cross-references stay in the same scroll; many short pages force navigation that nobody does.
- **Editorial in voice.** Not a transcript of retrieved chunks. The generator's job is to *write*, not to compile. A page with seven half-sentence bullet points and no narrative thread is a failed generation, not a successful one.
- **Citation-dense.** Every claim links to its corpus source.
- **Regenerated, not edited.** Any human-authored content lives in the curation overlay (which is then re-ingested) — never as a manual edit to a generated page.

### Page taxonomy (target shape, not authored)

The shape the *generator aims for*, not a directory humans fill in:

- **Why this project exists** — the strategic intent, key constraints, what the project is/isn't.
- **Architecture overview** — major components, how they connect, where the seams are.
- **Decision log** — major decisions with rationale, including rejected alternatives.
- **Operational guide** — how to build, deploy, troubleshoot common failures.
- **People and ownership** — who knows what, current and historical.
- **Glossary** — project-specific vocabulary, acronyms, internal jargon.
- **Active threads** — what's currently being worked on, blocked-on-what.
- **Open questions** — things the corpus cannot answer; explicit invitations for curation contribution.

The generator is given the page taxonomy as part of its prompt; the prompt is the design surface. Re-prompting with a new taxonomy is how the wiki structure changes.

### Coverage gaps as curation seeds

When the generator can't fill a section, it doesn't fabricate. It writes a placeholder noting *what's missing and why* (the corpus didn't carry the answer). Those placeholders are explicit invitations to the curation overlay — "we don't know who decided X; if you do, add it." This couples the generator's blind spots to the curation contribution path directly.

## Curation contribution

The open design question. There is a strong claim that curation matters (see [why.md](why.md#why-a-curation-layer)) and a weak claim about how it gets contributed.

### The contribution-interface problem

A curation entry is *small, structured-enough-to-index, optionally cross-linked to corpus items*. The interface choice determines who contributes and how often.

| Interface | Friction | Discoverability | Structure |
|-----------|----------|-----------------|-----------|
| Pull request to a markdown directory | High | Low | High |
| Web form ("add an annotation") | Low | Medium | Medium |
| Chat-platform `@bot` slash command | Lowest | High | Low |
| Inline annotation on a wiki page | Low | High | Medium |
| Email-to-bot ("send your complaint") | Lowest | Low | Lowest |

No single interface dominates. High-friction-but-structured interfaces (PRs) get content that's well-formed but rare. Low-friction-but-unstructured interfaces (`@bot`, email) get frequent contributions that need post-processing.

### Proposed first version

- A **lightweight web form** for structured contributions ("annotate a ticket", "flag a deprecation", "add a 'who knows' entry").
- A **chat-platform `@bot` slash command** for unstructured but discoverable contributions ("@codex remember: X about Y").
- Both write to the same backing store. An LLM normaliser passes over unstructured entries to give them indexable shape.

This is a guess. The real first version is the one that gets deployed and used; the second version is shaped by what friction the first version revealed. The design discipline is **deploy something contributable to early**, not **design the perfect interface up front**.

### What contribution-failure looks like

Covered in [why.md's warning signs](why.md#warning-signs-during-operation):

- Curation overlay carries entries from one or two people only → contribution path is too narrow; redesign the interface, don't lecture people about contributing.
- Curation overlay is empty → either the corpus is gap-free (unlikely) or no one is using the system enough to notice gaps. Adoption metric, not content metric.

## Quality targets

What "good" looks like for each consumer's output. The same targets that drive the [success criteria in why.md](why.md#what-good-looks-like).

- **Investigation-style draft** (ticket-drafting webhook): synthesises across tickets, code, version-control history, and specs with explicit rationale and citations. The reviewer edits, doesn't rewrite.
- **Static wiki page**: narrative, navigable, scrollable in one read for any single page, citation-dense. A new joiner can read the wiki cold and answer their own first-week questions.
- **IDE chat answer**: directly answers the developer's question, with the source citations to verify. Concise; the developer is busy.
- **Chat-bot reply**: discovers prior context the asker didn't know existed. The most valuable answers are of the form "this was discussed in `<ticket-key>` in `<date>`, see comments by `<person>`."

## Rejected

### Rejected: real-time chat as a primary consumer

A latency floor in the tens of seconds makes it structurally unviable. Designing for it would compromise retrieval coverage in pursuit of an unachievable target.

### Rejected: one artefact for all consumers

A "queryable wiki" that humans read AND bots query. Conflates two incompatible UX requirements (see [why.md](why.md#why-two-outputs)). The artefact ends up good for neither — slow to read, shallow to query.

### Rejected: PR-only curation

A curation-via-pull-request workflow as the only contribution path. Friction is too high; adoption metric will fail. PR-style contribution is fine as one channel among several, but never the only one.

### Rejected: regenerating the static wiki on every ticket ingest

Wasteful and produces noise (every regeneration drifts slightly). Schedule it; coupling to the curation overlay is what keeps it fresh in human-relevant ways.

## Deferred decisions

- **Specific contribution-interface implementations.** Web form framework, chat-platform app architecture, normaliser model. First version's job is to be deployed, not to be optimal.
- **Static wiki regeneration cadence.** Manual trigger only, scheduled, or triggered by curation-overlay churn? Decided once we have one wiki to regenerate.
- **Multi-project consumer surfaces.** A bot that knows about multiple projects, an IDE plugin that switches contexts. Single-project first; multi-project pattern is itself a deferred design problem.
