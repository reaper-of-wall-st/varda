# Sandbox

Every agent gets its own fully-furnished container: the user stops writing sandbox
scripts, and the daemon owns the whole lifecycle.

Varda is greenfield: the four crates are scaffolds and none of this exists yet.
Everything in this document is design intent, stated as such, not shipped
capability.

## One container per agent, always

One container per agent, always (design intent). No shared execution environment,
no "maybe a container" fallback. Genesis, the first agent, gets a container just
like every subagent that follows.

Inside the box, the agent gets the machine. The container is "your computer from
the terminal": full internet, full bash, full Python (design intent). The sandbox
does not weaken the agent; it unshackles it. Anything the user's terminal can do,
the agent can do. The sandbox answers "where does that mess live", not "what may
the agent do".

The internet part costs nothing on the default bridge: containers there get
outbound access by NAT, no port publishing [^3].

## The daemon owns the lifecycle

The user never writes a sandbox script again (design intent). The daemon — the
`varda-daemon` crate — creates containers on demand through the Docker Engine API
and owns every step of the container's life.

The Engine API is a REST interface [^1]; the current reference is v1.55, served by
Docker 29.7.x [^1]. Its container verb set is the lifecycle itself:

```text
create  ->  start  ->  inspect (state + IP)  ->  stop/kill  ->  remove
```

- `create` provisions the container and `start` brings it up; start is
  idempotent — the API answers 304 if it is already running [^2].
- `inspect` is how the daemon learns the container's state and its IP on the
  bridge; inspect is the documented way to learn the address after creation [^2].
- `stop`/`kill` tear it down; `remove` deletes it, and the API refuses with 409
  while the container is still running [^2].

That verb set is the entire surface, and the daemon is the only actor that calls
it. An agent inside a container never sees Docker (design intent).

## One daemon per directory

One directory is one world: one daemon, one shared workspace, the agents that
live in it (design intent). The daemon enforces that cardinality with two files
in `.varda/` — a lock file that makes the claim and a log file that shows it.
Start a second daemon in the same directory and it finds the lock and stands
down. No daemon registry, no global process; the directory is the boundary.

## The mounts: one world, one scratchpad

Each container carries two mounts (design intent):

- **The shared project workspace** — the world's common directory, `/workspace`
  for every agent in it, mounted read-write. Agents collaborate through it: one
  writes, another reads. It is a bind mount, so the files are the project's
  files, owned by the host [^2].
- **A per-agent `/tmp` scratch** — each agent's own disposable space. This one
  has a hard requirement: the per-agent `/tmp` must be a bind mount or a named
  volume, or it dies with the container. An unmounted `/tmp` is
  container-local storage; a bind mount lives on the host, and a named volume
  lives in Docker-managed storage the API explicitly does not remove when the
  container is removed [^2]. Agents are cattle (below), so the scratch has to
  outlive the cattle.

## No Kubernetes

No orchestration layer (design intent). Plain Docker, long-lived containers.
One host, one daemon, one container per agent, and a recreate-when-dead loop. If
that is too small, the fix is another directory and another daemon, not a
control plane.

## Cattle, and sacred state

Agents are cattle, state is sacred (design intent). A container is a commodity,
and the daemon does not mourn when one dies. If the agent was recently active,
the daemon relaunches it: same image, same mounts, state reloaded from the
persisted trace that [persistence.md](persistence.md) describes. The container
is gone; the agent walks on.

Because containers are cattle, the ownership rule stays strict: agents are
persistent by default, and the parent alone closes a child. The system never
reaps on its own and never touches a child's container — there is no
predictable shutdown signal to hand a reaper (design intent; stated the same
way in [subagents.md](subagents.md)).

## The whole path

Two paths in the picture. The lifecycle path runs down: daemon, Engine API,
container. The queue path runs the other way: one queue inside the daemon, and
every agent container dials the bridge gateway IP at the daemon's queue port —
that is the agent's only route into the system (design intent; the queue itself
is [message-queue.md](message-queue.md)'s territory).

```text
host
+-----------------------------------------------------------------------+
|                                                                       |
| varda-daemon (one per directory)                                      |
| +-----------------------------+  <--- queue path: every agent dials   |
| | in-daemon queue (firehose)   |       in over the network at the     |
| | bound to bridge / 0.0.0.0,   |       gateway IP (the host's address |
| | not 127.0.0.1 (queue port)   |       on the bridge; 172.17.0.1 by   |
| +---------------+--------------+       default) : queue port          |
|                 |  Docker Engine API (REST, v1.55)                    |
|                 |  create -> start -> inspect                         |
|                 |        -> stop/kill -> remove                       |
|                 v                                                     |
|  Docker Engine                                                        |
|                 |  creates, owns                                      |
|                 v                                                     |
| +--------------------------------------------+                        |
| | container (one per agent)                   |                       |
| |  eth0: 172.17.0.2 (default bridge subnet)   |                       |
| |  /workspace  <- bind mount, shared, rw      |                       |
| |  /tmp        <- per-agent volume or bind    |                       |
| |  dials gateway 172.17.0.1 : queue port      |                       |
| +--------------------------------------------+                        |
|                                                                       |
| .varda/    daemon.lock    daemon.log    state db                      |
|                                                                       |
+-----------------------------------------------------------------------+
```

Four caveats keep that diagram honest:

1. **The gateway IP is the host's address on the bridge.** On the default
   bridge, containers get addresses from 172.17.0.0/16 and the bridge's
   gateway — the host itself — is 172.17.0.1 by default [^3][^2]. It is the
   container's default route, so anything sent to it reaches the host [^3].
   Default, not guarantee: the subnet and gateway are configurable in the
   daemon's bridge settings [^3].
2. **Docker Desktop runs the engine in a VM.** On Mac and Windows the bridge
   lives inside a lightweight Linux VM, so 172.17.0.1 is the VM's address, not
   the user's host [^4]. The documented host alias there is
   `host.docker.internal`, which Docker provides automatically [^4][^5].
3. **Bind the queue to the bridge, not loopback.** A listener on 127.0.0.1
   exists only in the host's own network namespace; a container dialing the
   gateway IP can never reach it. The daemon must bind the queue port on the
   bridge interface or 0.0.0.0.
4. **The per-agent `/tmp` must be a bind mount or a named volume**, or it dies
   with the container [^2]. Restated from The mounts, because it is the one
   mount that silently breaks "state is sacred".

[^1]: https://docs.docker.com/reference/api/engine/ — accessed 2026-08-15
[^2]: https://docs.docker.com/reference/api/engine/version/v1.55.yaml — accessed 2026-08-15
[^3]: https://docs.docker.com/network/bridge/ — accessed 2026-08-15
[^4]: https://docs.docker.com/desktop/features/networking/ — accessed 2026-08-15
[^5]: https://docs.docker.com/compose/how-tos/networking/ — accessed 2026-08-15
