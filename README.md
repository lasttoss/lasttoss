# Tran Trung Dung — backend engineer, realtime game servers

I build the server side of games: **Go and Nakama**, 30 Hz tick loops, matchmaking, and the
Kubernetes around them. 14+ years in mobile games, 4 years running Nakama in production at 99.99%
uptime, including a greenfield build that went live in one month.

Most of what I know I learned the hard way: from players. A room that leaks into another player's
session, a tick loop that misses its budget under load, a rollout that disconnects everyone in a
match - those are the problems worth writing down, so most of these repositories include a *"what
the load test found"* section and the fix it forced.

## Selected work

| Repository | What it is | The number that matters |
|---|---|---|
| [udp-realtime-gameserver](https://github.com/lasttoss/udp-realtime-gameserver) | 30 Hz UDP game server in Go: authoritative rooms of 10, region bucketed matchmaking, HMAC signed join tickets, ghost snapshots, client-side prediction and reconciliation, Prometheus, a canvas viewer and a bot swarm, plus a Helm chart that runs it on Kubernetes | 1000 bots at a flat 30.0 snapshots/second each, tick p95 18 ms of a 33.3 ms budget |
| [nakama-go-server-kit](https://github.com/lasttoss/nakama-go-server-kit) | The part of a Nakama module that is hardest to test: a fixed-tick match loop that takes its time from an injected clock, shipped with the test kit that steps that time by hand | 95% of statements covered in one second under `-race`, and three races the tests found that a `sleep` would have hidden |
| [go-loadtest-swarm](https://github.com/lasttoss/go-loadtest-swarm) | The harness behind the numbers above: a YAML scenario, a latency histogram accurate to 1%, a report, and an exit code - so a load test can fail a build | p50 2ms next to a max of 1.2s in the same run, which is the pair a mean hides |
| [ts-nakama-game-demo](https://github.com/lasttoss/ts-nakama-game-demo) | Nakama server plugin in TypeScript: device auth, four RPCs, an authoritative match, storage-backed game state | private-room reuse without ever handing a player somebody else's room |
| [zinx-server](https://github.com/lasttoss/zinx-server) | TCP game server in Go (zinx) with JWT auth, MongoDB and Redis, and a chat room | the smoke test talks the raw TLV protocol, so the protocol itself is tested |
| [architecture-diagrams](https://github.com/lasttoss/architecture-diagrams) | How the log pipeline and the multi-region game backend fit together: ClickHouse, Airflow, Kafka, CockroachDB, geo routing | kept as mermaid sources so a diagram can be reviewed in a pull request |
| [go-log-api](https://github.com/lasttoss/go-log-api) | Ingest → rollup → metrics for game telemetry: batched writes, SQL rollups for DAU and revenue, backpressure | 503 under backpressure instead of unbounded memory |

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
