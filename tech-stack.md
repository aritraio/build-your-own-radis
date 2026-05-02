# Tech Stack Guide: Distributed Key-Value Store

## Purpose of This File

This file explains what technology stack you should use for this project and how every piece fits together.

You are building a Redis-like system from scratch. That means the main goal is not to use many frameworks. The main goal is to understand networking, memory, concurrency, persistence, and distributed systems deeply.

Because of that, the recommended stack is intentionally simple.

---

## Recommended Stack Summary

| Area | Recommended Tool | Why |
| --- | --- | --- |
| Programming language | Go 1.22+ | Excellent for networking, concurrency, backend systems, and infrastructure projects |
| Networking | Go standard library `net` | Lets you build the TCP server from scratch |
| Buffered I/O | Go standard library `bufio` | Efficient reading/writing over TCP connections |
| Concurrency | Goroutines, channels, mutexes, RWMutex | Core Go primitives used in real backend systems |
| Storage engine | Custom in-memory engine | The point of the project is to build the database internals yourself |
| Data structures | Custom implementations | Lists, sets, hashes, skip lists should be implemented manually |
| Protocol | RESP | Redis Serialization Protocol used by Redis clients and `redis-cli` |
| Testing | Go `testing` package | Built-in, reliable, simple |
| Assertions | `testify` | Cleaner test assertions for a beginner |
| Benchmarking | Go benchmarks + `redis-benchmark` | Measures internal and external performance |
| Profiling | `pprof` | Standard Go profiler for CPU, memory, and goroutines |
| Linting | `golangci-lint` | Catches common Go mistakes |
| Persistence | Custom AOF and RDB-like files | Learn how durable in-memory databases work |
| Replication | Custom TCP-based leader-follower protocol | Learn distributed systems fundamentals |
| Containerization | Docker | Makes the server easy to run anywhere |
| CI | GitHub Actions | Runs tests automatically on every push |
| Documentation | Markdown | Best format for architecture, tradeoffs, and interview notes |

---

## Core Technology: Go

### Why Go?

Go is the best language choice for this project because it is widely used in infrastructure and backend systems.

You will learn:

- How TCP servers work.
- How goroutines handle concurrency.
- How locks protect shared memory.
- How to write fast parsers.
- How to benchmark and profile real systems.
- How to build a long-running server process.

Companies use Go for systems like:

- Kubernetes.
- Docker.
- Prometheus.
- etcd.
- CockroachDB components.
- Cloudflare infrastructure.

For a second-year CS student, this project will make your backend/systems profile much stronger.

### Go Version

Use:

```text
Go 1.22 or newer
```

You do not need advanced Go features at the beginning. Focus on:

- Structs.
- Interfaces.
- Errors.
- Goroutines.
- Channels.
- Mutexes.
- Testing.
- Benchmarks.

---

## What Not to Use

This project should avoid heavy frameworks and existing storage engines.

Do not use:

- Web frameworks like Gin, Fiber, Echo, or Chi for the database server.
- Existing key-value stores like BadgerDB, BoltDB, Pebble, RocksDB, or LevelDB.
- Existing Redis server libraries.
- Existing skip list libraries.
- Existing RESP server frameworks.

You may use small helper libraries for testing, but not for the core engine.

The learning comes from building the internals yourself.

---

## System Architecture and How the Stack Fits Together

The system will roughly work like this:

```text
redis-cli / Redis client
        |
        v
TCP connection using Go net package
        |
        v
Buffered reader/writer using bufio
        |
        v
RESP parser
        |
        v
Command router
        |
        v
In-memory engine
        |
        +------------------+
        |                  |
        v                  v
Persistence layer      Replication layer
AOF + RDB              Leader -> Follower
```

Each layer has a clear responsibility.

---

## Layer 1: TCP Server

### What to Use

Use Go standard library packages:

```text
net
bufio
context
sync
os/signal
```

### What It Does

The TCP server listens on a port, accepts client connections, and handles each connected client.

Example client tools:

- `redis-cli`
- A Go Redis client
- A Python Redis client
- A raw TCP client like `nc`

### How It Works

1. Server opens a TCP listener.
2. Client connects to the server.
3. Server creates a goroutine for that connection.
4. Goroutine reads RESP commands from the client.
5. Server executes commands.
6. Server writes RESP responses back.

### Why This Matters

This teaches real backend networking:

- Sockets.
- Long-lived connections.
- Concurrent clients.
- Buffered I/O.
- Graceful shutdown.
- Handling disconnects.

---

## Layer 2: RESP Protocol

### What to Use

Build your own parser and writer using:

```text
bufio.Reader
bufio.Writer
[]byte
strings
strconv
```

### What RESP Is

RESP means Redis Serialization Protocol.

It is the protocol Redis uses to communicate with clients.

A command like this:

```text
SET name aritra
```

is sent over the network like this:

```text
*3\r\n$3\r\nSET\r\n$4\r\nname\r\n$6\r\naritra\r\n
```

Your parser must convert that raw network data into a command object.

### What You Should Implement

You should support:

- Simple strings.
- Errors.
- Integers.
- Bulk strings.
- Arrays.
- Null bulk strings.

### Why This Matters

Protocol parsing is a major systems skill. It teaches:

- Byte-level thinking.
- Parsing partial data.
- Error handling.
- Client compatibility.
- Performance optimization.

---

## Layer 3: Command Router

### What to Use

Use normal Go code:

```text
map[string]HandlerFunc
interfaces
structs
errors
```

### What It Does

The router receives parsed commands and calls the correct handler.

Examples:

```text
GET name       -> string command handler
LPUSH q item   -> list command handler
ZADD rank 10 a -> sorted set command handler
PING           -> server command handler
```

### Recommended Design

Keep command handling separate from protocol parsing.

Good separation:

```text
RESP parser -> Command -> Router -> Engine -> Response -> RESP writer
```

Bad separation:

```text
RESP parser directly modifies storage
```

### Why This Matters

This makes the project easier to test and extend.

You can test command behavior without always opening TCP connections.

---

## Layer 4: In-Memory Engine

### What to Use

Build this yourself with Go data structures:

```text
map[string]*Value
sync.RWMutex
sync.Mutex
atomic counters
```

Later, you can use sharded maps:

```text
[]Shard
hash(key) % shardCount
```

### What It Stores

Every key should map to a typed value.

A value should know:

- Its type.
- Its actual data.
- Its expiry time if any.
- Its last access time for LRU.
- Its version for `WATCH` transactions.

Possible internal value types:

```text
String
List
Hash
Set
SortedSet
```

### Why This Matters

The engine is the heart of the database.

You will learn:

- Memory layout tradeoffs.
- Locking strategy.
- Type safety.
- Expiry handling.
- Atomic command behavior.

---

## Layer 5: Data Structures

### What to Use

Implement data structures manually.

| Redis Type | Internal Implementation |
| --- | --- |
| String | Go string or byte slice |
| List | Deque or doubly linked list |
| Hash | Nested map |
| Set | Map from member to empty struct |
| Sorted set | Skip list plus hash map |

### Most Important Data Structure: Skip List

Sorted sets are the most impressive part of the project.

A sorted set needs:

- Fast insert.
- Fast delete.
- Fast rank lookup.
- Fast range query.
- Ordered traversal by score.

Redis uses a skip list for sorted sets. You should also implement one.

### Why This Matters

This connects your data structures course directly to a real database system.

You will use:

- Hash maps.
- Linked lists.
- Probabilistic data structures.
- Sorting rules.
- Big-O analysis.

---

## Layer 6: TTL Expiry Engine

### What to Use

Use:

```text
time.Time
time.Duration
goroutines
tickers
mutexes
```

### How Expiry Works

Use two expiry strategies together.

### Lazy Expiry

Check if a key is expired when someone accesses it.

Example:

```text
GET session:123
```

If `session:123` is expired, delete it and return null.

### Active Expiry

Run a background goroutine that samples keys and deletes expired ones.

Do not scan all keys every second. That is too expensive.

A Redis-like approach:

1. Sample a small number of keys.
2. Delete expired keys.
3. If many sampled keys were expired, sample again.
4. Sleep briefly and repeat.

### Why This Matters

This teaches background workers and performance-aware cleanup.

---

## Layer 7: Persistence

Persistence means data survives server restart.

You should implement two persistence modes.

---

### AOF: Append-Only File

### What to Use

Use:

```text
os.File
bufio.Writer
sync.Mutex
time.Ticker
```

### How AOF Works

Every successful write command is appended to a log file.

Example:

```text
SET name aritra
INCR visits
LPUSH jobs email
```

On restart, replay the file from top to bottom.

### Fsync Policies

Support three policies:

| Policy | Meaning | Tradeoff |
| --- | --- | --- |
| `always` | Flush to disk after every write | Safest but slowest |
| `everysec` | Flush once per second | Good default |
| `no` | Let OS decide | Fastest but least durable |

### Why AOF Matters

AOF teaches:

- Durability.
- Write-ahead logging style systems.
- Recovery after crash.
- Disk I/O cost.
- Log compaction.

---

### RDB: Snapshot File

### What to Use

Use:

```text
encoding/binary
encoding/gob or custom binary format
os.File
atomic rename
```

### How RDB Works

Periodically serialize the full in-memory dataset to a snapshot file.

On restart, load the snapshot.

### AOF vs RDB

| Feature | AOF | RDB |
| --- | --- | --- |
| Stores | Every write command | Full dataset snapshot |
| Recovery | Replay commands | Load state directly |
| File size | Can grow large | Usually compact |
| Durability | Better | Depends on snapshot frequency |
| Restart speed | Slower for huge logs | Faster |

### Why RDB Matters

RDB teaches:

- Serialization.
- Consistent snapshots.
- Atomic file replacement.
- Recovery design.

---

## Layer 8: Pub/Sub

### What to Use

Use:

```text
channels
mutexes
maps
connection state
```

### How It Works

Clients subscribe to channels.

Other clients publish messages to channels.

The server forwards each message to all subscribers.

Example:

```text
Client A: SUBSCRIBE news
Client B: PUBLISH news hello
Client A receives: hello
```

### Important Design Question

What happens if a subscriber is slow?

You need to avoid blocking the whole server because one client is not reading messages.

Possible choices:

- Buffered message channels.
- Drop messages for slow subscribers.
- Disconnect very slow subscribers.

Document your choice.

---

## Layer 9: Pipelining

### What to Use

Use your existing RESP parser and buffered writer.

### How It Works

A client can send multiple commands without waiting for each response.

Example:

```text
SET a 1
SET b 2
GET a
GET b
```

The server reads and executes them in order, then sends responses in order.

### Why This Matters

Pipelining reduces network round trips and greatly improves throughput.

This is one of the easiest features to benchmark clearly.

---

## Layer 10: LRU Eviction

### What to Use

Use:

```text
logical clock or timestamp
sampled key selection
memory accounting
```

### How It Works

When memory usage exceeds a configured limit, evict old keys.

Recommended approach:

1. Track last access time for each key.
2. When memory is over limit, sample a few keys.
3. Evict the least recently used key among the sample.
4. Repeat until memory is under limit.

### Why Approximate LRU?

Exact LRU is expensive because every read/write must update a global ordered structure.

Approximate LRU is faster and closer to Redis's practical design.

---

## Layer 11: Transactions

### What to Use

Use:

```text
per-connection state
command queues
engine locks
key version numbers
```

### How It Works

A transaction queues commands and executes them together.

Example:

```text
MULTI
SET a 1
INCR counter
EXEC
```

For `WATCH`, track key versions.

If a watched key changes before `EXEC`, abort the transaction.

### Why This Matters

Transactions teach:

- Atomicity.
- Optimistic locking.
- Per-connection state.
- Concurrency edge cases.

---

## Layer 12: Replication

### What to Use

Use:

```text
net.Conn
custom replication protocol
RDB snapshot reuse
AOF-style command stream
replication offsets
reconnect loop
```

### How It Works

A follower connects to a leader.

Initial sync:

1. Follower connects to leader.
2. Leader sends a snapshot.
3. Follower loads the snapshot.

Live sync:

1. Leader records every successful write.
2. Leader streams writes to follower.
3. Follower applies writes in the same order.
4. Follower tracks replication offset.

### Why This Matters

This is where the project becomes distributed systems work.

You will learn:

- Leader-follower architecture.
- Replication lag.
- Reconnection.
- Partial sync vs full sync.
- Read-only followers.
- Manual promotion.

---

## Layer 13: Testing Stack

### Built-in Testing

Use:

```text
testing
```

For most tests, the standard library is enough.

### Testify

Use:

```text
github.com/stretchr/testify
```

This gives cleaner assertions:

```text
require.NoError(t, err)
require.Equal(t, expected, actual)
```

### Test Types

| Test Type | Purpose |
| --- | --- |
| Unit tests | Test parser, engine, data structures |
| Integration tests | Test TCP server with real connections |
| Compatibility tests | Test using `redis-cli` or Redis clients |
| Persistence tests | Restart and verify data survives |
| Replication tests | Start leader and follower together |
| Randomized tests | Compare implementation against simple reference model |

### Beginner Advice

Do not wait until the end to write tests. This project has many edge cases, and tests will save you time.

---

## Layer 14: Benchmarking Stack

### Go Benchmarks

Use Go's built-in benchmark support:

```text
go test -bench=. ./...
```

Use this for:

- Parser benchmarks.
- Engine benchmarks.
- Skip list benchmarks.
- Expiry benchmarks.

### redis-benchmark

Use official Redis benchmarking tool:

```text
redis-benchmark
```

Use this for external server benchmarks:

- GET throughput.
- SET throughput.
- Pipelining throughput.
- Latency numbers.

### What to Measure

Measure:

- Operations per second.
- p50 latency.
- p95 latency.
- p99 latency.
- Memory usage.
- Allocation count.
- CPU hotspots.

### Important Note

Your implementation will probably be slower than Redis. That is expected.

The important part is explaining why.

---

## Layer 15: Profiling Stack

### What to Use

Use Go's profiler:

```text
pprof
```

### Profile Types

| Profile | What It Shows |
| --- | --- |
| CPU profile | Where the server spends CPU time |
| Memory profile | Where allocations happen |
| Goroutine profile | How many goroutines exist and where they are blocked |
| Mutex profile | Lock contention |
| Block profile | Blocking operations |

### Why Profiling Matters

Do not guess bottlenecks.

Measure first, then optimize.

Example optimization path:

1. Benchmark `SET` throughput.
2. Capture CPU profile.
3. See parser is allocating too much.
4. Reduce string conversions.
5. Benchmark again.
6. Document improvement.

---

## Layer 16: Docker

### What to Use

Use Docker with a multi-stage build.

### Why Docker?

Docker makes it easy for someone else to run your project.

They should be able to do something like:

```text
docker build -t radis .
docker run -p 6379:6379 radis
```

Then connect using:

```text
redis-cli -p 6379
```

### What Docker Should Include

- Build stage using official Go image.
- Runtime stage using a minimal base image.
- Expose server port.
- Mount data directory for persistence.

---

## Layer 17: GitHub Actions CI

### What to Use

Use GitHub Actions.

### What CI Should Run

At minimum:

```text
go test ./...
go vet ./...
```

Later:

```text
golangci-lint run
```

Optional:

```text
go test -race ./...
```

### Why CI Matters

CI proves that the project builds and tests pass on a clean machine.

This makes the project more credible.

---

## Recommended Dependency Policy

Keep dependencies minimal.

### Allowed Dependencies

Good dependencies:

- `github.com/stretchr/testify` for tests.
- `golangci-lint` as a development tool.

### Avoid Dependencies For

Avoid external libraries for:

- TCP server.
- RESP parser.
- Storage engine.
- Skip list.
- Persistence format.
- Replication protocol.

### Reason

If you outsource the hard parts, the project becomes less impressive and less educational.

---

## Beginner-Friendly Learning Order

If you are new to systems, learn the stack in this order:

1. **Go basics**
   - Structs, interfaces, errors, modules.

2. **Go testing**
   - Unit tests, table-driven tests, benchmarks.

3. **TCP networking**
   - `net.Listen`, `net.Conn`, client/server model.

4. **RESP protocol**
   - Learn how Redis clients encode commands.

5. **Maps and locks**
   - Build the string key-value engine.

6. **TTL expiry**
   - Learn background goroutines and time-based deletion.

7. **Data structures**
   - Lists, hashes, sets, skip lists.

8. **Persistence**
   - AOF first, then RDB snapshots.

9. **Benchmarking and profiling**
   - Learn how to measure performance.

10. **Replication**
   - Add distributed systems concepts last.

---

## Final Recommended Tech Stack

Use this exact stack unless you have a strong reason to change it:

```text
Language: Go 1.22+
Network: net, bufio
Concurrency: goroutines, channels, sync.Mutex, sync.RWMutex
Protocol: RESP2
Storage: custom in-memory engine
Data structures: custom list, hash, set, skip list
Persistence: custom AOF + custom RDB-like snapshot
Testing: testing + testify
Benchmarking: go test -bench + redis-benchmark
Profiling: pprof
Linting: golangci-lint
Container: Docker
CI: GitHub Actions
Docs: Markdown
```

This stack is simple, realistic, and strong for interviews.

---

## What You Will Be Able to Explain After This Project

After completing this project, you should be able to explain:

- How Redis clients communicate with Redis servers.
- How a TCP server handles multiple clients.
- How an in-memory key-value engine is structured.
- How TTL expiry works without scanning every key constantly.
- How sorted sets use skip lists.
- How append-only persistence works.
- How snapshot persistence works.
- How pub/sub fan-out works.
- How pipelining improves throughput.
- How LRU eviction trades accuracy for speed.
- How transactions use optimistic locking.
- How leader-follower replication works.
- Why Redis is faster than your implementation.
- What tradeoffs Go introduces compared to C.

That is the real value of the tech stack.
