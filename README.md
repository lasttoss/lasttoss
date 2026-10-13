# Tran Trung Dung — backend engineer, realtime game servers

I build the server side of games: **Go and Nakama**, 30 Hz tick loops, matchmaking, and the
Kubernetes around them. 14+ years in mobile games, 4 years running Nakama in production at 99.99%
uptime, including a greenfield build that went live in one month.

Most of what I know I learned the hard way: from players. A room that leaks into another player's
session, a tick loop that misses its budget under load, a rollout that disconnects everyone in a
match - those are the problems worth writing down, so most of these repositories include a *"what
the load test found"* section and the fix it forced.

- **[Production notes](docs/production-notes.md)** - capacity numbers per 1,000 players, a Redis connection incident I reproduced, and what I will not claim

## Selected work

| Repository | What it is | The number that matters |
|---|---|---|
| [udp-realtime-gameserver](https://github.com/lasttoss/udp-realtime-gameserver) | 30 Hz UDP game server in Go: authoritative rooms of 10, region bucketed matchmaking, HMAC signed join tickets, ghost snapshots, client-side prediction and reconciliation, Prometheus, a canvas viewer and a bot swarm, plus a Helm chart that runs it on Kubernetes | 1000 bots at a flat 30.0 snapshots/second each, tick p95 18 ms of a 33.3 ms budget |
| [nakama-go-server-kit](https://github.com/lasttoss/nakama-go-server-kit) | The part of a Nakama module that is hardest to test: a fixed-tick match loop that takes its time from an injected clock, shipped with the test kit that steps that time by hand | 95% of statements covered in one second under `-race`, and three races the tests found that a `sleep` would have hidden |
| [valkey-scale-lab](https://github.com/lasttoss/valkey-scale-lab) | A Redis (Valkey) connection incident reproduced against a real server: a client per match, the pool that replaced it, and `KEYS` against `SCAN` | 600 live matches held 500 connections (the server's ceiling) and 103 were refused; one pooled client served the same 600 with single-digit connections, and `SCAN` cut the worst round trip an unrelated client saw from ~18.7 ms to ~3.8 ms |
| [go-loadtest-swarm](https://github.com/lasttoss/go-loadtest-swarm) | The harness behind the numbers above: a YAML scenario, a latency histogram accurate to 1%, a report, and an exit code - so a load test can fail a build | p50 2ms next to a max of 1.2s in the same run, which is the pair a mean hides |
| [ts-nakama-game-demo](https://github.com/lasttoss/ts-nakama-game-demo) | Nakama server plugin in TypeScript: device auth, four RPCs, an authoritative match, storage-backed game state | private-room reuse without ever handing a player somebody else's room |
| [zinx-server](https://github.com/lasttoss/zinx-server) | TCP game server in Go (zinx) with JWT auth, MongoDB and Redis, and a chat room | the smoke test talks the raw TLV protocol, so the protocol itself is tested |
| [architecture-diagrams](https://github.com/lasttoss/architecture-diagrams) | How the log pipeline and the multi-region game backend fit together: ClickHouse, Airflow, Kafka, CockroachDB, geo routing | kept as mermaid sources so a diagram can be reviewed in a pull request |
| [go-log-api](https://github.com/lasttoss/go-log-api) | Ingest → rollup → metrics for game telemetry: batched writes, SQL rollups for DAU and revenue, backpressure | 503 under backpressure instead of unbounded memory |
| [nakama-migration-kit](https://github.com/lasttoss/nakama-migration-kit) | PlayFab / SmartFox / GameSparks → Nakama: a resumable importer that writes the document and the record of it in one storage write, and a reconciler that compares meaning instead of bytes | 10,000 players → 47,500 documents; the second run wrote **0** and skipped all 47,500; a run killed mid-batch resumed and left **0** checkpoints behind |
| [liveops-console](https://github.com/lasttoss/liveops-console) | LiveOps control plane: versioned remote config with rollback, an audit trail the database refuses to let you edit, and a scheduled worker that produces one effect under concurrent workers | **55.7%** statement coverage against a real PostgreSQL; every concurrency claim is asserted by a test, not described |
| [rtc-media-bench](https://github.com/lasttoss/rtc-media-bench) | WebRTC measurement harness: real pion participants through a vendored open-source SFU on Kubernetes, plus a catalogue of four ways a call dies quietly | join p95 **12.3 ms** in-cluster; one of the four faults reports **no error at all** while the call never starts |
| [ui-test-orchestrator](https://github.com/lasttoss/ui-test-orchestrator) | Browser suite runner: a hand-written DevTools protocol driver, visual assertions that need two numbers, and a flake policy kept in a committed file | a 250 ms wait for a 900 ms endpoint flaked **43.8%** of runs; waiting 8 s flaked **0 of 168** |
| [otel-game-observability](https://github.com/lasttoss/otel-game-observability) | A game backend small enough to read in one sitting, instrumented so that every step of a match is one span | "which match was slow" answered from a trace, on the stack that produced it |
| [tech-radar](https://github.com/lasttoss/tech-radar) | A radar for model releases that explains what changed instead of ranking it - with no model provider by design | 6 of 7 frame fields correct in **9.1 s / 62 tokens**, run locally and quoted with the command |
| [decision-model-service](https://github.com/lasttoss/decision-model-service) | A narrow decision service: one log line in, one of three labels out, with a fallback that always answers and an evaluation that decides whether any of it is worth keeping | p50 **0.04 ms**; it commits to a label 41% of the time and is right 94% of those |
| [k8s-game-server-charts](https://github.com/lasttoss/k8s-game-server-charts) | A Helm chart for a game server: a rollout that does not disconnect the matches in progress, and probes that tell a database blip apart from a dead pod | verified on a kind cluster - 2 pods `1/1 Running`, `helm test` Succeeded, HPA and PDB created |
| [agent-memory-workflow](https://github.com/lasttoss/agent-memory-workflow) | How I run a multi-session agent memory: the rules a session starts with, the handoff it leaves for the next one, and the redactor that keeps a credential out of memory | the redactor is the package with the tests: a token that reaches memory has already leaked, so it is caught before it is written |

## How I work

- **Measure, then claim.** If a repository says "30 Hz" or "1000 clients", there is a harness in it
  that produced the number, and the run is in the README - `go-loadtest-swarm` is that harness, and its
  own README shows the run where a local server answered in 2ms at the median and 1.2 seconds at worst.
- **The interesting bug is the one that looks like it works.** Five of the fixes in the
  repositories above came from a test or a load run that disagreed with a green checkmark: a spawn
  ring with a sign error putting half the room on the same spot, a catch-all route answering 200
  for every typo, a chat room writing to a nil map, a token validator turning a garbage token into
  a 500, and a fake clock that read its own instant without the lock while handing a tick over -
  found by `-race` the moment the test suite grew, which is the one bug a test kit must not have.
- **One command to run it.** Every repository here starts with `docker compose up` or `make run`,
  and its README says how long the numbers took to produce.
- **Kubernetes is not magic, UDP is not TCP.** The relay ships with the reasoning behind its chart,
  including the rollout deadlock that `hostPort` and `maxUnavailable: 0` cause on a single node.

## Contact

- Email: **3w.rain@gmail.com**
- CV: available on request (EN / VI / 中文)
- Open to remote contract and full-time roles in game backend, platform and infrastructure.

<sub>Everything here is original work written on personal time; see each repository's
`PROVENANCE.md`.</sub>

## Repository standard

Six items, applied where they mean something rather than everywhere. The exclusions are the point of
the table: an item that cannot be honest in a repository of this kind is left out and said so.

| Item | Here |
|---|---|
| `docker-compose.yml` | not applicable - a profile README has no service |
| `Dockerfile` | not applicable - same reason |
| Helm chart | not applicable - same reason |
| Diagram | not applicable - a README-only repository carries no diagrams |
| Tests, run in CI | not applicable - there is no code to test |
| CI + `Makefile` | not applicable - nothing to build |
