# Prerequisites Before Building the Distributed Key-Value Store

## Purpose of This File

This file explains what you should learn before starting this project.

You are a second-year computer science student, so you probably already know basic programming, data structures, operating systems basics, and some networking theory. This guide tells you what is enough, what needs revision, and what can be learned while building.

You do not need to become an expert before starting. You only need enough foundation to avoid getting completely stuck.

---

## Big Picture

This project combines several areas of computer science:

- Programming in Go.
- TCP networking.
- Protocol parsing.
- Data structures.
- Concurrency.
- Operating system concepts.
- File I/O and persistence.
- Distributed systems basics.
- Testing, benchmarking, and profiling.

You do not need to master all of them before writing code.

Recommended approach:

1. Learn the minimum required basics.
2. Start the project.
3. Learn deeper concepts exactly when you need them.
4. Document what you learn.

---

## Prerequisite Roadmap

| Level | Topic | Priority | When You Need It |
| --- | --- | --- | --- |
| Foundation | Go basics | Must learn first | Day 1 |
| Foundation | Git and terminal comfort | Must learn first | Day 1 |
| Foundation | Basic data structures | Must know | Day 5 onward |
| Foundation | Basic testing | Must learn early | Day 3 onward |
| Systems | TCP networking | Must learn early | Day 2 onward |
| Systems | RESP protocol | Must learn early | Day 3 onward |
| Systems | Concurrency and locks | Must learn early | Day 5 onward |
| Systems | File I/O | Must learn before persistence | Day 16 onward |
| Systems | Benchmarking | Should learn mid-project | Day 9 onward |
| Advanced | Skip lists | Must learn before sorted sets | Day 13 onward |
| Advanced | Persistence design | Must learn before AOF/RDB | Day 16 onward |
| Advanced | Replication basics | Learn later | Day 23 onward |

---

## 1. Go Programming Basics

### Why You Need This

The whole project is written in Go. You should be comfortable enough to read and write medium-sized Go programs.

### What You Should Know

- Variables and constants.
- Functions.
- Structs.
- Methods.
- Interfaces.
- Packages and modules.
- Error handling.
- Slices and maps.
- Pointers.
- Goroutines basics.
- Mutex basics.
- Testing basics.

### Minimum Skill Target

Before starting the project, you should be able to write:

- A small CLI program.
- A function with tests.
- A struct with methods.
- A map-backed data store.
- A goroutine that runs in the background.

### Practice Tasks

- [ ] Write a Go program that stores key-value pairs in a map.
- [ ] Add `Set`, `Get`, and `Delete` methods.
- [ ] Add tests for those methods.
- [ ] Add a mutex to make the map safe for concurrent access.
- [ ] Start a goroutine that prints a message every second.

### Learn Just Enough From

- Official Go Tour.
- Go by Example.
- Go standard library documentation.
- Small practice programs.

---

## 2. Terminal, Git, and Project Workflow

### Why You Need This

You will run servers, tests, benchmarks, Redis CLI commands, Docker commands, and Git commands repeatedly.

### What You Should Know

- Navigating directories.
- Running commands.
- Reading command output.
- Creating files and folders.
- Using Git branches.
- Committing changes.
- Reading diffs.
- Running Go commands.

### Commands You Should Recognize

```text
go mod init
go test ./...
go run ./cmd/radis-server
go test -bench=. ./...
git status
git diff
git add
git commit
redis-cli
redis-benchmark
docker build
docker run
```

### Practice Tasks

- [ ] Create a small Go project from scratch.
- [ ] Run it from the terminal.
- [ ] Add tests.
- [ ] Commit the project to Git.
- [ ] Modify a file and inspect the diff.

---

## 3. Data Structures

### Why You Need This

A Redis-like database is mostly data structures plus networking and persistence.

### What You Should Already Know

You should revise:

- Arrays.
- Slices.
- Hash maps.
- Linked lists.
- Queues.
- Stacks.
- Sets.
- Trees basics.
- Sorting.
- Big-O notation.

### What You Need Specifically for This Project

| Feature | Data Structure Needed |
| --- | --- |
| String key-value store | Hash map |
| Lists | Deque or doubly linked list |
| Hashes | Nested hash map |
| Sets | Hash set |
| Sorted sets | Skip list plus hash map |
| TTL expiry | Map plus timestamps |
| LRU eviction | Access metadata and sampling |

### Minimum Skill Target

You should be able to explain:

- Why hash map lookup is usually O(1).
- Why list push/pop at ends can be O(1).
- Why sorted operations need more than a normal hash map.
- Why skip lists can give O(log n) average operations.

### Practice Tasks

- [ ] Implement a stack in Go.
- [ ] Implement a queue in Go.
- [ ] Implement a set using `map[string]struct{}`.
- [ ] Implement a simple linked list.
- [ ] Sort a slice of structs by score and name.
- [ ] Read about skip lists before implementing sorted sets.

---

## 4. Big-O and Performance Thinking

### Why You Need This

This project is performance-sensitive. You need to understand why some approaches are too slow.

### Concepts to Know

- O(1), O(log n), O(n), O(n log n).
- Time complexity.
- Space complexity.
- Amortized cost.
- Constant factors.
- Allocation overhead.
- Lock contention.

### Examples From This Project

| Problem | Bad Approach | Better Approach |
| --- | --- | --- |
| TTL expiry | Scan every key every second | Sample keys in background |
| Sorted set range | Sort all members on every query | Maintain skip list order |
| Concurrent storage | One global lock forever | Start simple, then consider sharding |
| Persistence recovery | Store random binary without version | Use versioned snapshot format |

### Practice Tasks

- [ ] Explain why scanning 1 million keys every second is expensive.
- [ ] Compare hash map lookup and sorted slice lookup.
- [ ] Benchmark two small Go functions.
- [ ] Measure allocations with Go benchmarks.

---

## 5. TCP Networking Basics

### Why You Need This

Redis is not an HTTP server. Redis uses raw TCP connections.

Your server must accept TCP clients and communicate using RESP.

### Concepts to Know

- Client-server model.
- IP address and port.
- TCP connection.
- Listening socket.
- Accepting connections.
- Reading from a connection.
- Writing to a connection.
- Connection close.
- Partial reads.
- Timeouts.

### Go APIs to Learn

```text
net.Listen
listener.Accept
net.Conn
conn.Read
conn.Write
conn.Close
bufio.Reader
bufio.Writer
```

### Minimum Skill Target

Before building the RESP server, you should be able to write a simple TCP echo server.

### Practice Tasks

- [ ] Write a TCP server in Go.
- [ ] Connect to it using `nc`.
- [ ] Read one line from the client.
- [ ] Send the same line back.
- [ ] Handle two clients at the same time using goroutines.

---

## 6. RESP Protocol Basics

### Why You Need This

RESP is how Redis clients talk to Redis servers.

To be compatible with `redis-cli`, your server must understand RESP.

### Concepts to Know

RESP has these main types:

- Simple strings.
- Errors.
- Integers.
- Bulk strings.
- Arrays.
- Null bulk strings.

### Example

Human command:

```text
SET name aritra
```

RESP encoding:

```text
*3\r\n$3\r\nSET\r\n$4\r\nname\r\n$6\r\naritra\r\n
```

Meaning:

- `*3` means array with 3 elements.
- `$3` means next bulk string has length 3.
- `SET` is the first argument.
- `name` is the second argument.
- `aritra` is the third argument.

### Minimum Skill Target

You should be able to manually decode simple RESP commands before writing the parser.

### Practice Tasks

- [ ] Write the RESP encoding for `PING`.
- [ ] Write the RESP encoding for `GET name`.
- [ ] Write the RESP encoding for `SET name aritra`.
- [ ] Parse a RESP array by hand on paper.
- [ ] Write tests for parser examples before implementing parser.

---

## 7. Concurrency Basics

### Why You Need This

Many clients may connect to your server at the same time.

That means multiple goroutines may read and write shared data.

Without proper synchronization, you can get race conditions.

### Concepts to Know

- Goroutines.
- Race conditions.
- Critical sections.
- Mutex.
- RWMutex.
- Channels.
- Atomic counters.
- Deadlocks.
- Goroutine leaks.

### Go APIs to Learn

```text
go func() {}
sync.Mutex
sync.RWMutex
sync.WaitGroup
sync.Once
sync/atomic
chan T
```

### Minimum Skill Target

You should understand why a normal Go map is not safe for concurrent writes.

### Practice Tasks

- [ ] Write a program that increments a shared counter from many goroutines.
- [ ] See what goes wrong without a lock.
- [ ] Fix it with `sync.Mutex`.
- [ ] Use `sync.RWMutex` for many readers and few writers.
- [ ] Run a test with the race detector.

---

## 8. Operating System Basics

### Why You Need This

Databases interact heavily with the operating system.

This project touches:

- Network sockets.
- Files.
- Buffers.
- Processes.
- Signals.
- Memory.

### Concepts to Know

- Process.
- Thread.
- File descriptor.
- Socket.
- Buffer.
- System call.
- Disk flush.
- Atomic rename.
- Signals like `SIGINT` and `SIGTERM`.

### Minimum Skill Target

You do not need advanced OS knowledge, but you should understand that network connections and files are OS resources that must be closed properly.

### Practice Tasks

- [ ] Open and write to a file in Go.
- [ ] Flush buffered data.
- [ ] Rename a temporary file.
- [ ] Handle Ctrl+C in a Go program.
- [ ] Read about what `fsync` does.

---

## 9. File I/O and Persistence Basics

### Why You Need This

An in-memory database loses data when the process exits unless it persists data to disk.

You will implement:

- AOF: append every write command.
- RDB: snapshot the full dataset.

### Concepts to Know

- Opening files.
- Appending to files.
- Buffered writes.
- Flushing buffers.
- Fsync.
- File corruption.
- Temporary files.
- Atomic rename.
- Serialization.
- Replay logs.

### Go APIs to Learn

```text
os.OpenFile
os.Create
file.Write
file.Sync
file.Close
os.Rename
bufio.Writer
encoding/binary
encoding/json
```

### Minimum Skill Target

Before implementing AOF, you should be able to write a list of commands to a file and read them back on startup.

### Practice Tasks

- [ ] Write strings to an append-only log file.
- [ ] Read the file back line by line.
- [ ] Reconstruct an in-memory map from the log.
- [ ] Write a snapshot of a map to disk.
- [ ] Load the snapshot back into memory.

---

## 10. Testing Basics

### Why You Need This

This project has many edge cases. Without tests, debugging will become painful.

### Concepts to Know

- Unit tests.
- Table-driven tests.
- Integration tests.
- Test fixtures.
- Temporary directories.
- Benchmark tests.
- Race detector.

### Go APIs to Learn

```text
testing.T
testing.B
t.TempDir
```

### Useful Commands

```text
go test ./...
go test -v ./...
go test -race ./...
go test -bench=. ./...
```

### Minimum Skill Target

You should be able to write tests before writing the main implementation for small components.

### Practice Tasks

- [ ] Write table-driven tests for a parser function.
- [ ] Write tests for a map-backed store.
- [ ] Write a test that uses a temporary file.
- [ ] Write a benchmark for `Set` and `Get`.

---

## 11. Debugging Basics

### Why You Need This

Systems projects fail in confusing ways:

- Client hangs.
- Server blocks.
- Data races happen.
- Files are partially written.
- Protocol parser gets stuck.

### Concepts to Know

- Reading error messages.
- Adding focused logs.
- Reducing a bug to a small test.
- Checking goroutine leaks.
- Checking file contents.
- Using the race detector.

### Practice Tasks

- [ ] Debug a failing test without guessing.
- [ ] Add logs around network reads and writes.
- [ ] Simulate malformed input.
- [ ] Simulate client disconnect.
- [ ] Simulate restart after writing data.

---

## 12. Benchmarking and Profiling Basics

### Why You Need This

Redis-like systems are judged partly by performance.

You need to know how to measure performance honestly.

### Concepts to Know

- Throughput.
- Latency.
- p50 latency.
- p95 latency.
- p99 latency.
- CPU profile.
- Memory profile.
- Allocations.
- Benchmark noise.

### Tools to Learn

```text
go test -bench
pprof
redis-benchmark
time
```

### Minimum Skill Target

You should be able to run a benchmark, read the result, and avoid making fake performance claims.

### Practice Tasks

- [ ] Benchmark map `Get` and `Set`.
- [ ] Benchmark a parser function.
- [ ] Run `redis-benchmark` against real Redis once.
- [ ] Capture a simple CPU profile from a Go benchmark.

---

## 13. Redis Basics

### Why You Need This

You are building a Redis-like system, so you should know Redis from a user's perspective first.

### Concepts to Know

- Key-value model.
- Strings.
- Lists.
- Hashes.
- Sets.
- Sorted sets.
- TTL.
- Pub/sub.
- Persistence basics.
- Replication basics.

### Commands to Practice in Real Redis

```text
PING
SET
GET
DEL
EXISTS
INCR
DECR
MGET
MSET
EXPIRE
TTL
LPUSH
RPUSH
LPOP
RPOP
LRANGE
HSET
HGET
HGETALL
SADD
SMEMBERS
ZADD
ZRANGE
SUBSCRIBE
PUBLISH
MULTI
EXEC
```

### Practice Tasks

- [ ] Install or run Redis locally.
- [ ] Use `redis-cli`.
- [ ] Try all commands listed above.
- [ ] Observe how Redis responds for missing keys.
- [ ] Observe wrong-type errors.
- [ ] Observe TTL behavior.

---

## 14. Distributed Systems Basics

### Why You Need This

The final part of the project includes leader-follower replication.

You do not need advanced distributed systems before starting. Learn this later in the project.

### Concepts to Know

- Leader and follower roles.
- Replication.
- Initial sync.
- Incremental sync.
- Replication lag.
- Failover.
- Manual promotion.
- Consistency.
- Network failure.
- Reconnect and retry.

### Minimum Skill Target

Before implementing replication, you should be able to explain:

- Why a follower needs an initial copy of data.
- Why writes must be applied in the same order.
- What can happen if the network disconnects.
- Why a replication offset is useful.

### Practice Tasks

- [ ] Read about primary-replica replication.
- [ ] Draw leader-to-follower command streaming.
- [ ] Think through what happens when follower disconnects.
- [ ] Think through what happens when leader dies.

---

## 15. Computer Science Topics to Revise

### Must Revise

- Hash maps.
- Linked lists.
- Big-O notation.
- Sorting.
- Basic networking.
- Processes and files.
- Race conditions.

### Good to Revise

- Trees.
- Binary search.
- Priority queues.
- Serialization.
- Operating system buffers.
- Disk I/O.

### Learn During Project

- RESP protocol.
- Go profiling.
- AOF design.
- RDB snapshot design.
- Skip lists.
- Replication offsets.
- LRU approximation.

---

## 16. What You Do Not Need Before Starting

You do not need to know everything in advance.

You do not need:

- Advanced distributed consensus.
- Raft.
- Paxos.
- Kubernetes internals.
- Kernel programming.
- Advanced compiler design.
- Production database internals.
- Advanced Go optimization.
- Redis source code mastery.

These can come later.

For this project, focus on building a working system step by step.

---

## 17. Suggested 10-Day Preparation Plan

If you feel underprepared, spend 10 days before starting the 30-day build.

### Day 1: Go Basics

- [ ] Complete Go Tour basics.
- [ ] Practice structs, methods, slices, and maps.
- [ ] Write a simple key-value map program.

### Day 2: Go Testing

- [ ] Learn table-driven tests.
- [ ] Add tests to the key-value map program.
- [ ] Learn `go test ./...`.

### Day 3: Go Concurrency

- [ ] Learn goroutines.
- [ ] Learn mutexes.
- [ ] Make map access safe with a lock.

### Day 4: TCP Networking

- [ ] Build a TCP echo server.
- [ ] Connect using `nc`.
- [ ] Handle multiple clients.

### Day 5: Redis User Basics

- [ ] Run Redis locally.
- [ ] Practice Redis string commands.
- [ ] Practice lists, hashes, sets, and sorted sets.

### Day 6: RESP Protocol

- [ ] Learn RESP arrays and bulk strings.
- [ ] Manually encode `PING`, `GET`, and `SET`.
- [ ] Write a simple parser for one RESP command.

### Day 7: Data Structures Revision

- [ ] Revise maps, lists, sets, and sorting.
- [ ] Implement a set in Go.
- [ ] Implement a basic linked list or deque.

### Day 8: File I/O

- [ ] Append commands to a file.
- [ ] Read file contents on startup.
- [ ] Rebuild a map from a log file.

### Day 9: Benchmarks and Profiling

- [ ] Write a Go benchmark.
- [ ] Benchmark map `Set` and `Get`.
- [ ] Read about `pprof`.

### Day 10: Mini Redis Prototype

- [ ] Build a tiny TCP server.
- [ ] Support only `PING`, `SET`, and `GET`.
- [ ] Use a very simple text protocol, not RESP.
- [ ] This gives confidence before the real project.

---

## 18. Minimum Prerequisites Checklist

Before starting the main project, check these off:

- [ ] I can write and run a Go program.
- [ ] I can write basic Go tests.
- [ ] I understand Go maps and slices.
- [ ] I understand basic pointers.
- [ ] I understand basic goroutines.
- [ ] I understand why shared maps need locks.
- [ ] I can write a TCP echo server.
- [ ] I know what `redis-cli` is.
- [ ] I have tried basic Redis commands.
- [ ] I understand RESP arrays and bulk strings at a high level.
- [ ] I can write to and read from files in Go.
- [ ] I understand hash maps and linked lists.
- [ ] I understand basic Big-O notation.
- [ ] I can use Git for commits and diffs.
- [ ] I can run terminal commands comfortably.

If you can check most of these, start the project.

You can learn the rest while building.

---

## 19. Red-Yellow-Green Readiness Assessment

### Green: Start Immediately

You are ready if:

- You can write small Go programs.
- You understand maps, slices, and structs.
- You can write tests.
- You understand basic TCP client/server ideas.
- You are comfortable learning while building.

### Yellow: Prepare for 7-10 Days First

You are almost ready if:

- You know programming but are new to Go.
- You know data structures but not networking.
- You have used Redis but do not know RESP.
- You have weak testing habits.

Use the 10-day preparation plan first.

### Red: Build Smaller Projects First

Do smaller projects first if:

- You are not comfortable writing programs independently.
- You have never used a terminal.
- You do not understand functions, arrays, or maps.
- You cannot debug basic compiler errors.

Suggested smaller projects:

- CLI todo app in Go.
- File-based key-value store.
- TCP echo server.
- In-memory cache with TTL.

---

## 20. Best Learning Strategy for This Project

Use this strategy:

1. **Do not binge theory for weeks.**
   - Learn enough and start building.

2. **Build tiny versions first.**
   - Tiny TCP server before RESP.
   - Tiny map store before full engine.
   - Tiny log file before AOF.

3. **Use tests as learning tools.**
   - Tests force you to understand expected behavior.

4. **Compare with Redis behavior.**
   - Run real Redis and observe responses.

5. **Document tradeoffs.**
   - Your explanation matters as much as the code.

6. **Expect confusion.**
   - Systems projects are difficult because many layers interact.

7. **Debug from the boundary inward.**
   - Network input.
   - Protocol parser.
   - Command router.
   - Engine.
   - Response writer.

---

## 21. Recommended Learning Resources

### Go

- Official Go Tour.
- Go by Example.
- Effective Go.
- Go standard library documentation.

### Networking

- Beej's Guide to Network Programming for concepts.
- Go `net` package docs.
- Small TCP echo server tutorials.

### Redis

- Redis command documentation.
- Redis protocol specification.
- Redis persistence documentation.
- Redis replication documentation.

### Systems and Databases

- Designing Data-Intensive Applications by Martin Kleppmann.
- Redis source code for later reference.
- Blog posts on append-only logs and snapshots.

### Data Structures

- Skip list visual explanations.
- MIT or university data structure notes.
- Visualgo for visual intuition.

---

## Final Advice

This project is supposed to feel hard.

You are combining topics that are usually taught separately:

- Data structures.
- Networking.
- Operating systems.
- Databases.
- Distributed systems.
- Performance engineering.

As a second-year student, you do not need to understand everything perfectly before starting. If you can build small Go programs, understand maps/lists, and are willing to learn TCP and concurrency, you can begin.

The correct path is:

```text
Learn basics -> build small component -> test it -> integrate it -> document tradeoffs -> repeat
```

If you follow that loop, this project will teach you more than many courses.
