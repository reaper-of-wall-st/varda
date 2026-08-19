# The Message Queue

One queue, inside the daemon, is the firehose of Varda's entire transport chain: everything is serialized in and out of it, and nothing travels any other way.

Agent communication is the part of the harness that most other systems sleep on. The mainstream production agent frameworks hand results off in-process: in Claude Code a subagent finishes and its result returns to the main conversation [^1], and LangGraph's deployable server queues runs behind an HTTP API, no broker named [^2]. That is fine while the process is alive. When the orchestrator dies, every in-flight message and every unfinished trace dies with it. Varda's answer is not a mesh of per-agent channels with a control plane bolted on. It is one queue, in-process (Eclipse Zenoh), transient; the `.varda/` database is the record. Nothing here is implemented yet — the crates are scaffolds — but this is an invariant the implementation must respect, not an implementation detail.

## One queue, inside the daemon

The queue is not a service; it runs inside the daemon process (`varda-daemon`). Agents do not talk to the daemon directly — they talk to the queue over the network. Each agent container gets its own IP on the Docker bridge — `172.17.0.2`, `172.17.0.3`, and so on, from the default `172.17.0.0/16` subnet [^3] — and dials the bridge gateway at the daemon's queue port. The gateway is the host's own address on the bridge (default `172.17.0.1`) and the container's default route, so a plain TCP connection to `gateway:queue-port` reaches the daemon on the host. That is stock Docker behavior, not a trick [^3].

The queue is per-world: one daemon per directory, one queue per daemon. The agents of one project share a single firehose; the agents of two projects do not. And the scope is one host, one world, no cluster: the queue has no replication, no partitioning, no external service to babysit — its whole operational surface is one process and one directory.

The network dial is not a convenience, it is the entire design. A container can reach exactly one thing on the host — the gateway — so the queue's listener is all an agent needs. No shared memory to mount, no socket files to plumb, no per-agent channel to provision. The client side of the transport is a dial, and the dial is the same for every agent in every world.

Serialization is doing real work here. One queue means one total order over the world's events: spawn before first message, tool call before tool result, drain before next turn. When something goes wrong, that order is the evidence. A mesh of channels gives you no such thing — you get a dozen partial orders and a guess.

Two caveats, both load-bearing. **Docker Desktop**: on Mac and Windows the Docker Engine runs inside a lightweight Linux VM [^4], so `172.17.0.1` is the VM's bridge, not your machine; the documented way for a container to reach the user's host there is the `host.docker.internal` alias [^5]. **Bind address**: the daemon must listen on the bridge (or `0.0.0.0`), not `127.0.0.1`. Loopback only exists inside the host's own network namespace, so a loopback-bound port is unreachable from a container — binding to loopback is the single most likely way this topology breaks in practice.

## What flows

Everything the system says to itself:

- **Spawn requests** — a parent admitting a child into the tree.
- **Parent-to-child messages** — the only agent-to-agent communication that exists, in both directions.
- **Full agent traces** — tool calls, tool results, the thinking between them. Not summaries; the raw record.
- **Compaction records** — the checkpoints that keep a long agent's history bounded.
- **Metrics.**

That list is exhaustive by design: all agent-to-agent and agent-to-daemon information rides the same stream. Payloads range from a few bytes of control to full traces with megabytes of tool output. The queue is one mechanism that has to be comfortable at both ends, not a fast lane and a bulk lane. If it happens, it flows. There is no side channel for lifecycle, no out-of-band control plane, no shared-memory shortcut. And the firehose is the persistence feed too: the same stream the turn loop consumes is the stream the persistence layer appends to disk ([persistence](persistence.md)).

## Who is a client

Three kinds of client, no more:

- **The TUI** (`varda-tui`) — a local client on the host. It renders everything because everything flows; there is no hidden state it cannot see.
- **Every worker node** (`varda-node`) — one per agent container. It dials in from the bridge, and that dial is the entire agent-side transport.
- **The daemon's own persistence layer** — the only writer to disk, and it writes what it reads off the firehose.

The queue is drained each turn: whatever accumulated since the last drain becomes the appended input of the next turn, and nothing is restructured — the loop itself is in [turn-loop](turn-loop.md). No message sits undrained between turns, and no state lives anywhere except the queue and the database.

## Durability is not a feature

An in-memory queue dies with its process. That is acceptable here, because durability lives in the database, not the broker: the queue is a transient transport, and the `.varda/` database (turso) is the record. Every message is appended to the database as it is consumed, and the database is reopened on restart: a restarted daemon recovers the record from the database, not from a message log. The database's sequence number carries the total order, so the evidence of the world's events survives a restart. Restart mid-fan-out — a parent has spawned twelve children, seven of them done — and the database has all nineteen messages; the daemon does not guess where the tree was, it reads it back.

Note the shape of that: the database is the source of truth — the single record of everything that happened — and the queue is a transient transport into it. There is one write path, through the queue, and everything downstream reads from the database. That is what makes the system debuggable: any behavior it has ever exhibited is reconstructable from the database.

The mechanism is locked (decision date 2026-08-17): **Eclipse Zenoh 1.10.0** [^10][^11] — an embeddable Rust router — runs inside the daemon process. No external service, no Go binary, no container: the broker is a dependency of `varda-daemon`, and the transport is `transport_tcp`. Two shapes were considered and rejected. An in-process durable append-only log — Varda's own append, fsync, replay code under `.varda/` — was rejected: the database already carries the record, and a second durable log would only duplicate it. NATS embed was rejected on a hard fact: NATS cannot be embedded in-process in Rust. There is no embeddable `nats-server` crate [^6]; the only Rust artifact with that name is an unpublished test helper [^7] whose source spawns the external Go binary [^8]. The closest off-the-shelf durable store — NATS JetStream's file store, where messages survive restarts and can be replayed [^9] — sits behind exactly that second process.

## The topology

```
   Docker bridge (default 172.17.0.0/16)
   +---------------------------------------------------------------+
   |  agent container                       agent container        |
   |  +------------------+                  +------------------+   |
   |  | 172.17.0.2       |                  | 172.17.0.3       |   |
   |  +------------------+                  +------------------+   |
   +           |                                     |             +
               | dials the bridge gateway:port       |
               | (the host's address on the bridge;  |
               | default 172.17.0.1)                 |
   +           v                                     v                +
   | host                                                             |
   |                                                                  |
   |  +-------------------------------------------------+  +--------+ |
   |  | daemon process                                  |  |        | |
   |  |  +-------------------------------------------+  |  | TUI    | |
   |  |  | THE QUEUE (in-process; the firehose)      |<-|--| (local)| |
   |  |  +-------------------------------------------+  |  +--------+ |
   |  |           | drained each turn       | appended  |             |
   |  |           v                         v           |             |
   |  |      turn loop           persistence -> .varda/ |             |
   |  +-------------------------------------------------+             |
   +------------------------------------------------------------------+
```

Every arrow in that picture is the same queue, and the picture is the policy: if a message needs to cross a boundary in it, it goes through the queue; if it cannot be drawn in it, it does not happen.

[^1]: https://code.claude.com/docs/en/sub-agents — accessed 2026-08-15
[^2]: https://docs.langchain.com/langsmith/agent-server-api/thread-runs/create-background-run.md — accessed 2026-08-15
[^3]: https://docs.docker.com/network/bridge/ — accessed 2026-08-15
[^4]: https://docs.docker.com/desktop/features/networking/ — accessed 2026-08-15
[^5]: https://docs.docker.com/compose/how-tos/networking/ — accessed 2026-08-15
[^6]: https://crates.io/crates/nats-server — accessed 2026-08-15
[^7]: https://github.com/nats-io/nats.rs/tree/main/nats-server — accessed 2026-08-15
[^8]: https://raw.githubusercontent.com/nats-io/nats.rs/main/nats-server/src/lib.rs — accessed 2026-08-15
[^9]: https://docs.nats.io/concepts/jetstream — accessed 2026-08-15
[^10]: https://zenoh.io/docs/ — accessed 2026-08-17
[^11]: https://github.com/eclipse-zenoh/zenoh — accessed 2026-08-17
