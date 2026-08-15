# The Turn Loop

Verda's turn loop is built around one law: the history prefix is byte-stable, new
input is appended, and nothing is restructured — because prompt-cache hits are a
design goal, not an optimization.

## The Caching Law

Every major provider caches the exact prefix of the prompt, from byte one. Change
anything before a cache point and everything after it is recomputed; there is no
per-file or per-segment caching [1]. The mechanism differs by provider: on
Anthropic-style providers the harness places explicit cache-control breakpoints,
while OpenAI and Gemini cache server-side, automatically, with nothing for the
client to write [2][3]. The economics make the law worth
building around: a cache read costs about 0.1x the base input price, and a fresh
cache write costs 1.25x, as documented by both Anthropic and OpenAI [2][4].

So the loop is shaped by the prefix. The stable stuff — system layer, tool
definitions, project context — sits at the front and is never touched after;
conversation history is immutable, turns only ever append. New input — a human
message, a queued event, a tool result, retrieved memory — lands at the tail. The
model layer Verda builds on ships exactly this pattern as a first-class
configuration: an hour TTL on the static prefix, five minutes on the moving tail,
and a cache breakpoint that advances forward as the conversation grows [5].

One correction to a common assumption: the loop itself is explicitly budgeted. The
model layer's default loop budget is a single model call — a turn that calls a tool
needs a budget of its own, and Verda sets it deliberately instead of inheriting the
default [6]. A harness that does not think about its loop budget finds out on the
first tool round-trip.

## The Queue Drains Every Turn

Design intent: the daemon's message queue is fully drained every turn, and what is
in it — events, completions, a child agent's reply, a finished command — becomes
the appended input of the next turn. Nothing in the prefix is ever rewritten to
make room; the queue's contents arrive as new bytes at the tail.

This is what makes the law work under pressure. A reactive harness is tempted to
rewrite context to stay current. Verda's answer: make the world come to the loop,
not the loop to the world — the queue buffers, the drain serializes, and the
append keeps the prefix intact. The prefix stays byte-identical across turns, so
the cache hits every turn; and ordering is total and auditable — every queued
message has a position in the log, so replay is well-defined.

Placement matters too. Appended input is also the right place for anything injected
— retrieved memory, event summaries — because models read the ends of long contexts
more reliably than the middle (the U-shape from "Lost in the Middle" holds even for
explicitly long-context models [7]), and end-of-turn is the only injection point
that does not touch the prefix.

## Long-Running Commands

Design intent: when a tool call starts a command, the turn waits for the response
body until the timeout — around 60 seconds. If no body arrives by then, the command
does not fail; it keeps running. The agent holds a queryable handle, and when the
command finishes, the completion queues a turn. There is no timeout-failure mode:
a timeout is a handoff, not an error.

That decision buys the full async surface:

- Parallel tool calls in one turn — the model requests several, all are executed,
  and all results come back together.
- Fan-out — a parent launches many long-running children or commands in one turn
  and keeps working while they run.
- Await-all — a turn can wait for the whole set to finish, or completions can queue
  turns as they land.

The loop never blocks on the slowest command. It keeps cycling — drain, model,
tools — while slow work re-enters through the queue.

## What the Industry Does About Compaction

**The universal pattern: rewrite the prefix, pay one cache miss.** Every major
harness compacts by replacing the rendered history with a summary, then continuing
to append from the new, shorter prefix — and every major harness documents that
this costs exactly one cache miss. Claude Code says it plainly: compaction
"replaces your message history with a summary. By design, this invalidates the
conversation layer, since the next request has a new, shorter history that doesn't
share a prefix with the old one" [8]. OpenHands' own docs are equally frank:
"condensation destroys the prompt cache, but doing so regularly keeps the cost of
rebuilding the prompt cache low" [9] — the industry does not avoid the miss; it
budgets it.

**The triggers in the wild fall into four families.**

- Auto threshold: Gemini CLI compresses when history exceeds 50% of the model's
  token limit, keeping the most recent 30% [10]. Codex has a token limit for
  auto-compaction, with a hard cap at the model's full context window [11][12].
  Claude Code compacts by default as you approach the model's limit, with a
  user-configurable window [8][15].
- Manual: a /compact command in both Claude Code and Codex [8][12].
- Agent-initiated: Codex exposes a new_context tool the model can call — a
  rollover to a new context window that "does not clear, reset, or otherwise
  affect environment state" [13].
- Hard reset with anti-thrash: OpenHands distinguishes a soft trigger (skip and
  retry) from a hard reset (summarize and restart) when the window overflows [9].
  Claude Code stops auto-compacting and shows an error instead of looping when one
  large output refills the context immediately after each summary [14].

**Why not just run to the limit?** Context rot. "Lost in the Middle" shows
performance is highest when relevant information sits at the beginning or end of
the context and degrades significantly in the middle — a U-shape [7]. Chroma's
18-model "Context Rot" report finds the degradation is monotonic: performance
consistently drops as input length grows [16]. The operational reading: compact on
a growth budget, not at the limit, and keep the surviving context high-signal.
Recent turns sit at the safe end of the U, so the tail is the last thing you
compress.

**The stable-map alternative: Aider.** Aider takes the opposite tack — and it has
no /compact at all. Instead of summarizing, it keeps the prompt cacheable by
keeping it small and ordered: a repo map (a graph-ranked map of the files and
symbols the task touches, held within a token budget) plus /clear and /drop for
manual resets, and a cache-ordered prefix where the stable content — system prompt,
read-only files, the map — comes first and the editable files come last [17][18][19].
The lesson for Verda: preventing growth beats patching it — offload bulky,
regenerable content out of the history and keep a stable map in place of raw reads.

## Verda's Compaction

Design intent, with cited precedents: Verda does not rewrite history. Compaction is
an appended checkpoint record on the append-only log — the raw log, everything
persisted every turn, is never rewritten. The rendered prefix is a pure function of
three parts:

```text
  [ stable system layer ] + [ latest compaction record ] + [ high-signal suffix ]
```

- Stable system layer: the byte-stable front (system, tools, project context). It
  never counts against the budget.
- Latest compaction record: the most recent checkpoint — a handoff-quality summary
  of everything before it, with window metadata.
- High-signal suffix: the raw turns since the record. This is the part that grows.

When the suffix crosses the budget, compaction runs in two stages:

1. Stage 1 — a mechanical prune of stale tool results. In the rendered prefix, old
   tool results are replaced with short pointers; the data itself stays in the
   database and is reloadable through the handle. No model call. Tool output is the
   bulk of agent context, so this alone recovers most of the space. Claude Code's
   documented pass does the same: it "clears older tool outputs first, then
   summarizes the conversation if needed" [14].
2. Stage 2 — a compaction record, only if still over budget. A summarization pass —
   run against the warm cache, so it costs a fraction of the context size [1] —
   appends one compaction record. The next rendered prefix is the stable layer, the
   record, and a short suffix.

The trigger is a prefix-scoped growth budget: tokens added since the last
compaction record, not the total context — the stable layer and the record do not
count. The model's window is the hard cap, and an anti-thrash stop means that if
compaction is not making room (one giant output refills the window immediately),
the loop stops and reports an error instead of looping [14].

The one deliberate cost is exactly one cache miss per compaction — the same tax
every harness pays [8][9] — after which a shorter, denser prefix re-caches, and
replay, crash-reload, and auditability all still hold.

The pattern has direct precedent, and Verda's is the same shape made explicit:

- OpenAI's Responses API ships server-side compaction as an opaque "compaction
  item" emitted in the stream, and its docs explicitly authorize dropping items
  that came before the most recent compaction item [20].
- OpenHands models forgetting as tombstone-style Condensation events on an
  append-only event log — the log itself is never edited; a view applies the
  markers when it builds the prompt [9].
- Codex tracks compaction windows (window number and ids) and can budget growth
  after the carried prefix — the same prefix-scoped trigger [11][12].

## The Turn Cycle

```text
  the turn cycle: one loop = one turn

  +-------------------------------------+
  | 1. drain the daemon's queue         |   events + completions (finished
  |   (events, completions)             |  commands, child messages) become
  +-------------------------------------+   the appended input of this turn
                   |
                   v
  +-------------------------------------+
  | 2. prompt = stable rendered prefix  |   the prefix is byte-identical to
  |   + appended input                  |  last turn's -> prompt cache hit
  +-------------------------------------+   (stable layer + latest compaction
                   |   record + high-signal suffix)
                   v
  +-------------------------------------+
  | 3. model call                       |   the loop is explicitly budgeted
  +-------------------------------------+   (default budget: one model call)
                   |
                   v
  +-------------------------------------+
  | 4. tool calls                       |  parallel, fan-out, await-all;
  |                                     |  no response body by ~60s? the
  +-------------------------------------+  command keeps running; the agent
                   |                     holds a handle; completion
                   v                     queues a turn
  +-------------------------------------+
  | 5. tool results appended            |
  +-------------------------------------+
                   |
                   v
  +-------------------------------------+
  | 6. persist to the database          |   every turn, every agent
  +-------------------------------------+   (the raw log is append-only and
                   |   never rewritten)
                   v
  +-------------------------------------+      +----------------------------------+
  | 7. prefix growth over budget?       |+----->| 8. compaction checkpoint         |
  |   no  -> next turn                  |      |   stage 1: prune stale tool      |
  |   yes -> stage 1 prune              |      |   results (rendered pointers;    |
  |       stage 2 if still over         |      |   data stays in the DB,          |
  +-------------------------------------+      |   reloadable)                    |
                                               |   stage 2: if still over         |
                                               |   budget, append one             |
                                               |   compaction record (one         |
                                               |   deliberate cache miss; raw     |
                                               |   log untouched)                 |
                                               +---------+------------------------+
                                                         |
  +-------------------------------------+                |
  | 9. next turn (back to 1)            |<-----------------+
  +-------------------------------------+
```

The loop stays cheap because compaction is a budgeted, appended event — not a rewrite.

## Sources

[1] https://code.claude.com/docs/en/prompt-caching — accessed 2026-08-15
[2] https://developers.openai.com/api/docs/guides/prompt-caching — accessed 2026-08-15
[3] https://ai.google.dev/gemini-api/docs/caching — accessed 2026-08-15
[4] https://platform.claude.com/docs/en/build-with-claude/prompt-caching — accessed 2026-08-15
[5] https://rig.rs/docs/integrations/model_providers/anthropic — accessed 2026-08-15
[6] https://github.com/0xPlaygrounds/rig/blob/main/crates/rig-agent/src/agent/prompt_request/mod.rs — accessed 2026-08-15
[7] https://arxiv.org/abs/2307.03172 — accessed 2026-08-15
[8] https://code.claude.com/docs/en/context-window — accessed 2026-08-15
[9] https://github.com/All-Hands-AI/agent-sdk/blob/main/openhands-sdk/openhands/sdk/context/condenser/README.md — accessed 2026-08-15
[10] https://github.com/google-gemini/gemini-cli/blob/main/packages/core/src/context/chatCompressionService.ts — accessed 2026-08-15
[11] https://developers.openai.com/codex/config-reference — accessed 2026-08-15
[12] https://github.com/openai/codex/blob/main/codex-rs/core/src/compact.rs — accessed 2026-08-15
[13] https://github.com/openai/codex/blob/main/codex-rs/core/src/tools/handlers/new_context_window_spec.rs — accessed 2026-08-15
[14] https://code.claude.com/docs/en/how-claude-code-works — accessed 2026-08-15
[15] https://code.claude.com/docs/en/model-config — accessed 2026-08-15
[16] https://research.trychroma.com/context-rot — accessed 2026-08-15
[17] https://aider.chat/docs/repomap.html — accessed 2026-08-15
[18] https://aider.chat/docs/usage/caching.html — accessed 2026-08-15
[19] https://aider.chat/docs/usage/commands.html — accessed 2026-08-15
[20] https://developers.openai.com/api/docs/guides/compaction — accessed 2026-08-15
