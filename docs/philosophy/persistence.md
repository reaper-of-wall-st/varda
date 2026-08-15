# Persistence

Everything is persisted, every turn, for every agent — and everything persisted is reloadable.

The four `varda-*` crates are scaffolds. Nothing in this document is a capability you
can run; it is the design the system will be built to. Statements that are decisions
rather than industry practice are labeled design intent.

## Why everything is a record

Agent state, left alone, is ephemeral: the container dies, the history dies.
Memory, where it exists at all, is a flat markdown file loaded wholesale: CLAUDE.md is
read at the start of every session [^1], and AGENTS.md-style instruction files are
loaded the same way [^2]. There is nothing to query, nothing to score, and nothing that
stays fresh once the file stops being true.

Verda inverts that. The log of what happened is the source of truth: every turn, for
every agent, the full record lands in a database that outlives the daemon that wrote it.
A crash is then an interruption, not a loss, because the state was on disk the moment it
happened.

## What is persisted per turn

For every turn of every agent, the record captures:

- the tool calls the agent made,
- the results those calls returned,
- the agent's thinking,
- the full trace of the turn, and
- any compaction record that closed it.

No sampling, no "keep only the last N". A database row is not where you economize. A
compaction record is a checkpoint appended to the log when a turn's context is
compacted; the turn loop owns its shape ([turn-loop](turn-loop.md)).

Ownership follows one rule: the daemon owns persistence (design intent). Agents never
write their own state. A turn's record reaches the daemon over the message queue — the
same firehose everything else in Verda travels on — and the daemon's persistence layer
is a client of that queue, appending each turn as it arrives. One writer, one file, one
owner, and no agent-local state files to orphan.

## Where it lives: one database file in `.varda/`

Storage is an embedded **libSQL (Turso's fork of SQLite)** database file in `.varda/` in
the project directory (design intent). The same directory holds the daemon's log and
lock files; the database is the third resident. No external database process, no second
storage engine — one file per project directory, and the file is the archive.

The engine is not an afterthought for the memory section below. The local libSQL build
ships both index primitives that the design needs: FTS5 full-text search is compiled into
the libSQL Rust build [^3], and libSQL's C core carries native vector column types,
usable from a local file with no extension [^4][^5]. The keyword arm and the vector arm of
a hybrid index fit inside the same file that already stores the entries. That is what
"one file" buys you.

```text
 agent (container)

  one turn: model call -> tool calls -> tool results -> thinking
     |
     |  firehose: every turn, every agent
     v
+---------------------------------------------------------------+
| daemon (one per directory)                                    |
|                                                               |
|  queue: everything in, everything out                         |
|      |                                                        |
|      v  (drained every turn)                                  |
|  persistence layer: appends the record                        |
+---------------------------------+------------------------------+
                                 |
                                 v
+---------------------------------------------------------------+
| .varda/ in the project directory                              |
|                                                               |
|  libSQL database file:                                        |
|   full traces | compaction records | memory entries           |
|   (labels + bodies)                                           |
+---------------------------------------------------------------+

 crash: the file outlives the daemon. Reopen it, rebuild the
 container picture, reload the traces. Nothing to reconstruct
 from memory.
```

## Reload: crash is routine

Because every turn is on disk, a dead daemon is a routine event with a mechanical recovery:

1. Open the database in `.varda/`.
2. Rebuild the container picture — which agents exist and what their state was —
   from what the database holds.
3. Reload the traces for each agent that comes back.

The database is the bootstrap (design intent). And because the record is complete,
replay is possible: any agent's full trace can be re-read from the file, turn by turn
(design intent).

## Revival is TTL-bounded

"Everything is reloadable" does not mean "everything is revived". On reload, an agent
comes back only if it was active inside the TTL window: its last turn or message falls
within the window, which defaults to 24 hours and is configurable (design intent).
"Active" means exactly that — last turn or message within the window — and nothing else.

The window exists because idle and abandoned are the same state on disk. Reviving every
agent a directory ever spawned would turn each restart into a resuscitation sweep. A
24-hour cutoff keeps restarts honest: recent work continues where it stopped, and
abandoned work stays on disk, complete and reloadable, if anyone ever wants it.

Two rules round out the lifecycle (design intent). While the daemon is live, only an
agent's parent can close it — the system never reaps on its own, because no predictable
shutdown signal exists to hand a child. And a recently-active agent whose container
failed is relaunched by the daemon from its persisted state; the container is cattle,
the state is sacred (mount mechanics live in [sandbox](sandbox.md)).

## Memory: the record that serves itself

Memory is where persistence stops being an archive and becomes a service. Three parts:
entries, an index over their labels, and a per-turn loop that queries it and injects
what it finds.

### Entries

An agent writes a memory entry with an explicit tool call — no daemon-side auto-
extraction (design intent). Explicit writes are the precision lever: auto-extractors
produce redundant, low-salience entries, and the memory literature is blunt: over-
extraction reduces precision [^6]. The closest production analog is Letta's
archival memory, which the agent writes through an insert tool and searches on
demand [^7].

An entry is a category, a set of labels, and a body. The labels carry the searchable
meaning; the body is stored as-is and fetched only when the entry is retrieved. Indexing
the labels rather than the bodies keeps per-turn queries cheap and keeps bodies out of
the index — the research recommendation for this shape, and a natural fit for the design
(design intent).

### The hybrid label index

Labels are hybrid-indexed (design intent): a keyword arm (FTS5) and a vector arm (dense
embeddings of the label text), both in the same database file. A query runs against
both, and the two ranked lists are fused by Reciprocal Rank Fusion.

RRF is the default that needs no tuning. It fuses ranked lists without score
normalization and without training data, and the paper that introduced it fixed its
single constant (k = 60) in a pilot run, reporting the choice "was not critical" [^8].
Hybrid search itself — a BM25 keyword signal fused with a dense-vector signal — is a
first-class mode across the major vector engines [^9][^10].

Where the evidence turns, so does the doc. Fusion effectiveness depends on the dataset:
Vespa's own hybrid tutorial concludes hybrid effectiveness "depends on the dataset and
the retrieval strategies" and says to evaluate on your own data [^10], and vendor
guidance is explicit that hand-tuned weights without measurement "are unlikely to beat
the default reliably" [^9]. So RRF with its standard constant ships as the default, and
the fusion weights and thresholds are an open tuning question — a research
recommendation to keep defaults until an eval set exists, not a design constant.

### The per-turn loop

Each turn, the index is queried against that turn's input (design intent; the same
query, retrieve, construct loop that production memory systems run [^11]). The fused list
is cut down: a small top-k, anything below a similarity threshold is dropped, and a
rerank step is the optional precision upgrade [^12][^13]. The surviving entries' bodies
are fetched from the database and appended at the END of the turn's input, under a hard
token budget (design intent).

```text
 turn input
     |
     v
 1. query both label indexes: FTS5 (keyword) + vector  <----+
                                                       +----+------------------+
                                                       | libSQL file, .varda/  |
                                                       |  label indexes:       |
                                                       |   FTS5 (keyword)      |
                                                       |   vector (dense)      |
                                                       |  entry bodies         |
                                                       +----+------------------+

     |                                                      |
     v
 2. fuse the ranked lists (RRF, no tuning)
     |                                                      |
     v
 3. top-k, drop below the similarity threshold,
     optional rerank
     |                                                      |
     v
 4. append the surviving bodies at END of turn input    <---+
     under a hard token budget
```

Two reasons the bodies go at the end. LLMs use information at the beginning and the end
of long contexts far better than the middle — the "lost in the middle" effect [^14]. And
the turn loop's byte-stable prefix law: appending is the only injection point that does
not rewrite history and bust the prompt cache ([turn-loop](turn-loop.md)).

The hard budget is the design, not a failure mode. The industry pattern for injected
memory is a hard cap — Claude Code loads the first 200 lines or 25 KB of its auto memory
per session, full stop [^1] — and bounded retrieval beats full context on tokens and
latency [^15]. Memory that does not fit the budget does not go in; it stays in the file,
retrievable next turn when the input warrants it.

### Staleness

Memories rot; the architecture has to plan for it. The production answer is to
invalidate, not delete: Zep's temporal knowledge graph marks contradicted facts invalid
with timestamps instead of removing them [^11], and Mem0's update phase reconciles new
entries against similar existing ones at write time with ADD/UPDATE/DELETE/NOOP
operations [^15]. A row of category, labels, and body is the unit those operations act on
— update the labels or invalidate the entry, and the index follows, with no rewrite of
history.

[^1]: https://code.claude.com/docs/en/memory — accessed 2026-08-15
[^2]: https://agents.md/ — accessed 2026-08-15
[^3]: https://github.com/tursodatabase/libsql/blob/main/libsql-ffi/build.rs — accessed 2026-08-15
[^4]: https://github.com/tursodatabase/libsql/tree/main/libsql-sqlite3/src — accessed 2026-08-15
[^5]: https://turso.tech/vector — accessed 2026-08-15
[^6]: https://langchain-ai.github.io/langmem/concepts/conceptual_guide/ — accessed 2026-08-15
[^7]: https://docs.letta.com/v1-sdk/memory/archival-memory — accessed 2026-08-15
[^8]: https://dl.acm.org/doi/10.1145/1571941.1572114 — accessed 2026-08-15
[^9]: https://qdrant.tech/documentation/concepts/hybrid-queries/ — accessed 2026-08-15
[^10]: https://docs.vespa.ai/en/learn/tutorials/hybrid-search.html — accessed 2026-08-15
[^11]: https://arxiv.org/abs/2501.13956 — accessed 2026-08-15
[^12]: https://github.com/mem0ai/mem0/blob/main/docs/core-concepts/memory-operations/search.mdx — accessed 2026-08-15
[^13]: https://qdrant.tech/documentation/search-precision/reranking-semantic-search/index.md — accessed 2026-08-15
[^14]: https://arxiv.org/abs/2307.03172 — accessed 2026-08-15
[^15]: https://arxiv.org/abs/2504.19413 — accessed 2026-08-15
