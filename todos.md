# 30-Day Todo Plan: Distributed Key-Value Store

## Project Target

Build a Redis-compatible key-value store in Go with:

- TCP server and RESP protocol support.
- Core string commands.
- TTL expiry.
- Lists, hashes, sets, and sorted sets.
- AOF and RDB persistence.
- Pub/sub and pipelining.
- LRU eviction and transactions.
- Leader-follower replication.
- Benchmarks, profiling, Docker, and documentation.

This is an aggressive 30-day plan. The goal is to finish a strong interview-ready version, not a production-grade Redis replacement.

---

## Daily Working Rules

Follow these rules every day:

1. **Start with tests or expected behavior**
   - Write down what should work before implementing.
   - Add tests for edge cases, not only happy paths.

2. **Keep each day shippable**
   - Every day should end with code that builds.
   - If a feature is incomplete, hide it behind clear boundaries.

3. **Run validation daily**
   - Run `go test ./...`.
   - Run a few `redis-cli` checks once the TCP server exists.

4. **Document important decisions immediately**
   - Locking strategy.
   - Expiry strategy.
   - Persistence tradeoffs.
   - Replication limitations.

5. **Benchmark only after correctness**
   - First make it work.
   - Then measure.
   - Then optimize.

---

## Day 1: Project Setup and Architecture Skeleton

### Goals

- Initialize the Go project.
- Create the base directory structure.
- Add initial architecture notes.

### Tasks

- [ ] Create `go.mod` with Go 1.22+.
- [ ] Add base folders:
  - [ ] `cmd/radis-server/`
  - [ ] `internal/config/`
  - [ ] `internal/server/`
  - [ ] `internal/protocol/`
  - [ ] `internal/command/`
  - [ ] `internal/engine/`
  - [ ] `internal/datastructures/`
  - [ ] `internal/persistence/`
  - [ ] `internal/replication/`
  - [ ] `docs/`
  - [ ] `benchmarks/`
- [ ] Create `cmd/radis-server/main.go`.
- [ ] Add basic config defaults:
  - [ ] Host.
  - [ ] Port.
  - [ ] Data directory.
  - [ ] Persistence flags.
- [ ] Add minimal README with project purpose.
- [ ] Create first draft of `docs/architecture.md`.

### Validation

- [ ] Run `go test ./...`.
- [ ] Run `go run ./cmd/radis-server` and confirm it starts or prints expected placeholder output.

### Deliverable

A clean Go project skeleton with a runnable entrypoint.

---

## Day 2: TCP Server

### Goals

- Build the networking foundation.
- Accept multiple TCP clients.

### Tasks

- [ ] Implement `server.Server` type.
- [ ] Add TCP listener using `net.Listen`.
- [ ] Add accept loop.
- [ ] Spawn one goroutine per connection.
- [ ] Add graceful shutdown path.
- [ ] Add connection read/write loop placeholder.
- [ ] Add basic logging for connection open/close.
- [ ] Write tests for server startup if practical.

### Validation

- [ ] Start server locally.
- [ ] Connect with `nc localhost <port>`.
- [ ] Confirm server accepts and closes connections safely.
- [ ] Run `go test ./...`.

### Deliverable

A concurrent TCP server that can accept multiple clients.

---

## Day 3: RESP Parser

### Goals

- Parse Redis protocol input into command objects.

### Tasks

- [ ] Define RESP value types.
- [ ] Implement parsing for:
  - [ ] Simple strings.
  - [ ] Errors.
  - [ ] Integers.
  - [ ] Bulk strings.
  - [ ] Arrays.
  - [ ] Null bulk strings.
- [ ] Convert RESP array of bulk strings into command representation.
- [ ] Handle malformed input without panics.
- [ ] Add parser unit tests:
  - [ ] Valid command arrays.
  - [ ] Empty arrays.
  - [ ] Invalid bulk length.
  - [ ] Missing CRLF.
  - [ ] Partial input behavior.

### Validation

- [ ] Run parser tests.
- [ ] Run `go test ./...`.

### Deliverable

A tested RESP parser capable of reading Redis-style commands.

---

## Day 4: RESP Writer and Basic Commands

### Goals

- Return Redis-compatible responses.
- Support `PING` and `ECHO`.

### Tasks

- [ ] Implement RESP writer helpers:
  - [ ] Simple string.
  - [ ] Error.
  - [ ] Integer.
  - [ ] Bulk string.
  - [ ] Null bulk string.
  - [ ] Array.
- [ ] Define command router interface.
- [ ] Add command normalization to uppercase.
- [ ] Implement `PING`.
- [ ] Implement `ECHO`.
- [ ] Implement unknown command error.
- [ ] Wire parser, router, and writer into connection loop.

### Validation

- [ ] `redis-cli -p <port> PING` returns `PONG`.
- [ ] `redis-cli -p <port> ECHO hello` returns `hello`.
- [ ] Unknown command returns an error.
- [ ] Run `go test ./...`.

### Deliverable

First Redis-compatible command flow over TCP.

---

## Day 5: In-Memory Engine and String Commands Part 1

### Goals

- Implement typed in-memory storage.
- Add core string writes and reads.

### Tasks

- [ ] Define engine value type.
- [ ] Add type metadata for values.
- [ ] Implement basic map-backed store.
- [ ] Add locking around store access.
- [ ] Implement `SET key value`.
- [ ] Implement `GET key`.
- [ ] Implement `DEL key [key ...]`.
- [ ] Implement wrong-type error helper.
- [ ] Add unit tests for engine `GET`, `SET`, and `DEL`.
- [ ] Add TCP integration tests if possible.

### Validation

- [ ] `redis-cli SET name aritra` returns `OK`.
- [ ] `redis-cli GET name` returns `aritra`.
- [ ] `redis-cli DEL name` returns `1`.
- [ ] Run `go test ./...`.

### Deliverable

Working string storage with basic mutations.

---

## Day 6: String Commands Part 2 and Server Introspection

### Goals

- Complete the initial string command set.
- Add basic server commands.

### Tasks

- [ ] Implement `EXISTS key [key ...]`.
- [ ] Implement `INCR key`.
- [ ] Implement `DECR key`.
- [ ] Implement `MGET key [key ...]`.
- [ ] Implement `MSET key value [key value ...]`.
- [ ] Implement `DBSIZE`.
- [ ] Implement `FLUSHDB`.
- [ ] Implement initial `INFO` output.
- [ ] Add tests for integer parsing and invalid integer values.
- [ ] Add tests for multi-key commands.

### Validation

- [ ] Confirm all string commands through `redis-cli`.
- [ ] Confirm invalid arity errors.
- [ ] Confirm `INCR` rejects non-integer values.
- [ ] Run `go test ./...`.

### Deliverable

Complete initial string command set.

---

## Day 7: Expiry Engine

### Goals

- Add TTL metadata and lazy expiry.

### Tasks

- [ ] Add expiration field to stored values.
- [ ] Implement lazy expiry check on key access.
- [ ] Implement `EXPIRE key seconds`.
- [ ] Implement `PEXPIRE key milliseconds`.
- [ ] Implement `TTL key`.
- [ ] Implement `PTTL key`.
- [ ] Implement `PERSIST key`.
- [ ] Add tests for missing keys, persistent keys, and expiring keys.
- [ ] Document expiry semantics in `docs/architecture.md` or `docs/expiry.md`.

### Validation

- [ ] `SET x 1`, `EXPIRE x 1`, then `GET x` after delay returns null.
- [ ] `TTL` returns expected sentinel values.
- [ ] Run `go test ./...`.

### Deliverable

Lazy TTL expiry with Redis-like expiry commands.

---

## Day 8: Active Expiry Worker and Pipelining

### Goals

- Add background TTL cleanup.
- Ensure pipelined command streams work.

### Tasks

- [ ] Implement active expiry goroutine.
- [ ] Sample keys instead of scanning the whole database.
- [ ] Add expiry worker configuration.
- [ ] Ensure connection loop handles multiple RESP commands continuously.
- [ ] Improve buffered write flushing for command streams.
- [ ] Add tests for eventual active expiry cleanup.
- [ ] Add pipelining test with multiple commands in one connection.

### Validation

- [ ] Expired keys are eventually removed from `DBSIZE`.
- [ ] Pipelined commands return ordered responses.
- [ ] Run `go test ./...`.

### Deliverable

TTL engine with lazy and active expiry plus pipeline-compatible command handling.

---

## Day 9: Initial Benchmarks and Locking Review

### Goals

- Establish baseline performance.
- Review engine concurrency strategy.

### Tasks

- [ ] Add Go benchmarks for RESP parser.
- [ ] Add engine benchmarks for `GET` and `SET`.
- [ ] Run `redis-benchmark` for:
  - [ ] `PING`.
  - [ ] `GET`.
  - [ ] `SET`.
  - [ ] Pipelined `GET`/`SET`.
- [ ] Record results in `benchmarks/redis_benchmark.md`.
- [ ] Profile CPU during simple benchmark.
- [ ] Decide whether to keep single lock or move to sharded map.
- [ ] Document locking decision.

### Validation

- [ ] Benchmarks run successfully.
- [ ] Results are committed in benchmark notes.
- [ ] Run `go test ./...`.

### Deliverable

Baseline benchmark report and initial performance profile.

---

## Day 10: Lists

### Goals

- Implement list data type and commands.

### Tasks

- [ ] Create list/deque data structure.
- [ ] Add list typed value support.
- [ ] Implement `LPUSH`.
- [ ] Implement `RPUSH`.
- [ ] Implement `LPOP`.
- [ ] Implement `RPOP`.
- [ ] Implement `LLEN`.
- [ ] Implement `LRANGE` with negative indexes.
- [ ] Add tests for ordering, empty lists, and ranges.
- [ ] Add wrong-type tests.

### Validation

- [ ] Verify list commands through `redis-cli`.
- [ ] Run `go test ./...`.

### Deliverable

Redis-like list support.

---

## Day 11: Hashes

### Goals

- Implement hash data type and commands.

### Tasks

- [ ] Create hash value structure.
- [ ] Implement `HSET key field value [field value ...]`.
- [ ] Implement `HGET key field`.
- [ ] Implement `HDEL key field [field ...]`.
- [ ] Implement `HGETALL key`.
- [ ] Implement `HLEN key`.
- [ ] Add tests for insert, update, delete, and missing fields.
- [ ] Add wrong-type tests.

### Validation

- [ ] Verify hash commands through `redis-cli`.
- [ ] Run `go test ./...`.

### Deliverable

Redis-like hash support.

---

## Day 12: Sets

### Goals

- Implement set data type and commands.

### Tasks

- [ ] Create set value structure.
- [ ] Implement `SADD key member [member ...]`.
- [ ] Implement `SREM key member [member ...]`.
- [ ] Implement `SMEMBERS key`.
- [ ] Implement `SISMEMBER key member`.
- [ ] Implement `SCARD key`.
- [ ] Add tests for duplicate members.
- [ ] Add tests for missing keys.
- [ ] Add wrong-type tests.

### Validation

- [ ] Verify set commands through `redis-cli`.
- [ ] Run `go test ./...`.

### Deliverable

Redis-like set support.

---

## Day 13: Skip List Core

### Goals

- Build the sorted set foundation from scratch.

### Tasks

- [ ] Implement skip list node structure.
- [ ] Implement random level generation.
- [ ] Implement insert operation.
- [ ] Implement delete operation.
- [ ] Implement range traversal by rank.
- [ ] Implement score/member ordering.
- [ ] Add tests for insertion order.
- [ ] Add tests for deletion.
- [ ] Add randomized tests comparing skip list order to sorted slice order.

### Validation

- [ ] Skip list tests pass.
- [ ] Run `go test ./...`.

### Deliverable

Standalone tested skip list implementation.

---

## Day 14: Sorted Set Commands

### Goals

- Expose skip list through Redis-like sorted set commands.

### Tasks

- [ ] Create sorted set wrapper with map plus skip list.
- [ ] Implement `ZADD key score member [score member ...]`.
- [ ] Implement `ZREM key member [member ...]`.
- [ ] Implement `ZRANGE key start stop`.
- [ ] Add `ZRANGE ... WITHSCORES` support.
- [ ] Implement `ZSCORE key member`.
- [ ] Implement `ZRANK key member`.
- [ ] Add tests for score updates.
- [ ] Add tests for equal-score tie-breaking.
- [ ] Add command integration tests.

### Validation

- [ ] Verify sorted set commands through `redis-cli`.
- [ ] Run `go test ./...`.

### Deliverable

Skip-list-backed sorted set support.

---

## Day 15: Data Structure Integration and Compatibility Sweep

### Goals

- Stabilize all data types before persistence.

### Tasks

- [ ] Run a full command compatibility sweep manually with `redis-cli`.
- [ ] Add missing arity checks.
- [ ] Standardize error messages.
- [ ] Standardize null and empty-array responses.
- [ ] Review wrong-type handling for every command.
- [ ] Add integration tests for mixed data types.
- [ ] Update command compatibility matrix in README.
- [ ] Add docs for internal value model.

### Validation

- [ ] All supported commands behave consistently.
- [ ] Run `go test ./...`.

### Deliverable

Stable in-memory Redis-compatible command subset.

---

## Day 16: AOF Logging

### Goals

- Persist write commands to an append-only file.

### Tasks

- [ ] Create AOF writer.
- [ ] Define list of write commands.
- [ ] Serialize write commands as RESP arrays.
- [ ] Append only successful write commands.
- [ ] Add config for enabling/disabling AOF.
- [ ] Add fsync policy config:
  - [ ] `always`.
  - [ ] `everysec`.
  - [ ] `no`.
- [ ] Implement background fsync for `everysec`.
- [ ] Add tests for command serialization.

### Validation

- [ ] Write commands appear in AOF file.
- [ ] Read-only commands do not appear in AOF file.
- [ ] Run `go test ./...`.

### Deliverable

AOF write path with configurable fsync policy.

---

## Day 17: AOF Replay and Rewrite

### Goals

- Recover state from AOF.
- Compact AOF safely.

### Tasks

- [ ] Implement AOF replay on startup.
- [ ] Ensure replay does not append commands back into AOF.
- [ ] Add replay tests for all supported write command families.
- [ ] Implement AOF rewrite from current engine snapshot.
- [ ] Write rewrite output to temporary file.
- [ ] Atomically replace old AOF file.
- [ ] Add tests for rewrite correctness.
- [ ] Add corrupted/partial AOF handling.

### Validation

- [ ] Start server, write data, restart server, verify data remains.
- [ ] Run rewrite and verify data remains.
- [ ] Run `go test ./...`.

### Deliverable

Durable AOF persistence with replay and compaction.

---

## Day 18: RDB Snapshot Format

### Goals

- Design and implement snapshot serialization.

### Tasks

- [ ] Define RDB-like binary or structured snapshot format.
- [ ] Include snapshot version and metadata.
- [ ] Encode all supported value types.
- [ ] Encode expiry metadata.
- [ ] Implement engine snapshot method.
- [ ] Implement snapshot writer using temp file plus rename.
- [ ] Add unit tests for encoding each value type.

### Validation

- [ ] Snapshot file is created successfully.
- [ ] Snapshot includes strings, lists, hashes, sets, sorted sets, and expiry metadata.
- [ ] Run `go test ./...`.

### Deliverable

RDB snapshot writer and format definition.

---

## Day 19: RDB Load and Persistence Recovery Policy

### Goals

- Restore state from snapshots.
- Define startup recovery order.

### Tasks

- [ ] Implement snapshot reader.
- [ ] Validate snapshot version and checksum if added.
- [ ] Skip keys that are already expired on load.
- [ ] Add startup recovery logic.
- [ ] Decide recovery order when both AOF and RDB exist.
- [ ] Add automatic save policy configuration.
- [ ] Add tests for snapshot round-trip.
- [ ] Add tests for corrupt snapshot handling.
- [ ] Document AOF/RDB tradeoffs in `docs/persistence.md`.

### Validation

- [ ] Start server from snapshot and verify data.
- [ ] Verify recovery behavior with both AOF and RDB enabled.
- [ ] Run `go test ./...`.

### Deliverable

RDB load path and documented persistence recovery policy.

---

## Day 20: Pub/Sub

### Goals

- Add Redis-like publish/subscribe support.

### Tasks

- [ ] Create pub/sub broker.
- [ ] Track subscribers by channel.
- [ ] Implement `SUBSCRIBE channel [channel ...]`.
- [ ] Implement `UNSUBSCRIBE [channel ...]`.
- [ ] Implement `PUBLISH channel message`.
- [ ] Add subscriber connection mode.
- [ ] Clean up subscriptions on disconnect.
- [ ] Protect against slow subscriber blocking.
- [ ] Add multi-client tests.

### Validation

- [ ] Open two `redis-cli` sessions.
- [ ] Subscribe in one session.
- [ ] Publish in another session.
- [ ] Confirm message delivery.
- [ ] Run `go test ./...`.

### Deliverable

Working pub/sub across client connections.

---

## Day 21: LRU Eviction

### Goals

- Enforce approximate memory limit with eviction.

### Tasks

- [ ] Add memory limit config.
- [ ] Track approximate memory usage per key/value.
- [ ] Track key access timestamp or logical clock.
- [ ] Update access metadata on reads and writes.
- [ ] Implement sampled LRU eviction.
- [ ] Prefer expired keys during cleanup where practical.
- [ ] Add tests for eviction under memory pressure.
- [ ] Document approximation limitations.

### Validation

- [ ] Configure small memory limit.
- [ ] Insert many keys.
- [ ] Confirm older keys are evicted.
- [ ] Run `go test ./...`.

### Deliverable

Approximate LRU eviction under memory limit.

---

## Day 22: Transactions

### Goals

- Implement transaction queueing and optimistic locking.

### Tasks

- [ ] Add per-connection transaction state.
- [ ] Implement `MULTI`.
- [ ] Queue commands while in transaction mode.
- [ ] Implement `EXEC`.
- [ ] Implement `DISCARD`.
- [ ] Add key version tracking in engine.
- [ ] Implement `WATCH key [key ...]`.
- [ ] Abort transaction if watched key changed.
- [ ] Add tests for basic transaction behavior.
- [ ] Add tests for watched key conflicts.

### Validation

- [ ] Verify `MULTI`, queued commands, and `EXEC` through `redis-cli`.
- [ ] Verify `WATCH` aborts on conflict.
- [ ] Run `go test ./...`.

### Deliverable

Redis-like transaction subset.

---

## Day 23: Replication Protocol Design and Leader Write Stream

### Goals

- Design replication internals.
- Start leader-side write streaming.

### Tasks

- [ ] Write `docs/replication.md` draft.
- [ ] Define follower handshake request.
- [ ] Define leader sync response.
- [ ] Define replication offset model.
- [ ] Add leader role config.
- [ ] Track replication offset for successful writes.
- [ ] Add connected follower registry.
- [ ] Stream successful write commands to followers.
- [ ] Expose replication info in `INFO`.

### Validation

- [ ] Leader starts in leader mode.
- [ ] Write commands increment offset.
- [ ] `INFO` shows role and offset.
- [ ] Run `go test ./...`.

### Deliverable

Leader-side replication foundation.

---

## Day 24: Follower Initial Sync

### Goals

- Make follower connect to leader and receive full state.

### Tasks

- [ ] Add follower role config.
- [ ] Implement follower connection to leader.
- [ ] Implement replication handshake.
- [ ] Reuse RDB snapshot format for full sync.
- [ ] Load leader snapshot into follower engine.
- [ ] Mark follower as read-only for normal clients.
- [ ] Reject client writes on follower.
- [ ] Add integration test for full sync.

### Validation

- [ ] Start leader.
- [ ] Write data to leader.
- [ ] Start follower.
- [ ] Confirm follower has leader data.
- [ ] Confirm follower rejects writes.
- [ ] Run `go test ./...`.

### Deliverable

Follower initial synchronization from leader.

---

## Day 25: Live Replication and Reconnect

### Goals

- Apply leader writes on follower after initial sync.
- Handle connection loss.

### Tasks

- [ ] Apply streamed write commands on follower.
- [ ] Ensure follower replay does not append to local AOF incorrectly unless intended.
- [ ] Track follower applied offset.
- [ ] Add reconnect loop with backoff.
- [ ] Implement partial catch-up if leader has required offset.
- [ ] Fall back to full resync if partial catch-up is unavailable.
- [ ] Add manual promotion mechanism.
- [ ] Add tests for live replication.
- [ ] Add tests for reconnect and convergence.

### Validation

- [ ] Start leader and follower.
- [ ] Write new keys to leader.
- [ ] Confirm follower receives them.
- [ ] Restart follower and confirm it catches up.
- [ ] Run `go test ./...`.

### Deliverable

Live leader-follower replication with reconnection behavior.

---

## Day 26: Benchmarking and Profiling Pass

### Goals

- Run full benchmarks and capture profiles.

### Tasks

- [ ] Run `redis-benchmark` against this server.
- [ ] Run matching benchmark against real Redis.
- [ ] Benchmark with pipelining enabled.
- [ ] Benchmark with AOF disabled.
- [ ] Benchmark with AOF `everysec`.
- [ ] Capture CPU profile during GET/SET workload.
- [ ] Capture memory profile during large dataset load.
- [ ] Capture goroutine profile during many client connections.
- [ ] Record results in `benchmarks/redis_benchmark.md`.

### Validation

- [ ] Benchmark data includes environment details.
- [ ] Results include ops/sec and p50/p99 latency where available.
- [ ] Profiles are saved under `benchmarks/profiles/`.
- [ ] Run `go test ./...`.

### Deliverable

Honest benchmark and profiling report.

---

## Day 27: Optimization Pass

### Goals

- Improve one or more bottlenecks using measured data.

### Tasks

- [ ] Review CPU profile hotspots.
- [ ] Review allocation-heavy paths.
- [ ] Optimize RESP parsing if it is a bottleneck.
- [ ] Reduce unnecessary string/byte conversions.
- [ ] Consider buffer reuse where safe.
- [ ] Review lock contention.
- [ ] Implement sharded map if justified by benchmark data.
- [ ] Re-run before/after benchmarks.
- [ ] Document optimization results.

### Validation

- [ ] Before/after numbers are recorded.
- [ ] Optimization does not break tests.
- [ ] Run `go test ./...`.

### Deliverable

Measured performance improvement with documented reasoning.

---

## Day 28: Docker, CI, and Developer Experience

### Goals

- Make the project easy to run and validate.

### Tasks

- [ ] Add multi-stage `Dockerfile`.
- [ ] Add `.dockerignore`.
- [ ] Add `docker-compose.yml` for single-node server.
- [ ] Add optional compose setup for leader/follower.
- [ ] Add GitHub Actions workflow:
  - [ ] Build.
  - [ ] Test.
  - [ ] Lint if configured.
- [ ] Add Makefile or task commands if desired:
  - [ ] `make test`.
  - [ ] `make run`.
  - [ ] `make bench`.
  - [ ] `make docker-build`.
- [ ] Verify fresh setup instructions.

### Validation

- [ ] Docker image builds.
- [ ] Container starts.
- [ ] `redis-cli` can connect to containerized server.
- [ ] CI workflow passes locally as much as possible.
- [ ] Run `go test ./...`.

### Deliverable

Runnable Dockerized project with CI checks.

---

## Day 29: Documentation and Interview Polish

### Goals

- Make the project presentable and easy to understand.

### Tasks

- [ ] Expand README.
- [ ] Add architecture diagram.
- [ ] Add supported command matrix.
- [ ] Add quick start instructions.
- [ ] Add configuration documentation.
- [ ] Add persistence documentation.
- [ ] Add replication documentation.
- [ ] Add benchmark summary.
- [ ] Add limitations section.
- [ ] Add future improvements section.
- [ ] Add resume bullet with measured numbers.

### Validation

- [ ] Follow README from a clean terminal and verify it works.
- [ ] Confirm docs do not overclaim production readiness.
- [ ] Confirm benchmark claims are backed by data.

### Deliverable

Interview-ready documentation.

---

## Day 30: Final Stabilization and Demo

### Goals

- Final test pass.
- Prepare a demo script.
- Cut a stable project milestone.

### Tasks

- [ ] Run full test suite.
- [ ] Run race detector on important packages if feasible.
- [ ] Run final redis-cli compatibility sweep.
- [ ] Run final benchmark sample.
- [ ] Test persistence restart flow.
- [ ] Test pub/sub demo.
- [ ] Test transaction demo.
- [ ] Test leader/follower demo.
- [ ] Create `docs/demo.md` with step-by-step demo commands.
- [ ] Tag or mark final milestone in git.
- [ ] Write final project retrospective:
  - [ ] What works.
  - [ ] What is simplified.
  - [ ] What would be improved next.

### Validation

- [ ] All tests pass.
- [ ] Demo script works from start to finish.
- [ ] README is accurate.
- [ ] Project can be explained in 2 minutes and in 15 minutes.

### Deliverable

Completed 30-day Redis-like key-value store project milestone.

---

## Weekly Milestones

### End of Week 1

- TCP server works.
- RESP parser and writer work.
- `PING`, `ECHO`, and string commands work.
- TTL lazy expiry exists.

### End of Week 2

- Active expiry works.
- Pipelining works.
- Initial benchmarks are recorded.
- Lists, hashes, and sets work.

### End of Week 3

- Skip list works.
- Sorted sets work.
- AOF and RDB persistence work.
- Pub/sub works.

### End of Week 4

- LRU eviction works.
- Transactions work.
- Replication works.
- Benchmarks and optimization pass are complete.

### Final 2 Days

- Docker and CI are complete.
- Documentation is complete.
- Final demo and stabilization are complete.

---

## Minimum Viable Version

If time becomes tight, prioritize this version:

- TCP server.
- RESP parser/writer.
- `redis-cli` compatibility.
- Strings.
- TTL expiry.
- Lists, hashes, sets.
- Skip-list-backed sorted sets.
- AOF persistence.
- Benchmarks.
- README and architecture docs.

This version is already strong enough for interviews.

---

## Stretch Goals

Only attempt these after the minimum viable version is stable:

- Exact LRU comparison.
- More Redis commands.
- Cluster-style partitioning.
- Lua scripting subset.
- RESP3 support.
- TLS support.
- More advanced replication acknowledgements.
- Jepsen-style consistency testing.
- Web dashboard for metrics.

---

## Final Checklist

- [ ] Server runs from `go run ./cmd/radis-server`.
- [ ] Server works with `redis-cli`.
- [ ] All documented commands are tested.
- [ ] TTL expiry works lazily and actively.
- [ ] All data structures are implemented from scratch.
- [ ] Sorted sets use a skip list.
- [ ] AOF replay restores state.
- [ ] RDB snapshot restores state.
- [ ] Pub/sub works across clients.
- [ ] Pipelining improves throughput.
- [ ] LRU eviction works with memory limits.
- [ ] Transactions work for supported semantics.
- [ ] Leader-follower replication demo works.
- [ ] Benchmarks compare against Redis.
- [ ] README includes honest limitations.
- [ ] Docker setup works.
- [ ] CI passes.
- [ ] Resume bullet includes real measured numbers.
