# Varda

Varda — the world's best LLM harness.

## What Varda is

Varda is a Rust-based, multi-agent, containerized LLM agent harness. The
design in one paragraph: one process, the daemon, owns a whole world of
agents in a project directory. Every agent runs in its own fully furnished
container; every message between agents, or between an agent and the
daemon, rides one message queue; and everything that happens is written to
a local database in `.varda/`, so a crashed world reloads instead of
starting over.

Humans drive the world through a TUI and a CLI. The first agent in every
world is **Genesis**, a primary agent strongly instructed to delegate
bigger tasks rather than do the work itself. The north star is not human
at all: reinforcement learning environments that train subagent-based
behavior use the same scriptable surface, with the TUI out of the loop.

Everything in this README is design intent. The four crates
(`varda-core`, `varda-daemon`, `varda-node`, `varda-tui`) are cargo
scaffolds; this document and `docs/philosophy/` are the spec they get
built to.

## What is broken

Five things, each checked against the ten major harnesses surveyed for
this repo (Claude Code, Codex CLI, pi, Goose, Aider, OpenHands, Amp,
Cline, Continue, Gemini CLI), all facts as of 2026-08-15.

**Minimalism as a feature ceiling.** pi is the clearest example:
no subagents, no built-in sandboxing, no cross-session memory [^1][^2][^3].
Pi is not featureless — it ships per-directory sessions with branching,
auto-compaction, and an extension system [^2]. Minimal as an aesthetic is
fine; minimal as a ceiling on what a harness can do is not. The
project's answer, in one line: "minimal is not the goal — problem solving is."

**Agent communication is months old, at best:** the peer-messaging
capabilities shipped by major harnesses are months-old at best and still
marked experimental [^4][^5]. The two data points come from one product:
agent teams, published 2026-06-15 and still flagged experimental [^4][^5],
and cross-session messaging, v2.1.224, published 2026-08-07 [^5].

**Memory is flat files.** Industry practice is flat markdown loaded
wholesale (CLAUDE.md/AGENTS.md-style, hard byte caps); no surveyed harness
has DB-backed indexed retrieval [^6][^7]. Claude Code loads CLAUDE.md plus
a model-written note capped at 200 lines / 25KB into every session [^6];
Codex concatenates an AGENTS.md chain once per run [^7]. A handful of
files, re-read in full every session, and the rest of the project
forgotten.

**The Node/TypeScript center of gravity (6 of 10 surveyed harnesses).**
Six of the ten surveyed harnesses run on Node/TypeScript at the core:
Claude Code, pi, Gemini CLI, Cline, Continue, Amp [^1][^5][^14][^15][^16][^17]
[^18]. This is not a monoculture; the counterweights are real:
Rust (Codex CLI at ~96% Rust, Goose at ~70%) and Python (Aider, OpenHands)
[^10][^11][^12][^13]. The center of gravity matters for one concrete reason:
the Node runtime is where the RAM problem lives.

**Gigabytes per agent.** Community-reported figures from Claude Code's
own issue tracker — user reports, not vendor benchmarks: a 1.5-2 GB
active-memory baseline per session [^8], and a 13 GB peak on an 8 GB
machine [^9]. Fan a tree of agents out over a Node-based harness and every
agent carries that baseline with it.

## How it fits together

The system as designed, before any of it is built (design intent). The
networking labels match Docker's documented bridge model [^19] and the
lifecycle verbs are the Docker Engine API's own [^20]:

```text
                    +------------------------------------+
                    | TUI / CLI                          |
                    | the human's interface, or a script |
                    +------------------------------------+
                                       |
                                       |  every message in and out rides the queue
                                       v
+----------------------------------------------------------------------------+
| daemon (one per project directory)                                         |
|                                                                            |
|   +---------------+   +--------------------+   +----------------+          |
|   | message queue |   | Docker lifecycle   |   | .varda/        |          |
|   | the firehose  |   | create -> start -> |   | database, log, |          |
|   |               |   | inspect -> stop -> |   | lock files     |          |
|   +---------------+   | remove             |   +----------------+          |
|                       +--------------------+                               |
+--------------------------------------+-------------------------------------+
                                       |
                                       |  containers reach the daemon's queue
                                       |  through the gateway IP: the host's
                                       |  address on the Docker bridge
                                       |  (default 172.17.0.1), at the queue port
                                       v
  +---------------------------------+   +---------------------------------+
  | container: agent 1              |   | container: agent 2              |
  | fully furnished dev environment |   | fully furnished dev environment |
  +---------------------------------+   +---------------------------------+
```

Everything crosses one channel: the queue. Agents talk to the daemon and
to each other by no other route. A container that dies is relaunchable
because the state it needs is not in the container.

## Why Rust

A harness is a long-lived process that owns a tree of agents, a queue,
and a database, and stays up while all of them run. The
community-reported baseline for a single Node-based session is 1.5-2 GB
[^8]; a world of agents multiplies it. Rust is the counterweight that
already proved the point: Codex CLI (~96% Rust) and Goose (~70% Rust) are
marketed on exactly this lightness [^10][^11].

The second reason is the bar the code has to meet: ponytail-clean,
readable code, no over-engineering, and the maintainer reads every PR.
That is design intent and a process commitment, and it is the reason this
codebase can stay small enough for one person to review in full.

## Model-agnostic by design

Varda does not assume one model provider. The model layer is a
replaceable seam in the turn loop, and the architecture's job is to keep
the prompt prefix byte-stable so that whatever provider sits behind the
seam caches well. No provider is named in these docs, and no model crate
will be: the harness is the product, not the model contract.

## The philosophy, in five documents

- [Subagents](docs/philosophy/subagents.md): the strict tree, Genesis, the human's place.
- [Sandbox](docs/philosophy/sandbox.md): one fully furnished container per agent.
- [Message queue](docs/philosophy/message-queue.md): one durable firehose.
- [Persistence](docs/philosophy/persistence.md): persist everything every turn; reload it.
- [Turn loop](docs/philosophy/turn-loop.md): the byte-stable prefix and compaction.

## Status

Varda is greenfield and docs-first. Nothing runs yet, nothing is
installable, and nothing in these docs describes shipped capability: the
architecture above is design intent, and the four crates are scaffolds.

Repository: https://github.com/reaper-of-wall-st/varda

[^1]: https://github.com/earendil-works/pi — accessed 2026-08-15
[^2]: https://pi.dev/docs/latest — accessed 2026-08-15
[^3]: https://pi.dev/docs/latest/sessions — accessed 2026-08-15
[^4]: https://docs.anthropic.com/en/docs/claude-code/agent-teams — accessed 2026-08-15
[^5]: https://registry.npmjs.org/@anthropic-ai/claude-code — accessed 2026-08-15
[^6]: https://docs.anthropic.com/en/docs/claude-code/memory — accessed 2026-08-15
[^7]: https://developers.openai.com/codex/guides/agents-md.md — accessed 2026-08-15
[^8]: https://github.com/anthropics/claude-code/issues/9604 — accessed 2026-08-15
[^9]: https://github.com/anthropics/claude-code/issues/22183 — accessed 2026-08-15
[^10]: https://github.com/openai/codex — accessed 2026-08-15
[^11]: https://github.com/aaif-goose/goose — accessed 2026-08-15
[^12]: https://github.com/Aider-AI/aider — accessed 2026-08-15
[^13]: https://github.com/OpenHands/software-agent-sdk — accessed 2026-08-15
[^14]: https://github.com/google-gemini/gemini-cli — accessed 2026-08-15
[^15]: https://github.com/cline/cline — accessed 2026-08-15
[^16]: https://github.com/continuedev/continue — accessed 2026-08-15
[^17]: https://ampcode.com/manual — accessed 2026-08-15
[^18]: https://registry.npmjs.org/@ampcode/plugin — accessed 2026-08-15
[^19]: https://docs.docker.com/network/bridge/ — accessed 2026-08-15
[^20]: https://docs.docker.com/reference/api/engine/version/v1.55.yaml — accessed 2026-08-15
