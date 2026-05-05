# Corpus

> What the index sees. The choice of *what to ingest, in what shape, with what filtering* is load-bearing for retrieval quality at scale. Most pilot-to-production failures in RAG systems are corpus failures wearing architecture costumes.

## What goes in the indexed corpus

Each source is included for a specific reason. *Why* multi-source matters at all is covered in [why.md](why.md#why-multiple-sources); this section is what's actually ingested.

### Internal tickets

The dominant source. Internal trackers hold the *narrative* of how decisions got made — discussion threads, attachment context, cross-references between tickets, the comment history that explains why the obvious solution wasn't chosen. Filtered through the gatekeeper (below); the noise floor is high.

### External tickets

Customer-facing tickets and external-collaboration trackers. Same structural shape as internal trackers but a different audience and tone. Worth ingesting separately so a consumer can distinguish "what we said internally" from "what we said to the customer."

### Code

Indexed structurally as an AST graph, not as text. Lives in a separate index from the text store. See [`architecture.md`](architecture.md#index) for why the two indices stay specialised.

### External specifications

Standards documents, test plans, certification specs the project has to conform to. These are *immutable inputs the project negotiates against*, distinct from internal design intent. Voluminous; selective ingestion via the document-parser path. Not all clauses matter — focus on the sections the project actually touches.

### Build-system configuration history

Build/CI job configurations and their per-change history are noisy raw text but high-value once distilled. The ingest path summarises rather than ingests verbatim: an LLM-driven summariser produces a **change-narrative** entry per significant config-change cluster (author, date, *what changed and why* inferred from description fields and adjacent commits). The narratives get indexed; the raw config doesn't.

This is direct prior art from manual investigation work — engineers already produce hand-written summaries of "how did this build job evolve" when investigating regressions. The system industrialises that output.

### Curation overlay

Net-new annotations contributed by humans, indexed alongside corpus content. The contribution mechanics are designed in [`consumers.md`](consumers.md). Important here: the curation overlay is *part of the corpus*, not a sidecar — a query that hits a curation entry and a ticket together is the whole point.

## What stays out (query-time only)

Sources too voluminous, too noisy, or too currency-sensitive to pre-process. Consumers reach these via tools, scoped by the question:

- **Full version-control history** — fetched on demand, scoped by ticket key, revision range, file path, or date window.
- **Build logs** — fetched on demand for runtime evidence about a specific build or failure.
- **Live ticket state** — fetched on demand if the indexed snapshot is stale or insufficient.
- **Large attachments** — fetched on demand if they prove relevant. Pre-ingesting all attachments would dominate index volume.

## The gatekeeper

The gatekeeper is a filter that decides whether a ticket (or other ingest unit) enters the corpus. It is load-bearing for retrieval quality at scale, not just hygiene.

The reason is structural. Hybrid graph traversal degrades past roughly a thousand nodes. LLM-driven entity extraction over typical engineering tickets produces on the order of 100–150 nodes per ticket; a corpus of even a few hundred tickets, ingested without filtering, lands one to two orders of magnitude past the degradation threshold. The dominant ticket shapes in any corpus — routine assertion-add tickets, dependency bumps, lint fixes — overwhelm the graph with low-rationale nodes that share names ("update", "fix", "test") and dilute every retrieval.

### Inclusion criteria

A ticket enters the corpus if **any** of these hold:

- Has discussion (>2 substantive comments, not auto-generated bot comments).
- Carries cross-references to other tickets, commits, or external documents.
- Has non-log attachments (design docs, screenshots of bugs, decision artefacts).
- Is explicitly tagged as a decision/design/RCA ticket (project-specific labels).
- Resolves a non-trivial problem (status went through investigation, not just opened-and-closed).

### Exclusion criteria

A ticket is **excluded** if **all** of these hold:

- Single-author, no discussion.
- No attachments, or only auto-generated logs.
- Title and body match a routine pattern for the project (assertion-add, version bump, typo fix).
- Closed within hours of opening.

The gatekeeper is *project-tunable*: routine patterns vary. One project's routine is "add assertion for a new test ID"; another's is "bump dependency version." Per-project tuning lives in `projects/<name>/gatekeeper.md`.

The cost of being too strict is missing some context; the cost of being too loose is graph degradation that hurts every query. Err strict; the validation pass below catches over-filtering early.

## Format requirements

Indexed text needs to be RAG-friendly: narrative-shaped, free of UI cruft, with markup the embedder can handle.

### What needs stripping

- Source-specific markup (ticket-tracker wiki markup, custom code blocks) → markdown equivalents.
- Change-history dumps ("Field X changed from Y to Z" rows) → drop unless the change carries rationale, in which case collapse into a one-line narrative entry.
- Auto-bot comments (build notifications, code-review-bot summaries) → drop.
- UI noise (linked-issue rendering, watcher lists, display fields like "Days since last update") → drop.

### Adapting per-source ingest tool output

Per-source ingest tools usually emit human-readable formats — fine for manual reading, not RAG-shaped. Adaptation has two paths:

- **Fork** the tool, accept divergence, pay the maintenance cost.
- **Post-process** native output with a transform step in the ingest pipeline.

Lean toward post-process unless format divergence becomes large. The transform is a single LLM-or-regex pass; a fork couples two release cycles.

## Validation pass

Before committing to a full-corpus ingest, run a staged validation. Mechanism-level validation — proving the extraction pipeline produces sensible nodes and edges on a small sample — does not prove that the corpus shape works at production scale. The staged pass tests scale separately.

### Staging

| Stage | Tickets | Question being answered |
|-------|---------|-------------------------|
| 1 | ~30 mixed | Does extraction quality hold across heterogeneous ticket shapes (short, long, attachment-heavy, comment-heavy, cross-project)? |
| 2 | ~50 | Graph hygiene — node count growth rate, edge density, entity-deduplication quality (string-only dedup will miss "Jane Smith" vs "jane.smith"). |
| 3 | ~200 | Retrieval quality at intermediate scale. Run a fixed sample of representative queries; compare hybrid vs naive retrieval coverage. |
| 4 | ~500 | Onset of retrieval degradation. Same query sample as stage 3; measure recall and precision deltas. |

### Decision points

- After **stage 1**: tune extraction prompts and gatekeeper criteria. If extraction quality varies wildly by ticket shape, the gatekeeper may need shape-aware rules.
- After **stage 2**: assess whether semantic entity merging needs a manual step (the curation overlay can absorb merge directives).
- After **stage 3**: baseline retrieval quality before scale stress.
- After **stage 4**: the partition decision. If retrieval has degraded meaningfully versus stage 3, **stop**. Don't ingest the remaining tickets into one graph — partition first (per-project or per-component sub-indices) and re-ingest from a clean store. If stage 4 is still healthy, continue to full corpus.

This is the loop. The cost of stopping at 500 and partitioning is hours of re-ingest; the cost of discovering degradation in production is rebuilding from scratch with a worse mental model of what failed.

## Re-ingestion cadence

Tickets mutate. New comments arrive. Decisions get reversed in late comments that don't update the original body. A write-once corpus drifts.

The text index supports incremental updates. The pragmatic policy:

- **Weekly batch incremental** — re-ingest tickets touched since the last run. The per-source ingest tool's ledger tracks per-ticket last-indexed state.
- **Triggered full re-ingest** — when the extraction prompt changes meaningfully, when gatekeeper criteria change, or when entity-merge directives accumulate enough to warrant a clean rebuild. Expect this a handful of times per year.
- **Out-of-band triggered re-ingest** — a curator can flag a specific ticket for re-ingest when they know the indexed version is stale (e.g., a major decision was just added in a comment).

Full corpus re-ingest is not free — local-extraction rates make a full rebuild an hours-to-days operation for a corpus of more than a few hundred tickets. Treat full re-ingest as a planned, scheduled event, not a casual operation.

## Rejected

### Rejected: ingest everything, gatekeep nothing

Pulling all tickets straight into the index, trusting retrieval to handle the noise. Graph traversal degrades past ~1k nodes; even a modest project corpus produces one to two orders of magnitude more than that. The gatekeeper isn't optional cleanup; it's the difference between a usable graph and an unusable one.

### Rejected: one unified graph regardless of scale

Assuming a single text index works for the whole corpus. Validation stage 4 is the test of this assumption. If it fails, partition. Refusing to partition would force a choice between dropped coverage and degraded retrieval — neither is acceptable.

### Rejected: pre-ingesting full version-control history, all build logs, all attachments

Long-tail content. Pre-processing buries the narrative substrate. Belongs to query-time tools — see [`architecture.md`](architecture.md#query-time-retrieval-tools).

### Rejected: forking ingest tools upfront

Speculative coupling cost. The post-process transform path is reversible; a fork is not.

## Deferred decisions

- **Partition shape**, if validation stage 4 forces partitioning: per-tracker (one sub-index per ticket-tracker project) versus per-component (build / streaming / alerting / etc.). Decided after stage-4 data.
- **Build-config-history summarisation timing**: at ingest-time per change-cluster, or as a one-off corpus-prep batch. Decided after the first ingestion attempt.
- **Semantic entity-deduplication strategy**: lean on curation-overlay merge directives, or run a periodic LLM-based dedup pass over the graph. Decided after stage 2.
