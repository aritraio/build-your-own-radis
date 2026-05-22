# Distributed Key-Value Store — Build Your Own Redis

> **Track:** Systems Engineering · Backend · Distributed Systems
> **Timeline:** 5–6 weeks
> **Language:** Go
> **Why this exists:** This is the single most respected systems project you can build. It covers TCP networking, protocol design, concurrency, persistence, and benchmarking — the exact skills that get you hired at Google, Amazon, Stripe, Cloudflare, or any serious product company. It also proves you're not just a Python/AI developer, which is the biggest gap in your portfolio right now.

---

## 1. One-line Summary

A Redis-compatible, in-memory key-value store written from scratch in Go, supporting the RESP wire protocol, core data structures (strings, lists, hashes, sorted sets), TTL-based expiry, persistence (AOF + snapshots), pub/sub, pipelining, and leader-follower replication — with benchmarks proving it can handle 100K+ ops/sec on a single machine.

## 2. Why This Project Wins Interviews

Half of all systems design interviews boil down to some variant of:

- "Design a distributed cache."
- "How does Redis work under the hood?"
- "How would you implement TTL expiry efficiently?"
- "Explain leader-follower replication."
- "How do you persist in-memory state durably?"
- "What's the tradeoff between AOF and RDB snapshots?"

If you've **built** one, you don't rehearse answers — you just describe what you did. That's an unfair advantage.

This project also signals:

- **Go proficiency** — the lingua franca of infrastructure. Proves you're not a one-language developer.
- **Networking** — you wrote a TCP server from scratch. You understand connections, buffers, and protocol parsing.
- **Concurrency** — goroutines, mutexes, channels, lock-free data structures. This is what Go interviews test.
- **Performance engineering** — you benchmarked, profiled, and optimized. You can talk about p99 latency, throughput, and memory footprint with real data.

## 3. Scope

### Phase 1: Core KV Engine (Weeks 1–2)

Build the in-memory engine and wire protocol.

- **TCP server** accepting concurrent client connections (goroutine-per-connection or event loop).
- **RESP protocol parser** — Redis Serialization Protocol. Your store should be usable with `redis-cli` and any Redis client library.
- **Core commands:**

| Category | Commands |
|----------|----------|
| Strings | `GET`, `SET`, `DEL`, `EXISTS`, `INCR`, `DECR`, `MGET`, `MSET` |
| Expiry | `EXPIRE`, `TTL`, `PEXPIRE`, `PTTL`, `PERSIST` |
| Server | `PING`, `ECHO`, `INFO`, `DBSIZE`, `FLUSHDB` |

- **TTL engine:** Lazy expiry (check on access) + active expiry (background goroutine that samples and evicts expired keys). Explain the tradeoff in your README.
- **Memory-efficient key storage:** Use a hash map with open addressing or Go's `sync.Map` for concurrent access. Benchmark both.

### Phase 2: Data Structures (Week 3)

Extend beyond strings. This is what separates your project from a toy.

| Category | Commands | Internal Structure |
|----------|----------|--------------------|
| Lists | `LPUSH`, `RPUSH`, `LPOP`, `RPOP`, `LRANGE`, `LLEN` | Doubly-linked list or deque |
| Hashes | `HSET`, `HGET`, `HDEL`, `HGETALL`, `HLEN` | Nested hash map |
| Sets | `SADD`, `SREM`, `SMEMBERS`, `SISMEMBER`, `SCARD` | Hash set |
| Sorted Sets | `ZADD`, `ZREM`, `ZRANGE`, `ZSCORE`, `ZRANK` | Skip list + hash map (this is how Redis does it) |

> [!IMPORTANT]
> **The sorted set is the hard part.** Implementing a skip list from scratch is genuine data structures work. This is what makes interviewers lean forward. If you skip this, you skip the best part of the project.

### Phase 3: Persistence (Week 4)

An in-memory store without persistence is a toy. Add two persistence modes:

- **AOF (Append-Only File):** Log every write command to disk. On restart, replay the log. Implement `fsync` policies: `always` (durable but slow), `everysec` (good tradeoff), `no` (fast but risky). Implement AOF rewriting (compact the log by snapshotting current state).
- **RDB Snapshots:** Periodically serialize the entire dataset to a binary file using `fork()`-style COW (or Go's equivalent — serialize in a background goroutine with a consistent read snapshot). Configurable intervals: e.g., "save after 1000 writes in 60 seconds."
- **Hybrid:** AOF for durability, RDB for fast restart. Document the tradeoffs.

### Phase 4: Advanced Features (Week 5)

- **Pub/Sub:** `SUBSCRIBE`, `PUBLISH`, `UNSUBSCRIBE`. Channel-based message fan-out to all subscribers. Use Go channels internally.
- **Pipelining:** Batch multiple commands in a single round-trip. Parse and execute them in order, return all responses at once. Massive throughput improvement — benchmark it.
- **LRU Eviction:** When memory limit is reached, evict least-recently-used keys. Implement an approximated LRU (sampled eviction, like Redis does) and benchmark against exact LRU.
- **Transactions:** `MULTI`, `EXEC`, `DISCARD`. Queue commands and execute atomically. `WATCH` for optimistic locking.

### Phase 5: Replication & Polish (Week 6)

- **Leader-Follower Replication:**
  - Follower connects to the leader via TCP.
  - Leader sends an RDB snapshot for initial sync.
  - After sync, leader streams write commands to the follower in real-time (replication log).
  - Follower is read-only. On leader failure, a follower can be manually promoted.
  - Track replication offset for consistency verification.
- **Benchmarking:**
  - Use `redis-benchmark` (the real one) against your store and compare against actual Redis.
  - Measure: ops/sec for GET/SET, p50/p99 latency, pipelining throughput, memory footprint.
  - Document results honestly. You will be slower than Redis — explain exactly why (C vs Go, jemalloc vs Go GC, io_uring vs goroutines).
  - Target: **100K+ ops/sec** for simple GET/SET on a modern machine. This is achievable in Go.
- **README & Architecture Doc:**
  - Architecture diagram showing: client connections → protocol parser → command router → engine → persistence → replication.
  - Explain every design decision: why skip list for sorted sets, why approximated LRU, why fsync-everysec is the default.

## 4. Architecture

```
                   ┌──────────────────────────────┐
  redis-cli ──────▶│       TCP Listener            │
  any Redis   ────▶│   (goroutine per connection)  │
  client lib  ────▶│                                │
                   └──────────────┬─────────────────┘
                                  │
                                  ▼
                   ┌──────────────────────────────┐
                   │     RESP Protocol Parser      │
                   │  (deserialize → Command obj)  │
                   └──────────────┬─────────────────┘
                                  │
                                  ▼
                   ┌──────────────────────────────┐
                   │     Command Router            │
                   │  GET → StringEngine           │
                   │  ZADD → SortedSetEngine       │
                   │  SUBSCRIBE → PubSubEngine     │
                   └──────────────┬─────────────────┘
                                  │
                   ┌──────────────┴──────────────────────┐
                   │                                      │
                   ▼                                      ▼
        ┌────────────────────┐              ┌────────────────────┐
        │   KV Engine        │              │   Persistence      │
        │ (in-memory store)  │              │   AOF + RDB        │
        │ HashMap + SkipList │              │   Background I/O   │
        │ + LinkedList + Set │              │                    │
        └────────────────────┘              └────────────────────┘
                   │
                   ▼
        ┌────────────────────┐
        │   Replication      │
        │   Leader → Follower│
        │   (TCP stream)     │
        └────────────────────┘

        ┌────────────────────────────────────────────────┐
        │  Background Workers:                           │
        │  - TTL expiry (active sampling)                │
        │  - AOF rewrite (compaction)                    │
        │  - RDB snapshot (periodic)                     │
        │  - LRU eviction (when memory limit reached)    │
        └────────────────────────────────────────────────┘
```

## 5. Tech Stack

- **Language:** Go 1.22+. No frameworks. Standard library only for networking (`net`, `bufio`, `sync`). This is the point.
- **Testing:** Go's built-in `testing` package + `testify` for assertions.
- **Benchmarking:** `redis-benchmark` (official Redis tool) for comparative benchmarks. Go's `testing.B` for micro-benchmarks.
- **Profiling:** `pprof` for CPU and memory profiling. Document your optimization journey.
- **CI:** GitHub Actions for build, test, lint (`golangci-lint`), and benchmark regression detection.
- **Container:** Single Dockerfile. Multi-stage build for minimal image size.

## 6. The Hard Parts (This Is Where You Learn)

1. **RESP protocol parsing.** Write a zero-allocation parser. `redis-cli` must work unmodified against your server. This is your "protocol engineering" story.

2. **Concurrent access without killing throughput.** Redis is single-threaded. You're writing Go — you have goroutines. Design your locking strategy: per-key locking? Shard-level locking? Single command loop with channels? Each has tradeoffs. Document yours with benchmark data.

3. **Skip list implementation.** This is a real data structures exercise. Implement it from scratch — no libraries. Support O(log n) insert, delete, and range queries. This is interview gold.

4. **Persistence correctness.** AOF replay must produce an identical store. Write a property test: random command sequence → persist → restart → verify every key. RDB must be a consistent snapshot even while writes continue.

5. **TTL expiry that doesn't burn CPU.** Naive approach: scan all keys. Redis approach: sample 20 random keys, delete expired ones, repeat if >25% were expired. Implement the sampling approach and show the CPU difference.

6. **Replication consistency.** When the leader sends writes to the follower, what happens if the connection drops mid-stream? Implement reconnection with offset-based catch-up. Show a test where you kill the connection, reconnect, and verify the follower converges.

7. **Honest benchmarking.** Run `redis-benchmark` against both your store and real Redis. Your store will be slower. **That's fine.** Explain exactly why: Go's GC pauses, goroutine scheduling overhead, lack of io_uring, no jemalloc. This honesty is what separates you from someone who just claims "I built Redis."

## 7. Weekly Plan

| Week | Deliverable |
| --- | --- |
| 1 | TCP server, RESP parser, string commands (GET/SET/DEL/EXISTS/INCR), `redis-cli` compatibility verified |
| 2 | TTL engine (lazy + active expiry), MGET/MSET, pipelining, initial benchmarks |
| 3 | Lists, Hashes, Sets, Sorted Sets (skip list from scratch), command coverage tests |
| 4 | AOF persistence + RDB snapshots, restart-and-verify tests, pub/sub |
| 5 | LRU eviction, transactions (MULTI/EXEC), leader-follower replication |
| 6 | Full benchmarks vs Redis, profiling + optimization pass, architecture docs, README, Docker |

## 8. What to Put on the Resume

> Built a Redis-compatible in-memory key-value store in Go from scratch: RESP protocol, skip-list-backed sorted sets, AOF + RDB persistence, pub/sub, pipelining, leader-follower replication, and approximated LRU eviction. Benchmarked at X ops/sec (GET/SET) with p99 latency of Y μs using `redis-benchmark`. Fully compatible with `redis-cli` and standard Redis client libraries.

## 9. What NOT to Claim

- Do **not** claim it's faster than Redis. It won't be. C + io_uring + jemalloc will always beat Go's runtime for this workload.
- Do **not** claim it's production-ready. It's a learning project that demonstrates systems understanding.
- **Do** explain exactly where your implementation diverges from Redis and why. This shows you read the Redis source and understood the tradeoffs.

## 10. References

- **Read the Redis source code.** Start with `server.c`, `t_zset.c` (skip list), `aof.c`, `rdb.c`. It's clean C and surprisingly readable.
- **"Build Your Own Redis" by CodeCrafters** — good for initial structure, but go deeper than their curriculum.
- **"Designing Data-Intensive Applications" by Martin Kleppmann**, chapters 5–7 (replication, partitioning, consistency).
- **Redis documentation on internals:** Especially the pages on persistence, replication, and data types encoding.
- Do **not** wrap an existing KV library (BoltDB, BadgerDB). The point is building the engine yourself.
