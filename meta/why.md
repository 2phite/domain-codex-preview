# Why domain-codex exists

> The strategic intent of this project: the problem it addresses, and the alternatives that were considered and rejected. If you read only one document in `meta/`, read this one.

## The framing question

> **What can a domain expert assigned to a project do for everyone?**

Every long-running engineering project accumulates a domain expert — the person who has been there longest, has read the most tickets, remembers the hardware quirks, knows which decisions were made in which chat thread. Their colleagues route questions to them. New joiners shadow them. Managers consult them. They become the project's de facto memory.

This works, but it doesn't scale. The expert is one person. They take vacations. They eventually leave. Their answers vary by mood and recency.

And the painful part: nothing about what they know is secret. The raw material — tickets, code, version-control history, specs, internal docs — is sitting in plain sight, available to anyone who would read it. The expert just happens to be the only one who has read it all and joined it together.

The question this project answers: how do we build infrastructure that takes the same raw material the expert reads, performs the same synthesis, and exposes the result to everyone — through whichever interface they prefer?

## What we are building, in one sentence

A **searchable index** built from a project's chaotic written record (tickets, version-control history, code, specs, internal docs), plus a **curation layer** for what the written record cannot capture. Two outputs are generated from it: a queryable backend that AI consumers (assistants, chat bots, ticket-drafting webhooks) call on demand, and a regenerated static wiki that humans read cold.

## Why each piece is necessary

### Why multiple sources

No single source is complete:

- **Code** tells you *what* the system does.
- **Tickets** tell you *why* it does it that way.
- **Version-control history** tells you *when* and *who*.
- **Specs** tell you what was supposed to be done versus what actually shipped.
- **Internal docs** tell you what someone reached for as a summary at a moment in time.

Each source on its own answers only the questions its own search box already answers, badly. The expert's value is the cross-source linkage: *"this code constraint exists because of that ticket because of that spec clause."*

There is also a deeper point. Each source has a different way of being incomplete. Tickets are biased toward what was important enough to file. Code is honest about what runs but silent on intent. Version-control history is honest about timing but minimal on rationale. Specs drift from what shipped the moment release pressure starts. Internal docs go stale because nothing forces them to be regenerated when the world around them changes.

These different incompleteness patterns do not overlap. Where the sources agree, you have signal. Where they disagree, you have an interesting question. Multi-source is not "more is better"; it is *different liars triangulate*.

A practical addition: not every source belongs in the standing index. Some — full version-control history, raw build logs, large attachments — are too voluminous to pre-process, but become useful when something pulls a *targeted slice* in the act of answering a question. The index carries the narrative substrate; on-demand tools carry the long tail. (See [architecture.md](architecture.md) for how this split works.)

### Why a combined search method

The index needs to handle two kinds of question:

- *Lookup questions* — "what does function X do" — answered by surface-similarity search over text.
- *Traversal questions* — "why was approach A rejected" — answered by following relationships across multiple tickets and code modules.

A single search method handles only one. Combining semantic search with a graph of relationships handles both. Each on its own would leave half the consumer's needs unmet.

### Why a curation layer

A corpus has gaps. Tickets capture *decisions formal enough to file* but miss casual chat-resolved decisions, "ask the platform owner about X" tribal knowledge, and the unwritten reasons hacks exist. The team's residual knowledge — not just the expert's — is what the corpus alone cannot tell you.

The curation layer is where that residue is captured: deprecation flags, "ask the on-call first," workaround context, decision rationales that never made it into a ticket. It is the *only* part of the system humans author directly; everything else is derived.

The contribution interface — *how* humans get curation entries into the system — is a design problem in its own right (see [consumers.md](consumers.md)).

### Why two outputs

Live retrieval and a static wiki have incompatible requirements.

A live retrieval consumer can take its time — accuracy matters more than latency when a webhook is drafting a ticket reply, or when an assistant is grounding an answer for a developer.

A static wiki for humans reading cold must be navigable, editorial in voice, and immediately legible. A new joiner cannot wait minutes per question; they need a coherent narrative they can scan.

A single output cannot satisfy both. Conflating them produces something that is both a slow query backend and an unreadable wiki.

## What was rejected, and why

### Rejected: edit each ticket into the wiki as it is read

An incremental approach where the system reads each ticket and edits a growing collection of wiki pages. This produces an output that grows alongside ingestion. It sounds appealing and is wrong:

- **Compounding drift.** Pages written early have less context than pages written late. There is no way to retrofit early pages with what was learned later, short of redoing all the intermediate work.
- **Order dependence.** Reading tickets in a different order produces a different wiki. The output carries the history of *how it was built*, not just *what is true*.
- **Source mutation is missed.** Tickets get new comments and edits. A write-once incremental wiki has no way to revisit a page when its source has changed.
- **Strategy changes are unaffordable.** Halfway through, decide the wiki should be structured differently? Every prior page has to be migrated by hand.

The replacement is **generate, do not accumulate**. Ingest only updates the index, never the output. The wiki is a derived view, regenerated on demand. Change the structure? Re-prompt and regenerate. There is no edit trail to migrate.

### Rejected: human-curated docs as the source of truth

A subtler version of the same mistake: keep a human-curated document set as the canonical knowledge, with the index as a sidecar.

This assumes humans are reliable narrators. They are not — at least not for this purpose. A domain expert's memory is biased by recency, by which decisions they personally fought for, by what was vivid versus what was routine. The ticket corpus is *less biased than memory* because it captures the routine alongside the dramatic. Humans should *annotate* the corpus-derived narrative, not author it.

This determines who writes what. Corpora write the narrative. Humans write the annotations. Generators write the consumable outputs.

### Rejected: one output for everyone

Conflating live retrieval and the static wiki produces an output optimised for neither. See "Why two outputs" above.

### Rejected: cloud-only extraction (for now)

While extraction prompts are still being tuned, iteration speed dominates throughput. Local extraction costs nothing per call — and most ingest runs during this phase will be re-runs after a prompt tweak. Cloud per-call costs would distort the cost of *experimenting*: the project would start optimising against the bill rather than against output quality. Local-only also defers the data-handling question of sending content to a third-party provider until that decision actually needs to be made. The cloud option remains open; it just is not earned yet.

### Rejected: a code-only graph

A structural graph over code is necessary for code-side reasoning. It is not sufficient. It can tell you what calls what, but not why a function has a strange name, which ticket forced an awkward parameter, or what spec clause demanded a particular constraint. A code-only graph has no view into non-code narrative; the consumer would still have to read tickets and specs by hand to bridge the gap. Code-side reasoning belongs alongside narrative reasoning, not in place of it.

## What good looks like

If `domain-codex` succeeds, the following should be observable within months for a single project, with cross-project adoption following over a longer horizon.

- **Question routing changes.** The team's habit of routing "why does X work this way" to the expert visibly shifts — the bot or the wiki gets asked first; the expert is consulted second, on the residual.
- **The expert's calendar shifts.** Less time spent restating things the corpus already contains. More time spent annotating gaps, validating drift, and resolving cross-source contradictions.
- **New joiners self-serve in their first week.** A new joiner reading the static wiki cold can answer most of their own onboarding questions before asking anyone, and arrives at their first standup with informed questions rather than blank ones.
- **Drafts get edited, not rewritten.** When a ticket-drafting bot produces a first-pass reply, the human reviewer edits it. If reviewers consistently rewrite from scratch, retrieval is shallow and the system is not earning its keep — *which version happens* is itself a measurable signal of grounding quality.
- **A second project adopts the pattern.** Following these meta docs alone, with no hand-holding, a second team configures their own corpus, runs their own ingest, and produces their own codex. This is the test of whether the architecture is project-agnostic in practice or just in claim.

The thread connecting these: the value of having a domain expert assigned to a project compounds across the team, rather than concentrating in one person who eventually leaves.

## Warning signs during operation

The sections above are shaped to avoid the failure modes that come from *building the wrong thing*. These are the failure modes that come from *building the right thing and watching adoption fail* — symptoms to watch for once the system is running.

- **The curation layer carries entries from only one or two people.** Either the rest of the team has too much friction to contribute, or the contribution path is too narrow. Redesign the contribution interface; do not lecture people about contributing.
- **The curation layer is empty.** Either the corpus is genuinely gap-free (unlikely), or no one is using the system enough to notice gaps. Treat as an adoption signal, not a content one.
- **Bot answers grow longer over time.** Retrieval is widening rather than sharpening. The filtering criteria probably need updating, or the index has accumulated low-signal entries that need pruning.
- **New joiners stop using the static wiki after week one.** The output lost coherence somewhere. Regenerate with a different prompt or split it into smaller views before assuming the format itself is wrong.
- **Bot drafts get rewritten rather than edited.** Grounding is shallow. The retrieval the drafter saw did not carry the project-specific context that turns "plausible" into "useful."
- **The bot is consistently asked questions the corpus does not cover.** Worth ingesting that source — the team is telling you where the gaps are by asking around them.
