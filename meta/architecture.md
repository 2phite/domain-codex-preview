# Architecture

> The structural shape of `domain-codex`. For *why* this shape was chosen over alternatives, read [why.md](why.md) first — this doc names the components and how they connect, not the rationale for needing them.

## Two orthogonal axes

The architecture has two independent decompositions, both load-bearing:

1. **Pipeline stages** — ingest → index → generate → curate. The temporal flow of a single piece of content from raw source to consumable artefact.
2. **Retrieval mode** — *indexed corpus* vs *query-time retrieval tools*. The standing index carries the narrative substrate; query-time tools carry the long tail.

Most discussion of RAG architectures collapses these into one axis ("ingest everything, then query"). Treating them as orthogonal is what makes the design honest about scale.

## Pipeline stages

### Ingest

Raw sources are heterogeneous: tickets, code, version-control diffs, PDFs, images, build-system configuration. The index needs structured, narrative-shaped text. Ingest converts.

The pipeline for non-code text:

```
source fetch (per-source ingest tools)
  → format normalisation (source markup → markdown, strip UI noise)
  → optional non-text expansion (document parser + vision captioning for PDFs/images)
  → entity-and-relationship extractor + embedder
  → text-index store
```

Code follows a separate path: an AST-driven indexer parses source files into a structural graph emitted as a per-repo artefact. No LLM extraction — purely structural.

Ingest is **idempotent**. Re-running ingest on already-indexed sources updates the store; it never edits a derived artefact. This is the polarity correction described in [why.md](why.md#rejected-edit-each-ticket-into-the-wiki-as-it-is-read).

### Index

The index is what consumers query. It is the union of two specialised stores:

- **A hybrid graph+vector text index** over non-code content. Backs natural-language queries with cross-document reasoning.
- **An AST graph** over code. Backs structural queries (call graphs, paths between symbols, module-level explanations).

Both feed the same generators. Cross-store reasoning ("this code constraint is here because of that ticket") happens at the *consumer* level — the LLM consuming both indices does the linking.

There is no unified meta-index. Pulling code-as-text into the text index, or narrative tickets into the AST graph, is rejected (below). The two indices stay specialised; the consumer is the linker.

### Generate

Generators produce derived artefacts on demand from the index:

- **Static wiki pages** — long-form narratives over a project's corpus. Regenerated periodically.
- **Ticket-draft replies** — first-pass responses to incoming tickets, drawing on indexed context plus query-time tool calls for specifics.
- **Investigation-style synthesis** — on-demand answers to deep cross-source questions, in the shape of a hand-written investigation document.

The generation prompt — not the output — is the intellectual artefact worth maintaining. Outputs are regeneratable; prompts are the design surface.

### Curate

The curation overlay holds what the corpus cannot say (see [why.md](why.md#why-a-curation-layer)). Curation entries are themselves indexed and become retrievable alongside corpus content. The *contribution interface* — how humans get curation entries into the store — is the design problem [`consumers.md`](consumers.md) addresses.

## The retrieval-mode split

Not every source belongs in the standing index. Two regimes:

### Indexed corpus

Pre-processed, embedded, sitting in the text index ready for semantic+graph retrieval. High signal, narrative-shaped, fits in retrieval. See [`corpus.md`](corpus.md) for the source list and inclusion criteria.

### Query-time retrieval tools

Voluminous or episode-specific sources, exposed as **tools the LLM consumer invokes on demand**, scoped by the question being asked:

- A version-control-history tool — pulls a targeted slice of commit history (by ticket key, revision range, path, or date window). Consumer decides scope.
- A ticket-fetch tool — fetches a specific ticket's full content (changelog, attachments, edit diffs) when the indexed summary isn't enough.
- A build-log tool (per-project, where applicable) — pulls a build-log slice for a specific build or date range.

The consumer (an IDE assistant, a webhook, a bot) treats these as part of its tool palette. When a webhook drafts a ticket reply, it queries the index first to get the project narrative around the ticket, then pulls the actual diffs the ticket discusses.

This split is the answer to the "ingest everything" anti-pattern. Pre-ingesting full version-control history would bury the narrative under low-signal volume; ignoring it entirely would leave consumers blind to specifics. The split delivers both.

### What goes where

A source belongs in the **indexed corpus** if all hold:

- It carries narrative or rationale (not just events or evidence).
- Its full body is small enough that pre-processing the whole thing pays off across many queries.
- It changes slowly enough that re-ingestion cadence stays manageable.

It belongs to **query-time tools** if any hold:

- It's voluminous and only narrow slices are usually relevant (full commit history, build logs).
- It's authoritative-by-being-current (live ticket state, not last-ingested snapshot).
- The cost of pre-processing it would dominate the value retrievable from it.

## Topology

`domain-codex` is a **client/server system**, not a single-machine application. The split is forced by GPU economics:

```
┌─────────────────────────────┐         ┌──────────────────────────────┐
│ Ingest+Index server         │         │ Consumer clients             │
│ (GPU-bound)                 │  HTTP   │ (user-bound)                 │
│                             │ ←──────→│                              │
│ - Document parser           │         │ - IDE assistant              │
│ - Extractor + embedder      │         │ - Chat bot (future)          │
│ - Text-index store          │         │ - Ticket-drafting webhook    │
│ - Retrieval API             │         │ - Wiki regenerator (batch)   │
│                             │         │                              │
│ ▲ batch ingest workers      │         │ Each consumer also calls     │
│ ▼ query-handling workers    │         │ query-time tools directly    │
│   (share VRAM)              │         │ (history fetch, ticket fetch)│
└─────────────────────────────┘         └──────────────────────────────┘
```

### Why this split

- **Ingest is GPU-bound.** The document parser's vision model, the extractor LLM, and the embedder all want VRAM. Co-locating them on a single GPU host is the only viable shape until a cloud-extraction path is earned.
- **The index lives with the ingester.** Co-location avoids remote-write coordination — the ingester writes directly to the local store; consumers query via HTTP.
- **Consumer surfaces want to run where users are.** IDE plugins on the developer's machine, webhooks in cloud-adjacent infra, chat bots wherever the chat platform allows them. None of these need GPU.

### Concurrency model

Ingest workers and query workers compete for the same VRAM. The pragmatic policy:

- Ingest runs as a **batch worker**, queued.
- Query traffic gets priority.
- During heavy ingest (a full corpus rebuild, or a non-text batch with the document parser loaded), query latency may degrade. Schedule ingest off-hours where possible.

This is operational reality, not architectural ideal. A second GPU host removes the contention; until then, batch-and-yield is the discipline.

## Component inventory

By role, not product. Per-project tool choices for each role live in `projects/<name>/`.

| Role | Responsibility | Where it runs |
|------|----------------|---------------|
| **Hybrid text index** | Graph+vector index over non-code content. LLM-driven entity/relationship extraction at ingest, semantic+graph retrieval at query. | Server |
| **AST code graph** | Structural index over code. Per-repo node/edge graph queried by path / explain / list operations. | Per-repo working copy |
| **Document parser + non-text ingest layer** | Parses PDFs, images, attachments. Vision captioning turns non-text regions into text the index can consume; vision model runs at ingest only. | Server |
| **Extractor model + embedder** | Hosted behind an OpenAI-compatible endpoint. Provider-agnostic — local host can be swapped for a hosted equivalent. | Server |
| **Per-source ingest tools** | Fetch tickets, version-control history, build configs into ingest-ready form. Maintain per-source state (last-ingested timestamps, edit ledgers). | Either side |
| **Query-time fetch tools** | Pull narrow slices of voluminous sources (full ticket bodies, commit ranges, build logs) on demand from a consumer. | Consumer-side |

## Async-only consumer constraint

Hybrid retrieval is observed to run in the tens of seconds at corpus scale, and is **prefill-bound** at context windows around 30k tokens — the dominant cost is feeding retrieved context into the model, not generating the answer. Decode-side speed-ups (speculative decoding, multi-token prediction) target the wrong bottleneck and don't materially help. Sub-10s hybrid retrieval is not achievable on commodity single-GPU hardware.

This makes async UI a structural design constraint, not a temporary one. Real-time chat is ruled out as a primary consumer. The viable consumer shapes are:

- **Webhook drafts** — async generation, user reads when ready.
- **On-demand wiki regeneration** — minutes is fine.
- **Batched bot replies** — user posts a question, bot answers in 30–90s.
- **Async IDE chat** — the user is already used to multi-second turns.

The only available latency lever is **retrieval scope** (k, chunk size, hybrid-vs-naive mode). Each consumer can be tuned for its own coverage/latency tradeoff. This is a per-consumer choice, not an infrastructure setting.

## Rejected

### Rejected: single-machine architecture

Putting ingest, index, and consumer surfaces on one box. GPU-bound ingest blocks user-facing surfaces, and consumer surfaces want to run where users are (IDE, chat platforms), not where the GPU is. The architecture is a client/server system; co-locating everything is a pilot-time shortcut, not a target shape.

### Rejected: one unified meta-index across code and text

Pulling code-as-text into the text index, or narrative tickets into the AST graph, to give consumers a single retrieval surface. Each tool's strengths come from its specialisation: structural code queries lose value when the graph is full of narrative noise; narrative reasoning loses value when 60% of the corpus is function definitions. The consumer is the linker; the indices stay focused.

### Rejected: wrapping the local document parser in a custom service

Real engineering effort that doesn't earn its keep at pilot scale. Most local document parsers have no native server mode; running the whole ingestion pipeline on the GPU host, orchestrated via SSH or a thin job queue, is the simpler shape. If operational pain demands a real API boundary, the swap is to a cloud document-intelligence service — that path replaces the local parser entirely with a real API rather than wrapping it.

### Rejected: pre-ingesting all sources

Pulling full version-control history, all build logs, all attachments into the index. The long tail dilutes the narrative substrate and inflates retrieval scope past usable thresholds. The query-time-tool split is how we keep both.

## Deferred decisions

These are named here so they aren't hidden. The decision points themselves live in adjacent docs:

- **Index partitioning at scale.** Whether to run one text index per sub-corpus boundary (e.g., per Jira project) versus one unified store. Driven by [`corpus.md`](corpus.md#validation-pass)'s validation pass — graph traversal degrades past ~1k nodes, and typical extraction rates make even single-project sub-corpora candidates for partitioning.
- **Cloud extraction swap.** Local extraction is cheap-per-call and fast-to-iterate while extraction prompts are still being tuned. The cloud path becomes earned the moment scale or operational pain demands a real API boundary. The architecture stays compatible; the swap is a host change.
- **Sub-10s retrieval mode.** Currently structurally unavailable. Would require a fundamentally different retrieval architecture (smaller embeddings, smaller chunks, naive-only mode) trading coverage for speed. Filed for if a real-time consumer shape demands it.
