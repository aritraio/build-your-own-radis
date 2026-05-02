# Distributed Key-Value Store Workflow

## Project Goal

Build a Redis-compatible, in-memory distributed key-value store in Go from scratch.

The final system should support:

- **RESP protocol compatibility** so `redis-cli` and Redis client libraries can talk to the server.
- **Core Redis-like commands** for strings, expiry, lists, hashes, sets, and sorted sets.
- **Persistence** through append-only files and snapshots.
- **Advanced runtime features** like pub/sub, pipelining, LRU eviction, and transactions.
- **Leader-follower replication** with initial sync and live command streaming.
- **Benchmarks and documentation** that explain tradeoffs honestly.

The goal is not to beat Redis. The goal is to understand and demonstrate how Redis-like systems work internally.

---

## Guiding Engineering Principles

### 1. Build from first principles

Do not wrap an existing embedded database or key-value store. The storage engine, command router, protocol parser, data structures, persistence layer, and replication layer should be implemented directly.

### 2. Keep Redis compatibility where practical

The project should work with `redis-cli` for the supported command subset. Compatibility should be validated continuously as commands are added.

### 3. Prefer correctness before optimization

Every major feature should first be made correct with tests, then benchmarked, then optimized.

### 4. Design for explainability

Every non-trivial decision should be documented:

- Why a certain locking strategy was chosen.
- Why a certain persistence policy is the default.
- Why sorted sets use a skip list.
- Why active expiry uses sampling instead of scanning all keys.
- Where the implementation intentionally differs from Redis.

### 5. Benchmark honestly

Compare against real Redis, but do not claim to be faster than Redis. Explain performance differences using concrete observations from benchmarks and profiling.

---

## Recommended Repository Structure

```text
build-your-own-radis/
├── cmd/
│   └── radis-server/
│       └── main.go
├── internal/
│   ├── config/
│   │   └── config.go
│   ├── protocol/
│   │   ├── resp.go
│   │   ├── parser.go
│   │   └── writer.go
│   ├── server/
│   │   ├── server.go
│   │   ├── connection.go
│   │   └── pipeline.go
│   ├── command/
│   │   ├── command.go
│   │   ├── router.go
│   │   ├── strings.go
│   │   ├── expiry.go
│   │   ├── lists.go
│   │   ├── hashes.go
│   │   ├── sets.go
│   │   ├── sorted_sets.go
│   │   ├── pubsub.go
│   │   ├── transactions.go
│   │   └── server.go
│   ├── engine/
│   │   ├── store.go
│   │   ├── value.go
│   │   ├── shard.go
│   │   ├── expiry.go
│   │   ├── lru.go
│   │   └── snapshot.go
│   ├── datastructures/
│   │   ├── list.go
│   │   ├── set.go
│   │   ├── hash.go
│   │   └── skiplist.go
│   ├── persistence/
│   │   ├── aof.go
│   │   ├── aof_rewrite.go
│   │   ├── rdb.go
│   │   └── recovery.go
│   ├── pubsub/
│   │   ├── broker.go
│   │   └── subscriber.go
│   ├── replication/
│   │   ├── leader.go
│   │   ├── follower.go
│   │   ├── offset.go
│   │   └── sync.go
│   └── metrics/
│       ├── info.go
│       └── stats.go
├── tests/
│   ├── integration/
│   ├── compatibility/
│   └── persistence/
├── benchmarks/
│   ├── redis_benchmark.md
│   ├── microbenchmarks_test.go
│   └── profiles/
├── docs/
│   ├── architecture.md
│   ├── protocol.md
│   ├── persistence.md
│   ├── replication.md
│   └── performance.md
├── Dockerfile
├── docker-compose.yml
├── go.mod
├── go.sum
├── README.md
├── workflow.md
└── todos.md
```

This structure keeps the public executable small and pushes implementation details into `internal/` packages.

---

## Phase 0: Project Foundation

### Objective

Create the base Go project, define the runtime configuration model, and establish development quality gates.

### Steps

1. **Initialize Go module**
   - Create `go.mod`.
   - Use Go 1.22 or newer.
   - Add `testify` for readable tests.

2. **Create server entrypoint**
   - Add `cmd/radis-server/main.go`.
   - Parse basic configuration from flags or environment variables:
     - Host.
     - Port.
     - Data directory.
     - AOF enabled or disabled.
     - RDB enabled or disabled.
     - Memory limit.
     - Role: leader or follower.

3. **Add basic config package**
   - Keep defaults centralized.
   - Validate config before starting the server.

4. **Add basic logging and shutdown handling**
   - Handle `SIGINT` and `SIGTERM`.
   - Flush persistence files on shutdown.
   - Close active connections cleanly.

5. **Set up quality tools**
   - Add `go test ./...` as the minimum validation command.
   - Add `golangci-lint` configuration later once code shape stabilizes.
   - Add GitHub Actions for build and tests.

### Exit Criteria

- `go test ./...` passes.
- `go run ./cmd/radis-server` starts without crashing.
- Server accepts configuration from flags or environment variables.
- Project has a clear directory structure.

---

## Phase 1: TCP Server and RESP Protocol

### Objective

Build a network server that accepts Redis-style client connections and parses RESP commands.

### Key Concepts

RESP supports multiple wire types:

- Simple strings: `+OK\r\n`
- Errors: `-ERR message\r\n`
- Integers: `:1\r\n`
- Bulk strings: `$5\r\nhello\r\n`
- Arrays: `*2\r\n$3\r\nGET\r\n$3\r\nkey\r\n`
- Null bulk strings: `$-1\r\n`

Most client commands arrive as RESP arrays.

### Steps

1. **Create TCP listener**
   - Use Go's `net.Listen`.
   - Accept connections in a loop.
   - Spawn one goroutine per connection.

2. **Implement connection handler**
   - Use buffered reads and writes.
   - Keep each connection alive until the client disconnects or a fatal protocol error occurs.
   - Avoid reading the entire connection into memory.

3. **Implement RESP parser**
   - Parse arrays of bulk strings into a command object.
   - Validate malformed input.
   - Return protocol-level errors without crashing the server.

4. **Implement RESP writer**
   - Write simple strings, bulk strings, arrays, integers, null values, and errors.
   - Keep response formatting centralized.

5. **Add first server commands**
   - `PING`
   - `ECHO`
   - Unknown-command error response.

6. **Validate with redis-cli**
   - Start the server.
   - Run `redis-cli -p <port> PING`.
   - Confirm `PONG` response.

### Design Decisions to Document

- Why goroutine-per-connection is acceptable for the project.
- How the parser handles partial reads.
- What malformed protocol inputs produce.
- How pipelining will later reuse the same parser loop.

### Exit Criteria

- Multiple clients can connect concurrently.
- `redis-cli PING` works.
- `redis-cli ECHO hello` works.
- RESP parser has unit tests for valid and invalid input.

---

## Phase 2: Command Router and String Engine

### Objective

Add an in-memory storage engine and support core string commands.

### Commands

- `GET key`
- `SET key value`
- `DEL key [key ...]`
- `EXISTS key [key ...]`
- `INCR key`
- `DECR key`
- `MGET key [key ...]`
- `MSET key value [key value ...]`
- `DBSIZE`
- `FLUSHDB`
- `INFO`

### Steps

1. **Define command model**
   - Normalize command names to uppercase.
   - Preserve original arguments as byte slices or strings.
   - Add arity validation.

2. **Build command router**
   - Map command names to handler functions.
   - Keep protocol parsing separate from command execution.
   - Return RESP-compatible result objects.

3. **Design value model**
   - Store type metadata with each value.
   - Support at least:
     - String.
     - List.
     - Hash.
     - Set.
     - Sorted set.
   - Reject wrong-type operations with Redis-like errors.

4. **Implement storage engine**
   - Start with a map protected by a lock.
   - Prefer a sharded map once basic behavior is correct.
   - Keep expiry metadata close to the stored value.

5. **Implement string commands**
   - `GET` returns null for missing keys.
   - `SET` overwrites existing values.
   - `INCR` and `DECR` validate integer string values.
   - `MSET` should be atomic at command level.

6. **Implement server introspection commands**
   - `DBSIZE` returns number of non-expired keys.
   - `INFO` returns useful server and stats fields.
   - `FLUSHDB` clears the current database.

7. **Add tests**
   - Unit tests for each command.
   - Integration tests over TCP.
   - Compatibility tests using Redis clients where practical.

### Design Decisions to Document

- Locking strategy for the first implementation.
- Whether values are stored as strings, bytes, or typed structs.
- How wrong-type errors are represented.
- Whether command handlers mutate state directly or go through an engine interface.

### Exit Criteria

- Core string commands work through `redis-cli`.
- Unit and integration tests pass.
- Wrong-type and invalid-arity errors are tested.
- Basic server stats are available through `INFO`.

---

## Phase 3: Expiry Engine

### Objective

Implement TTL support with both lazy expiry and active background expiry.

### Commands

- `EXPIRE key seconds`
- `TTL key`
- `PEXPIRE key milliseconds`
- `PTTL key`
- `PERSIST key`

### Steps

1. **Represent expiry metadata**
   - Store expiration time as Unix milliseconds or `time.Time`.
   - Missing expiry means persistent key.

2. **Implement lazy expiry**
   - On key access, check whether the key is expired.
   - If expired, delete it before returning command result.

3. **Implement active expiry worker**
   - Run a background goroutine.
   - Sample a limited number of keys per cycle.
   - Delete expired sampled keys.
   - Continue sampling more aggressively if many sampled keys are expired.

4. **Implement expiry commands**
   - `EXPIRE` and `PEXPIRE` set expiration only if key exists.
   - `TTL` and `PTTL` return Redis-like sentinel values:
     - Missing key.
     - Existing key without expiry.
     - Existing key with expiry.
   - `PERSIST` removes expiry metadata.

5. **Add expiry tests**
   - Immediate expiry.
   - Delayed expiry.
   - Persistent keys.
   - Overwriting a key resets or updates expiry according to selected semantics.
   - Active expiry eventually removes expired keys.

6. **Benchmark expiry strategies**
   - Compare naive full-scan expiry to sampled active expiry.
   - Document CPU and latency impact.

### Design Decisions to Document

- Why lazy expiry alone is insufficient.
- Why full scans are avoided.
- How often the active expiry worker runs.
- How sampling size is chosen.

### Exit Criteria

- TTL commands work from `redis-cli`.
- Expired keys are invisible to reads.
- Background expiry removes keys without scanning the entire database every cycle.
- Expiry behavior is covered by tests.

---

## Phase 4: Pipelining and Initial Benchmarks

### Objective

Support multiple commands sent in one TCP round-trip and measure baseline performance.

### Steps

1. **Ensure parser loop supports command streams**
   - Read one RESP command.
   - Execute it.
   - Write one response.
   - Continue reading without waiting for a new connection.

2. **Avoid unnecessary flushes**
   - Use buffered writes.
   - Flush at sensible boundaries.
   - Preserve correctness for interactive clients.

3. **Run redis-benchmark**
   - Benchmark `PING`, `GET`, and `SET`.
   - Benchmark with and without pipelining.
   - Record throughput and latency.

4. **Add Go micro-benchmarks**
   - Parser benchmark.
   - Router benchmark.
   - Engine `GET`/`SET` benchmark.
   - Expiry check benchmark.

5. **Profile obvious bottlenecks**
   - Use `pprof` for CPU and memory.
   - Check allocation-heavy paths.
   - Identify parser allocations.

### Design Decisions to Document

- Whether each response is flushed immediately or batched.
- How parser memory reuse is handled.
- Baseline ops/sec before deeper optimization.

### Exit Criteria

- Pipelined requests return ordered responses.
- `redis-benchmark` can run against the server.
- Initial benchmark numbers are recorded.
- At least one CPU or memory profile is captured.

---

## Phase 5: Lists

### Objective

Implement Redis-like list operations.

### Commands

- `LPUSH key element [element ...]`
- `RPUSH key element [element ...]`
- `LPOP key`
- `RPOP key`
- `LRANGE key start stop`
- `LLEN key`

### Steps

1. **Choose internal list structure**
   - Start with a deque abstraction.
   - Support efficient push and pop at both ends.

2. **Implement list value type**
   - Store list as typed value in the engine.
   - Return wrong-type errors for non-list keys.

3. **Implement push commands**
   - Create list if key does not exist.
   - Add multiple elements in command order.
   - Return new list length.

4. **Implement pop commands**
   - Return null for missing key or empty list.
   - Delete key if list becomes empty if that is the chosen behavior.

5. **Implement range command**
   - Support negative indexes.
   - Clamp out-of-range indexes.
   - Return empty array for invalid ranges.

6. **Add tests**
   - Push/pop ordering.
   - Negative index ranges.
   - Wrong-type errors.
   - Empty list behavior.

### Exit Criteria

- List commands work through `redis-cli`.
- Edge cases match Redis behavior where practical.
- List implementation has unit tests and command tests.

---

## Phase 6: Hashes and Sets

### Objective

Add hash and set data types.

### Hash Commands

- `HSET key field value [field value ...]`
- `HGET key field`
- `HDEL key field [field ...]`
- `HGETALL key`
- `HLEN key`

### Set Commands

- `SADD key member [member ...]`
- `SREM key member [member ...]`
- `SMEMBERS key`
- `SISMEMBER key member`
- `SCARD key`

### Steps

1. **Implement hash structure**
   - Use nested map from field to value.
   - Track field count.
   - Return inserted-field count for `HSET`.

2. **Implement set structure**
   - Use map from member to empty struct.
   - Return added or removed member count.

3. **Add command handlers**
   - Validate arity carefully.
   - Create key if missing for mutating commands.
   - Return null or empty collections for missing keys as appropriate.

4. **Handle wrong types**
   - Existing string/list key should reject hash/set operations.

5. **Add tests**
   - Insert/update/delete behavior.
   - Missing key behavior.
   - Wrong-type behavior.
   - Multi-argument behavior.

### Exit Criteria

- Hash and set commands work through `redis-cli`.
- Data type behavior is isolated and tested.
- Command router remains clean and maintainable.

---

## Phase 7: Sorted Sets and Skip List

### Objective

Implement sorted sets backed by a skip list plus hash map.

### Commands

- `ZADD key score member [score member ...]`
- `ZREM key member [member ...]`
- `ZRANGE key start stop [WITHSCORES]`
- `ZSCORE key member`
- `ZRANK key member`

### Internal Design

Use two structures together:

- **Hash map:** member to score for O(1) score lookup.
- **Skip list:** ordered by score and member for O(log n) insert, delete, rank, and range traversal.

### Steps

1. **Implement skip list node**
   - Member.
   - Score.
   - Forward pointers by level.
   - Optional backward pointer for reverse traversal later.
   - Optional span values for rank support.

2. **Implement skip list core operations**
   - Insert.
   - Delete.
   - Find by rank.
   - Find rank by member and score.
   - Range traversal.

3. **Implement sorted set wrapper**
   - Keep skip list and dictionary in sync.
   - Updating an existing member should update its score.
   - Tie-break equal scores by member lexicographically.

4. **Implement sorted set commands**
   - Parse floating-point scores.
   - Return number of newly added members for `ZADD`.
   - Support `WITHSCORES` in `ZRANGE`.
   - Return null for missing `ZSCORE`.

5. **Add extensive tests**
   - Insertion order.
   - Score updates.
   - Duplicate members.
   - Equal-score tie-breaking.
   - Rank correctness.
   - Range boundaries.
   - Randomized tests comparing against a sorted slice model.

6. **Benchmark sorted set operations**
   - Insert N members.
   - Delete N members.
   - Range queries.
   - Rank lookup.

### Design Decisions to Document

- Why skip list is used instead of a balanced tree.
- How equal scores are ordered.
- Whether rank uses spans or a simpler traversal.
- Complexity of each operation.

### Exit Criteria

- Sorted set commands work through `redis-cli`.
- Skip list has standalone tests.
- Randomized correctness tests pass.
- Complexity and tradeoffs are documented.

---

## Phase 8: AOF Persistence

### Objective

Persist every write command to an append-only file and replay it on restart.

### Features

- AOF command logging.
- Startup replay.
- Configurable fsync policy.
- AOF rewrite/compaction.

### Steps

1. **Identify write commands**
   - Strings: `SET`, `DEL`, `INCR`, `DECR`, `MSET`.
   - Expiry: `EXPIRE`, `PEXPIRE`, `PERSIST`.
   - Lists, hashes, sets, sorted sets.
   - `FLUSHDB`.

2. **Append write commands after successful execution**
   - Only log commands that actually execute successfully.
   - Encode commands in RESP format for easy replay.

3. **Implement fsync policies**
   - `always`: fsync every write.
   - `everysec`: fsync from a background ticker.
   - `no`: let the OS decide.

4. **Implement replay**
   - On startup, read AOF from beginning.
   - Parse RESP commands.
   - Execute commands against the engine without re-appending them.
   - Fail clearly on corrupted input.

5. **Implement AOF rewrite**
   - Take a consistent snapshot of current state.
   - Generate minimal commands to reconstruct the state.
   - Write to a temporary file.
   - Atomically replace old AOF file.

6. **Add persistence tests**
   - Write data.
   - Stop server or recreate engine.
   - Replay AOF.
   - Verify exact state.
   - Test corrupted or partial AOF behavior.

### Design Decisions to Document

- Whether commands are logged before or after execution.
- What happens if append fails.
- Default fsync policy and why.
- How AOF rewrite avoids corrupting the existing file.

### Exit Criteria

- Data survives restart with AOF enabled.
- Replay does not duplicate writes into the AOF.
- AOF rewrite creates a smaller valid file.
- Fsync policies are tested or manually verified.

---

## Phase 9: RDB Snapshots

### Objective

Serialize the full dataset into a snapshot file for fast recovery.

### Steps

1. **Define snapshot format**
   - Include format version.
   - Include creation timestamp.
   - Include key count.
   - Include value type, key, value payload, and expiry metadata.

2. **Create consistent engine snapshot**
   - Avoid holding global locks for too long.
   - Copy references or values safely depending on mutation risks.
   - Ensure snapshot state is internally consistent.

3. **Write snapshot file**
   - Write to a temporary file first.
   - Flush and close file.
   - Atomically rename temp file to final path.

4. **Load snapshot on startup**
   - Validate header and version.
   - Decode all values.
   - Skip already-expired keys.
   - Restore expiry metadata.

5. **Add automatic save policy**
   - Example: save after 1000 writes in 60 seconds.
   - Make save policy configurable.

6. **Add snapshot tests**
   - Round-trip every data type.
   - Round-trip expiry metadata.
   - Corrupt snapshot detection.
   - Snapshot while writes are happening.

### Design Decisions to Document

- Snapshot file format.
- Consistency model during background snapshot.
- Startup precedence when both AOF and RDB exist.
- Why temporary file plus rename is used.

### Exit Criteria

- Snapshot can reconstruct full store state.
- Snapshot works for all supported data types.
- Snapshot save does not block normal operations for too long.
- Recovery behavior is documented.

---

## Phase 10: Pub/Sub

### Objective

Implement Redis-like channel-based publish/subscribe messaging.

### Commands

- `SUBSCRIBE channel [channel ...]`
- `UNSUBSCRIBE [channel ...]`
- `PUBLISH channel message`

### Steps

1. **Create pub/sub broker**
   - Map channel names to subscriber connections.
   - Protect subscriber maps with locks.
   - Support multiple subscribers per channel.

2. **Add subscriber connection mode**
   - After `SUBSCRIBE`, connection enters subscriber mode.
   - Only subscription-related commands should be accepted in this mode.

3. **Implement publish fan-out**
   - Send message to every subscriber of the channel.
   - Return number of subscribers that received the message.

4. **Handle disconnects**
   - Remove connection from all subscribed channels.
   - Avoid goroutine leaks.
   - Avoid blocking publisher forever on slow subscribers.

5. **Add tests**
   - One publisher, one subscriber.
   - Multiple subscribers.
   - Multiple channels.
   - Unsubscribe behavior.
   - Subscriber disconnect cleanup.

### Design Decisions to Document

- Whether fan-out is synchronous or asynchronous.
- How slow subscribers are handled.
- How connection lifecycle interacts with subscriptions.

### Exit Criteria

- `SUBSCRIBE` and `PUBLISH` work from separate `redis-cli` sessions.
- Disconnecting subscribers are cleaned up.
- Pub/sub does not block the entire server under normal load.

---

## Phase 11: LRU Eviction

### Objective

Evict keys when configured memory limit is reached.

### Features

- Configurable max memory.
- Approximate LRU policy.
- Optional comparison with exact LRU.

### Steps

1. **Track key access metadata**
   - Update last-access time or logical clock on reads and writes.
   - Keep metadata lightweight.

2. **Estimate memory usage**
   - Start with approximate size calculation.
   - Track key size, value size, and structural overhead roughly.
   - Document that this is an approximation.

3. **Implement eviction trigger**
   - After write commands, check memory usage.
   - If over limit, sample keys and evict the least recently used among the sample.
   - Continue until under limit or max eviction attempts reached.

4. **Benchmark eviction policies**
   - Exact LRU if implemented.
   - Sampled LRU.
   - Compare overhead and hit rate.

5. **Add tests**
   - Memory limit causes eviction.
   - Recently accessed keys are less likely to be evicted.
   - Expired keys are preferred for cleanup where reasonable.

### Design Decisions to Document

- Why approximate LRU is chosen.
- Sample size.
- Memory accounting limitations.
- Interaction between TTL expiry and eviction.

### Exit Criteria

- Server respects approximate memory limit.
- Eviction behavior is tested.
- Tradeoff between exact and sampled LRU is documented.

---

## Phase 12: Transactions

### Objective

Implement basic Redis-style transactions.

### Commands

- `MULTI`
- `EXEC`
- `DISCARD`
- `WATCH key [key ...]`

### Steps

1. **Add transaction state to connection**
   - Track whether connection is inside `MULTI`.
   - Queue commands instead of executing immediately.

2. **Implement command queueing**
   - Return `QUEUED` for queued commands.
   - Reject invalid commands at queue time where possible.

3. **Implement `EXEC`**
   - Execute queued commands atomically with respect to other commands.
   - Return array of command results.
   - Clear transaction state afterward.

4. **Implement `DISCARD`**
   - Clear queue.
   - Exit transaction mode.

5. **Implement optimistic locking with `WATCH`**
   - Track key versions in the engine.
   - Increment key version on writes.
   - If watched key changed before `EXEC`, return null transaction result.

6. **Add tests**
   - Basic transaction execution.
   - Discard behavior.
   - Invalid command behavior.
   - Watched key modification aborts transaction.
   - Concurrent transaction scenarios.

### Design Decisions to Document

- How atomicity is enforced.
- How key versions are tracked.
- Limitations compared to Redis transactions.

### Exit Criteria

- `MULTI`, `EXEC`, and `DISCARD` work through `redis-cli`.
- `WATCH` detects conflicting writes.
- Transaction semantics are tested and documented.

---

## Phase 13: Leader-Follower Replication

### Objective

Implement leader-follower replication with initial sync, live command streaming, and offset tracking.

### Features

- Leader accepts writes and reads.
- Follower connects to leader.
- Follower receives initial snapshot.
- Follower receives live write stream after sync.
- Follower is read-only for normal clients.
- Manual promotion is possible.

### Steps

1. **Define replication protocol**
   - Follower sends handshake to leader.
   - Leader responds with metadata and current replication offset.
   - Leader sends snapshot for initial sync.
   - Leader streams write commands after snapshot.

2. **Track replication offset**
   - Increment offset for each replicated write command or byte range.
   - Persist follower offset if practical.
   - Expose offset through `INFO`.

3. **Implement leader replication log**
   - Every successful write command is sent to connected followers.
   - Keep enough recent commands for catch-up after reconnect.

4. **Implement follower initial sync**
   - Connect to leader.
   - Request full sync.
   - Load snapshot into local engine.
   - Start applying live stream.

5. **Implement follower read-only mode**
   - Allow reads from clients.
   - Reject writes unless promoted.

6. **Implement reconnect behavior**
   - On connection loss, follower reconnects.
   - If leader still has required offset, perform partial catch-up.
   - Otherwise perform full resync.

7. **Implement manual promotion**
   - Add config or command for promoting follower to leader.
   - Stop applying leader stream.
   - Start accepting writes.

8. **Add tests**
   - Initial sync copies full state.
   - New writes appear on follower.
   - Follower rejects writes.
   - Reconnect catches up.
   - Full resync works when offset is unavailable.

### Design Decisions to Document

- Full sync versus partial sync behavior.
- Offset definition.
- Replication consistency guarantees.
- What happens during leader failure.
- Limitations versus Redis replication.

### Exit Criteria

- Two-server leader-follower demo works locally.
- Follower converges to leader after writes.
- Replication offset is visible in `INFO`.
- Replication limitations are documented clearly.

---

## Phase 14: Benchmarking, Profiling, and Optimization

### Objective

Measure the system honestly, identify bottlenecks, and optimize the highest-impact areas.

### Benchmark Targets

- `PING`
- `GET`
- `SET`
- `MGET`
- `MSET`
- Pipelined `GET`/`SET`
- Mixed read/write workload
- Expiry-heavy workload
- Sorted set workload
- Persistence-enabled workload

### Metrics

- Operations per second.
- p50 latency.
- p95 latency.
- p99 latency.
- Memory usage.
- CPU profile hotspots.
- Allocation count per command.
- Restart time from AOF and RDB.
- Replication lag.

### Steps

1. **Create benchmark scripts**
   - Include commands for running `redis-benchmark` against this server.
   - Include commands for running the same benchmark against Redis.
   - Keep environment details in benchmark notes.

2. **Run baseline benchmarks**
   - Benchmark with persistence disabled.
   - Benchmark with AOF everysec.
   - Benchmark with pipelining.

3. **Capture profiles**
   - CPU profile during heavy GET/SET.
   - Memory profile during large dataset load.
   - Goroutine profile during many connections.

4. **Optimize protocol path**
   - Reduce allocations in parser.
   - Reuse buffers where safe.
   - Avoid unnecessary string conversions.

5. **Optimize engine path**
   - Evaluate lock contention.
   - Consider sharded maps.
   - Benchmark lock granularity.

6. **Optimize persistence path**
   - Batch writes where safe.
   - Evaluate fsync cost.
   - Measure AOF rewrite impact.

7. **Document results**
   - Include benchmark tables.
   - Compare against Redis.
   - Explain why Redis is faster.
   - Explain where this implementation performs well.

### Exit Criteria

- Benchmark results are reproducible.
- At least one optimization is justified with before/after data.
- README includes honest performance claims.
- Target of 100K+ ops/sec for simple GET/SET is attempted and documented.

---

## Phase 15: Documentation, Packaging, and Final Polish

### Objective

Make the project understandable, runnable, and presentable.

### Steps

1. **Write README**
   - Project overview.
   - Feature list.
   - Supported commands.
   - Quick start.
   - Configuration.
   - Benchmark summary.
   - Architecture diagram.
   - Limitations.

2. **Write architecture documentation**
   - Connection lifecycle.
   - Protocol parser.
   - Command router.
   - Engine data model.
   - Persistence flow.
   - Replication flow.
   - Background workers.

3. **Write persistence documentation**
   - AOF behavior.
   - RDB behavior.
   - Startup recovery order.
   - Failure scenarios.

4. **Write replication documentation**
   - Leader/follower setup.
   - Initial sync.
   - Live stream.
   - Offset tracking.
   - Reconnect behavior.
   - Manual promotion.

5. **Create Dockerfile**
   - Multi-stage build.
   - Minimal runtime image.
   - Expose configured port.

6. **Create docker-compose setup**
   - Single-node server.
   - Optional leader/follower pair.
   - Optional Redis comparison service.

7. **Finalize CI**
   - Run tests.
   - Run lint.
   - Optionally run lightweight benchmarks.

8. **Prepare resume-ready summary**
   - Include supported features.
   - Include benchmark numbers.
   - Include strongest technical details.

### Exit Criteria

- Fresh clone can run the server using README instructions.
- Docker build works.
- Tests pass in CI.
- Documentation explains architecture and tradeoffs.
- Project is ready to show in interviews.

---

## Testing Strategy

### Unit Tests

Use unit tests for:

- RESP parsing and encoding.
- Command argument validation.
- Engine operations.
- Expiry calculations.
- Data structures.
- Skip list correctness.
- Persistence encoders and decoders.

### Integration Tests

Use integration tests for:

- TCP server request/response behavior.
- Redis CLI compatibility for supported commands.
- Persistence restart flows.
- Pub/sub with multiple connections.
- Transactions with concurrent clients.
- Replication between two server instances.

### Property and Randomized Tests

Use randomized tests for:

- AOF replay equivalence.
- RDB round-trip equivalence.
- Skip list ordering compared against a sorted slice.
- Random command sequence replay.

### Benchmark Tests

Use benchmarks for:

- Parser throughput.
- Engine GET/SET throughput.
- Lock contention.
- Expiry worker overhead.
- Skip list insertion and range queries.
- AOF append throughput.

---

## Suggested Command Compatibility Matrix

| Area | Commands | Priority |
| --- | --- | --- |
| Server | `PING`, `ECHO`, `INFO`, `DBSIZE`, `FLUSHDB` | Must have |
| Strings | `GET`, `SET`, `DEL`, `EXISTS`, `INCR`, `DECR`, `MGET`, `MSET` | Must have |
| Expiry | `EXPIRE`, `TTL`, `PEXPIRE`, `PTTL`, `PERSIST` | Must have |
| Lists | `LPUSH`, `RPUSH`, `LPOP`, `RPOP`, `LRANGE`, `LLEN` | Must have |
| Hashes | `HSET`, `HGET`, `HDEL`, `HGETALL`, `HLEN` | Must have |
| Sets | `SADD`, `SREM`, `SMEMBERS`, `SISMEMBER`, `SCARD` | Must have |
| Sorted sets | `ZADD`, `ZREM`, `ZRANGE`, `ZSCORE`, `ZRANK` | Must have |
| Pub/Sub | `SUBSCRIBE`, `PUBLISH`, `UNSUBSCRIBE` | Should have |
| Transactions | `MULTI`, `EXEC`, `DISCARD`, `WATCH` | Should have |
| Replication | Internal sync commands or custom protocol | Should have |

---

## Definition of Done

The project is complete when:

- The server starts from a fresh clone.
- `redis-cli` can execute all supported commands.
- Core data structures are implemented from scratch.
- TTL expiry works through lazy and active cleanup.
- AOF and RDB persistence can recover data after restart.
- Pub/sub works across multiple clients.
- Pipelining works and improves throughput.
- LRU eviction works under a memory limit.
- Transactions work for the supported subset.
- Leader-follower replication works in a local demo.
- Benchmarks compare this server against Redis.
- Documentation explains architecture, tradeoffs, limitations, and performance.
- Tests cover the critical correctness paths.

---

## Final Interview Story

When explaining the project, structure the story like this:

1. **Problem:** Redis is an in-memory data store with protocol compatibility, persistence, and replication.
2. **Core implementation:** Built TCP server, RESP parser, command router, and in-memory typed engine in Go.
3. **Data structures:** Implemented strings, lists, hashes, sets, and skip-list-backed sorted sets.
4. **Correctness:** Added TTL expiry, AOF replay, RDB snapshots, and restart verification tests.
5. **Distributed systems:** Added leader-follower replication with snapshot sync, live write streaming, and offset tracking.
6. **Performance:** Benchmarked with `redis-benchmark`, profiled with `pprof`, optimized parser and locking paths, and documented tradeoffs against real Redis.
7. **Reflection:** Clearly state what is production-like, what is simplified, and what you would improve next.
