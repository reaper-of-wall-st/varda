# Subagents

The tree is Verda's coordination model, and it exists for humans and for
reinforcement learning.

Every agent in a Verda world is a node in one strict tree. This doc is that
tree: the topology rule, who owns a child's lifecycle, what the root looks
like, how fan-out works, where the human sits, and how a program drives the
whole thing.

## The tree, and only the tree

Strict topology: every agent has exactly one parent. A subagent's parent is
the agent that spawned it. A primary agent's parent is the human. There is no
other structure — no sibling links, no shared channels, no peer mesh.

Agent-to-agent communication happens only between parent and child. Messages
go up (child to parent: questions, reports, results) and down (parent to
child: tasks, answers, close). If agent A needs something from agent B and
they are not parent and child, A asks its parent; the parent routes it or
does it itself.

Lateral messaging is rejected, and the reason is engineering, not
aesthetics. In a tree, every message has an owner: the parent of two agents
is the one process that knows what both of them are doing, so it can
arbitrate, dedupe, and explain what happened afterwards. Give every pair of
N agents a direct channel and you get N*(N-1)/2 possible failure surfaces —
message storms, loops, duplicated work — and no single process that can be
asked why a message exists. A tree has N-1 edges and one owner per edge.
"Agent A messaged agent B directly" is a bug class this design refuses to
have; the coordination surface stays exactly as small as the work requires.

## Persistence and ownership

Agents are persistent by default. A subagent that has returned its result
and is waiting for a message is still running, exactly like a background
process you started. Persistence is the baseline state; stopping is a
decision.

Only the parent closes a child. The system (the daemon) never reaps an agent
on its own and never touches a child's container. The rigidity has a reason:
there is no predictable shutdown signal in the loop. A container runs a
model-driven loop, and the only entity that can say "this work is done" is
the one that owns the work. A daemon heuristic or a timer would be guessing
at a fact that only the parent actually has, so the close decision lives
where the information is: the parent sends the close, the child exits, the
daemon does the rest. An agent that is not closed simply keeps running; that
is the design, not a leak. (Design intent.)

## Genesis

Genesis is the first agent in a world, and it is a primary agent, not a
subagent. Primary agents sit directly under the human; Genesis is the root
you talk to first.

Genesis is strongly instructed to delegate. Given a big task, its default
move is to break the task up and hand pieces to children — spawn, await,
synthesize — not to grind through it itself. The root of the tree is a
delegator by job description; doing the work itself is a fallback for small
tasks. (Design intent.)

## Fan-out

Fan-out is first-class, not an escape hatch. An agent can issue parallel
tool calls and can spawn many children in one turn and await all of them.
Independent subtasks get parallel children; the parent collects the results
and moves on. The tree grows wide when the work is parallel, and each child
works in its own context — the same shape as every subagent system surveyed
[^1][^2][^3][^4][^5][^6][^7][^8][^9][^10] — so the parent's context stays small.

This is where Verda separates from the industry default. Subagents are table
stakes in 2026: 8 of the 10 major harnesses surveyed in mid-2026 ship
parent-child subagents (one of them still marked experimental); the only
exceptions are two minimalist single-agent tools [^1][^2][^3][^4][^5][^6][^7][^8][^9][^10]. The feature itself
is no longer a differentiator — and in the surveyed set, subagent
communication is uniformly one-shot: the child works in its own context,
returns a summary to the parent, and the relationship is over [^1][^2][^3][^4][^5][^6][^7][^8][^9][^10]. What
is unclaimed is the persistent, parent-owned tree: children that stay alive
between turns, share a world with their siblings, and are closed explicitly
by their parent. That is Verda's differentiator (design intent), and it is
the difference between a subagent feature and a coordination model.

## The human sits on top

The human sits on top of the tree — above the primary agents, the parent of
every primary. From the top you talk to Genesis like a colleague: a task
goes in, the tree does the work, a synthesis comes back. You can also spawn
additional primary agents into the same world; P2 and P3 are Genesis's
siblings, not its children — a second root for parallel work in the same
codebase. And from the TUI you can message any agent in the directory's
network: if S2 under P2 is stuck, you message it directly instead of
routing through Genesis. The human is the one node that can reach every
other node; no agent can. A new conversation started in the same directory
attaches to the same world — same database, same memories, same shared
workspace. The directory is the world; a conversation is just a door into
it.

## Interfaces: TUI, CLI, and the north star

The TUI (built on Ratatui) is the human's interface: watch the tree grow,
message any agent, spawn primary agents. It packages a no-TUI CLI mode for
programmatic use — same capabilities, no terminal UI. And everything the
human can do, a script can do: the TUI is a client of the system, never a
gatekeeper.

That last line is the north star, and it is why the tree exists for humans
and RL. An environment that trains subagent-based behavior can drive the
harness end to end — spawn a world, feed tasks, observe traces, close
agents — with no human UI in the path. That is a north star, not a v1
deliverable: Verda designs for it (everything scriptable, nothing that
requires a terminal), but there is no RL API to design yet.

## The tree, drawn

```text
                                  the human
                                   (top of tree)
                    |                       |
                  spawn                   spawn
                    |                       |
                    v                       v
              +------------+          +------------+
              |  Genesis   |          |  primary   |
              |   (P1)     |          |   (P2)     |
              +------------+          +------------+
                  |    ^                  |    ^
                  |    |                  |    |
                  v    |                  v    |
              +------------+          +------------+
              |  S1, S2    |          |  S3        |
              |  subagents |          |  subagent  |
              |  of P1     |          |  of P2     |
              +------------+          +------------+
  spawn:    one-way, down — the human spawns primaries; a parent spawns
            subagents
  messages: parent <-> child, both directions — the only agent-to-agent
            path
  close:    one-way, down — only the parent of a child closes it; the human
            closes primaries; the daemon closes nothing
```

[^1]: https://docs.anthropic.com/en/docs/claude-code/sub-agents — accessed 2026-08-15 — "Create custom subagents (Claude Code docs)"
[^2]: https://learn.chatgpt.com/docs/agent-configuration/subagents.md — accessed 2026-08-15 — "Subagents (Codex/ChatGPT docs)"
[^3]: https://github.com/aaif-goose/goose/blob/main/documentation/docs/tutorials/subagents.md — accessed 2026-08-15 — "Using Subagents (Goose tutorial)"
[^4]: https://github.com/OpenHands/software-agent-sdk/blob/main/openhands-tools/openhands/tools/delegate/definition.py — accessed 2026-08-15 — "Delegate tool (OpenHands source)"
[^5]: https://ampcode.com/manual — accessed 2026-08-15 — "Owner's Manual (Amp)"
[^6]: https://docs.cline.bot/features/subagents — accessed 2026-08-15 — "Subagents (Cline docs)"
[^7]: https://github.com/continuedev/continue/blob/main/extensions/cli/src/tools/subagent.ts — accessed 2026-08-15 — "subagent tool (Continue CLI source)"
[^8]: https://github.com/google-gemini/gemini-cli/blob/main/docs/core/subagents.md — accessed 2026-08-15 — "Subagents (Gemini CLI docs)"
[^9]: https://pi.dev/docs/latest — accessed 2026-08-15 — "Pi Documentation (latest; no subagent feature)"
[^10]: https://github.com/Aider-AI/aider — accessed 2026-08-15 — "Aider-AI/aider (single-agent; no subagents)"
