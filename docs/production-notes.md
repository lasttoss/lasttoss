# Production notes

The numbers below are from systems I run in production or from load tests I ran myself. Each one
carries the repository that reproduces it, because a number in a profile that nobody can check is a
number I would not believe either.

I do not publish client names, infrastructure addresses, credentials or customer data. Where a
figure comes from a specific incident, the incident is described and the identities are removed.

---

## 1. What a game server costs per 1,000 concurrent players

Measured with my own load generator ([`go-loadtest-swarm`](https://github.com/lasttoss/go-loadtest-swarm)),
against a Nakama-based backend, on a Kubernetes cluster. Two client profiles matter, and the
difference between them is the whole point:

| profile | application CPU | UDP path CPU | ingress CPU |
|---|---|---|---|
| 1 bot, one op every 4 s (a curious client) | 0.33 core / 1,000 CCU | 0.14 | 0.23 |
| 4 bots, ~3.7 ops/s each (a busy real client) | **0.92 core / 1,000 CCU** | **0.66** | **0.74** |

The first profile is the one that makes a capacity plan look easy. The second is the one that
matches a client that is actually playing, and it is the number I plan with.

Consequences I have used in production:

- **~1 core per 1,000 CCU** for the application tier at a realistic pace, with headroom on top; the
  UDP path is a *second* consumer at ~0.7 core per 1,000 CCU, which is why it gets its own
  deployment and its own CPU limit rather than sharing the API's.
- An API pod with a 500 m CPU limit reaches that limit at roughly 750 concurrent players on the UDP
  path ([`udp-realtime-gameserver`](https://github.com/lasttoss/udp-realtime-gameserver), `docs/kernel-and-limits.md`).
- A pod that is at its CPU limit does not look slow in `top` - it looks slow in
  `container_cpu_cfs_throttled_periods_total`. Sizing the UDP tier correctly took that metric from
  **1,314 throttled periods to 0**.
- **6,000-7,000 CCU per API pod** is what I commit to for a Nakama-based title, from production
  measurements, not from the load test's ceiling.

## 2. The Redis incident: 10,120 connections from one pod

A live title, one match-server pod, one shared Redis (Valkey). The server started answering
`max number of clients reached`, and new lobbies stopped being created.

What was actually happening:

- the pod held **10,120 client connections** to that one Redis, and the number grew with the number
  of live matches;
- the cause was structural, not a leak: **the match handler created a `redis.Client`**. Every client
  owns a pool, every pool opens sockets on first use, so the connection count was a function of the
  match count;
- `connected_clients` had no panel, so the first hypothesis was "Redis is overloaded". It was not:
  it was serving a normal command rate to an abnormal number of sockets.

The three changes that fixed it:

1. **one client per process**, cached by `(address, database)` - the database is part of the key,
   because two databases on one address are two keyspaces and one server;
2. **an explicit pool**, with `MaxActiveConns` as the ceiling - `PoolSize` is not a ceiling, the pool
   allocates past it under load;
3. **`SCAN` instead of `KEYS`** for the keyspace walks that admin endpoints do.

I reproduced all three in a lab with a real server, because "we fixed it" is not a measurement:
**[`valkey-scale-lab`](https://github.com/lasttoss/valkey-scale-lab)**. There, 300 matches reach 303
connections; 600 matches hit a 500-connection ceiling and 103 matches are refused; the fixed version
serves the same 600 matches with **7 connections**. Over 200,600 keys, `KEYS` stalls an unrelated
client for **18.7 ms** at worst while `SCAN` stalls it for **3.8 ms** - `SCAN` is the slower tool
overall and the better one.

One detail from that lab that I had wrong for years: when Redis hits `maxclients`, the client's
error is **`EOF`**, not `max number of clients reached`. The sentence is in the *server* log. That
is why searching the application code for it finds nothing.

## 3. Running more than one pod of a Nakama OSS server

Nakama OSS keeps three things per node (presence, match state, and the matchmaker's queue). Scaling
out is a series of deliberate trades, written up in
[`nakama-go-server-kit`](https://github.com/lasttoss/nakama-go-server-kit), `docs/multi-pod.md`:

- **relay + a device-kick bus** rather than sticky sessions: a player's socket may land on any pod,
  and a kick reaches them in **under a second** through Redis pub/sub;
- measured with **2,000 CCU across 2 pods** in a development cluster, relay included;
- **client-side integration is a decision, not a detail**: the public WebSocket URL becomes
  per-host instead of per-path, which means the client team has to ship a change before this is
  usable. I would not deploy it without their agreement, and I would say that in the planning
  meeting rather than in the postmortem.

## 4. What I will not claim

- **6,000 CCU is a load test**, not a production peak. The production peaks I can quote are lower and
  I quote them separately.
- **The cost figures I give are estimates** (roughly 30 USD/month for the small production setup),
  marked as estimates.
- **99.99% uptime refers to the game infrastructure I operate**, not to "bug-free code".
- **I do not write the Unity or Cocos client.** I design the wire contract, and I write the
  server-side artifacts the client team compiles - `.proto`, `.fbs`, fixtures - plus the tests that
  keep them honest ([`wire-codec`](https://github.com/lasttoss/wire-codec)). The client-side
  TypeScript in that repository is a *mirror* of the contract, checked by a test for drift; it is
  not a client implementation, and it is labelled as such in the file.

## 5. Where the rest of the evidence is

| claim | repository | the command that shows it |
|---|---|---|
| the capacity numbers | `go-loadtest-swarm` | `docs/capacity-model.md`, `make bench` |
| kernel limits and UDP sizing | `udp-realtime-gameserver` | `docs/kernel-and-limits.md` |
| the Redis incident, reproduced | `valkey-scale-lab` | `make measure` |
| multi-pod Nakama, relay, kick bus | `nakama-go-server-kit` | `docs/multi-pod.md` |
| what to panel during an incident | `otel-game-observability` | `docs/incident-metrics.md` |
| one declaration, three wire formats | `wire-codec` | `make gate` |
