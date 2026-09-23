# Redis and Container Memory Management

This note summarizes Redis OSS memory behavior and Linux container memory accounting. It assumes Redis OSS 7+ (with notes applicable to Redis 8.x) and Linux cgroup v2 unless stated otherwise.

## 1. The two memory boundaries

Redis and a container have different memory boundaries. They must not be treated as interchangeable.

```text
Redis maxmemory
  Limits Redis's cache-memory eviction decision.

Redis process RSS
  Resident user-space pages mapped by redis-server.

Container cgroup memory.current
  Memory charged to the container cgroup. This can include process memory,
  page cache, kernel memory, and TCP socket buffers.
```

`maxmemory` is not a hard process-RSS limit and is not a container memory limit. A Redis process can be killed by the kernel or cgroup OOM killer even when its dataset is smaller than `maxmemory`.

## 2. Redis memory categories

### 2.1 Dataset memory

This is memory used by keys, values, expires metadata, and Redis data structures.

Useful metrics from `INFO MEMORY` include:

```text
used_memory
used_memory_dataset
used_memory_rss
mem_fragmentation_ratio
allocator_allocated
allocator_active
allocator_resident
```

`used_memory` is allocator memory known to Redis. `used_memory_rss` is resident process memory reported by the operating system. They can diverge because an allocator can retain freed pages for later reuse.

### 2.2 Client connection memory

Each Redis client can use memory beyond the dataset:

```text
Query buffer   Client -> Redis bytes not fully parsed or executed yet
Output buffer  Redis -> Client replies not fully written to the socket yet
Other buffers  Parsed arguments, queued MULTI commands, client state, etc.
```

Inspect individual clients with:

```bash
redis-cli CLIENT LIST
```

Important fields:

```text
qbuf       Query buffer bytes currently in use
qbuf-free  Preallocated but unused query-buffer capacity
argv-mem   Parsed arguments for an incomplete command
multi-mem  Memory used by queued MULTI commands
obl        Fixed output-buffer bytes in use
oll        Dynamic output-buffer block count
omem       Dynamic output-buffer memory
tot-mem    Total memory consumed by the client
```

Aggregate metrics commonly include `mem_clients_normal`, `mem_clients_pubsub`, and `mem_clients_slaves` in `INFO MEMORY`.

### 2.3 Buffers excluded from the eviction calculation

Some replication and persistence buffers are deliberately excluded from the memory amount used for `maxmemory` eviction decisions. Redis exposes this amount as:

```text
mem_not_counted_for_evict
```

This avoids an eviction feedback loop in which evicting keys creates replication/AOF updates, which then consume more buffer memory and trigger more eviction. It also means that a `maxmemory` value needs headroom for these buffers.

## 3. jemalloc: arenas, fragmentation, and RSS

### 3.1 What an arena is

An arena is an independent heap-management unit inside jemalloc. It is not a Redis database, a Redis Cluster slot, or one fixed contiguous memory range.

Each arena manages size classes, small-object slabs, large-object extents, free memory, locking, and page-decay state. Multiple arenas reduce allocator lock contention in multithreaded programs, but independent arena management can increase fragmentation.

```text
Redis main thread ----\
Redis I/O threads -----+--> jemalloc arenas and thread-local caches
BIO lazy-free thread --/
```

Threads and arenas are not guaranteed to be one-to-one. Threads may share arenas, and small allocations can first be served from a thread-local cache (`tcache`).

### 3.2 Why freeing a key does not immediately reduce RSS

```text
Key is deleted
  -> Redis frees the object to jemalloc
  -> jemalloc may retain the page for reuse
  -> RSS may remain unchanged
  -> later, jemalloc may purge unused pages and the OS may reclaim them
```

An allocator cannot return a page to the OS when that page still contains live allocations. This is a common reason for high RSS after a workload with heavy allocation and deallocation churn. It is not, by itself, proof of a memory leak.

When built with jemalloc, Redis can enable jemalloc background purging with:

```conf
jemalloc-bg-thread yes
```

Those allocator worker threads can use CPU while reclaiming unused pages. This CPU should not be mistaken for Redis command-execution CPU.

### 3.3 Memory churn

Memory churn means frequent allocation and deallocation. Common Redis examples are:

- Short-lived TTL keys.
- Repeatedly overwriting large values.
- Frequent eviction.
- Deleting or trimming large collections.
- Large client buffers that grow and shrink.

During churn, interpret `used_memory`, RSS, allocator metrics, and `lazyfree_pending_objects` together.

## 4. Redis thread model relevant to memory

Redis command execution and keyspace changes are serialized by the main thread. Redis is nevertheless not a one-thread process.

```text
Main thread
  Command execution, event loop, expiry, eviction, Cluster gossip, replication state.

I/O worker threads (io-threads > 1)
  Socket reads/writes and protocol parsing; they do not execute Redis commands.

BIO threads
  Slow file close, AOF fsync, and lazy freeing of large objects.

jemalloc background threads
  Optional allocator page purging.

RDB/AOF child process
  A forked process, not a thread, created for BGSAVE or BGREWRITEAOF.
```

`io-threads` improves networking and protocol-processing throughput, not parallel execution of `GET`, `SET`, Lua, or data-structure commands.

## 5. TTL expiration and physical memory release

TTL expiration has multiple stages.

```text
1. Expire timestamp passes.
2. The key is logically unavailable.
3. Redis removes it from the keyspace.
4. Redis frees the value object, synchronously or through lazy free.
5. jemalloc reuses or eventually purges memory pages.
```

Redis does not create one timer per key. It stores expiration timestamps and expires keys in two ways:

- **Passive expiration**: a command accesses an expired key; Redis removes it and treats it as absent.
- **Active expiration**: the main event loop periodically samples keys with TTLs and removes expired keys that are never accessed.

Therefore an expired key cannot be read after its deadline, but its physical memory does not necessarily disappear at the exact deadline.

Large aggregate objects may be removed from the keyspace first and deep-freed later by `bio_lazy_free`, for example with `UNLINK`, `FLUSHDB ASYNC`, or lazy-free settings. Monitor:

```text
lazyfree_pending_objects
```

In Redis Cluster, each primary expires keys for its own slots and propagates the resulting deletion to its replicas and AOF. Replicas do not independently expire keys while they remain replicas.

## 6. Client query/input buffers

The client query buffer holds request bytes that Redis has received from the kernel but has not fully parsed or executed.

```text
Client
  -> kernel receive buffer
  -> Redis query buffer
  -> RESP parsing and command execution
```

Relevant limits in current Redis OSS configuration are:

```conf
client-query-buffer-limit 1gb
proto-max-bulk-len 512mb
```

`client-query-buffer-limit` is per client. It protects Redis against unbounded request accumulation caused by malformed protocol streams, extremely large requests, or buggy clients. `proto-max-bulk-len` limits one RESP bulk argument. if one argument size is greater than "proto-max-bulk-len" config, redis-server does not accept the request. but it is not related sum of each argument. e.g. if proto-max-bulk-len is 512mb and key is 400mb, value is 400mb, that request is allowed.

Query-buffer growth usually indicates one of the following:

- Unbounded or excessively deep client pipelining.
- Very large request payloads.
- Redis main-thread saturation or a slow command.
- A client protocol bug.

Large pipelines also increase tail latency: after Redis reads a client's buffer, commands already present in that buffer are processed sequentially.

For Lettuce, bound the number of in-flight asynchronous commands. Do not dispatch an unbounded number of `RedisFuture`s on one connection.

## 7. Client output buffers

The client output buffer contains replies that Redis has generated but has not yet fully written to the client socket.

```text
Redis command execution
  -> Redis output buffer
  -> kernel send buffer
  -> client
```

Output buffers grow when the client or network cannot consume responses as fast as Redis produces them. Common causes include slow Pub/Sub subscribers, slow replicas, huge responses, stalled application event loops, and excessive request concurrency.

Output-buffer limits use this syntax:

```conf
client-output-buffer-limit <class> <hard-limit> <soft-limit> <soft-seconds>
```

Example:

```conf
client-output-buffer-limit normal 64mb 32mb 60
```

For each normal client:

- At 64 MB, Redis schedules the connection to be closed as soon as possible.
- At or above 32 MB continuously for 60 seconds, Redis closes the connection.
- Dropping below 32 MB resets the soft-limit timer.

The classes are:

```text
normal   Regular clients and MONITOR clients
replica  Replicas receiving a replication stream from this node
pubsub   Clients with one or more Pub/Sub subscriptions
```

The default limits are:

```conf
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
```

Choose a hard limit above the maximum legitimate single response. Otherwise a healthy request such as a large `LRANGE`, `HGETALL`, `SMEMBERS`, or `MGET` can be disconnected even when the client is not persistently slow.

When Redis closes a connection due to an output-buffer limit, it is a TCP connection failure rather than a normal RESP error reply. Lettuce can observe a connection exception, a command timeout, or a successful retry after reconnect depending on timing and configuration.

For non-idempotent commands, a disconnect produces an **in-doubt outcome**: Redis may have executed the command even if the client did not receive its reply. Automatic replay can cause at-least-once behavior. Treat commands such as `INCR`, `LPUSH`, `XADD *`, `PUBLISH`, and non-idempotent Lua scripts with particular care.

## 8. maxmemory, data eviction, and client eviction

### 8.1 maxmemory

`maxmemory` controls Redis key eviction behavior. When Redis is over its eviction memory threshold and a command needs more memory, Redis follows `maxmemory-policy`:

```text
allkeys-lru / allkeys-lfu / allkeys-random / volatile-*  Evict eligible keys
noeviction                                              Reject memory-growing writes
```

Normal/Pub/Sub client buffers use Redis allocator memory. A large amount of client memory can therefore contribute to a state in which Redis evicts dataset keys or rejects writes even though the dataset itself is small.

Eviction is not necessarily triggered at the exact instant a buffer grows. It is commonly triggered when a later command needs memory. If client buffers alone keep Redis over `maxmemory`, Redis can evict valid data without solving the underlying client-buffer problem.

### 8.2 maxmemory-clients

Use `maxmemory-clients` to cap aggregate client connection memory before it harms the dataset.

```conf
maxmemory 10gb
maxmemory-clients 5%
```

This does **not** allocate a reserved 500 MB client partition. It creates a separate aggregate threshold:

```text
All normal + Pub/Sub client memory
  <= 5% of maxmemory (500 MB in this example)
```

If the aggregate exceeds the threshold, Redis disconnects the smallest number of the largest memory consumers needed to get below it. It includes query buffers, output buffers, and other client-side Redis memory. It is not a per-client limit.

Replica and master replication connections are not client-eviction targets. `maxmemory-clients` also does not cover replication backlog, AOF buffers, RDB/AOF fork Copy-on-Write memory, modules, allocator fragmentation, page cache, or kernel socket buffers.

Use both aggregate and per-client protection:

```conf
maxmemory-clients 5%
client-output-buffer-limit normal 64mb 32mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
client-query-buffer-limit 128mb
```

Values must be selected from real workload characteristics: maximum valid response size, maximum valid request size, number of connections, and the service's failure semantics.

## 9. Kernel socket buffers and process RSS

The network path has two kernel-managed buffers that are separate from Redis user-space buffers.

```text
Client request
  -> kernel receive buffer      # kernel memory
  -> Redis query buffer         # Redis process memory
  -> RESP parsing and command execution
  -> Redis output buffer        # Redis process memory
  -> kernel send buffer         # kernel memory
  -> client response
```

Kernel receive/send buffers:

- Are not part of Redis `used_memory`.
- Are not part of Redis `used_memory_rss`.
- Are not part of the Redis process RSS shown by `ps` or `top`.
- Can be charged to the container cgroup and can contribute to cgroup OOM.

Redis user-space output-buffer limits do not directly limit bytes already accepted by the kernel send queue. Similarly, query-buffer limits do not include bytes still waiting in the kernel receive queue.

During copying, user-space and kernel-space copies can coexist temporarily.

## 10. Container memory and cgroup v2

For a cgroup v2 container:

```text
memory.current
  = user-space anonymous memory
  + file/page cache
  + kernel memory
  + TCP socket buffer memory
  + other cgroup-charged memory
```

This is why the following can happen:

```text
Redis used_memory_rss: 8 GiB
Redis process RSS:    8 GiB
Container memory:     9 GiB
```

The extra memory can include page cache, kernel structures, and socket buffers. Inspect cgroup v2 directly:

```bash
cat /sys/fs/cgroup/memory.current
grep -E '^(anon|file|kernel|sock|slab) ' /sys/fs/cgroup/memory.stat
cat /sys/fs/cgroup/memory.events
```

Useful meanings:

```text
anon      User-space anonymous memory, including most Redis heap memory
file      File cache and mapped file-backed memory
kernel    Kernel-memory total
sock      Network socket-buffer memory
slab      Kernel slab memory
oom       Number of cgroup OOM events
oom_kill  Number of processes killed by cgroup OOM
```

TCP buffer growth can cause a container OOM even when Redis data memory is below `maxmemory`. High connection counts, slow clients, slow Redis command execution, deep pipelines, and TCP autotuning can all increase socket memory.

## 11. Inactive file cache and monitoring

Inactive file cache is not free memory. It is memory currently charged to the cgroup, but it is commonly reclaimable under pressure.

```text
New allocation needs memory
  -> kernel attempts to reclaim inactive file cache
  -> cache page can be discarded if clean
  -> later file access may require disk I/O again
```

Reclaim is not guaranteed to be instant or cost-free. Dirty pages can require writeback first, and other kernel conditions can slow reclaim.

Do not confuse `inactive_file` with `inactive_anon`. Anonymous memory is generally not freely discardable; it requires swap or special memory semantics to be reclaimed.

### 11.1 Prometheus/cAdvisor metrics

The exact implementation varies by cAdvisor and runtime, but the usual interpretation is:

```text
container_memory_rss
  Process RSS-oriented memory. It misses kernel socket buffers.

container_memory_usage_bytes
  Current cgroup memory usage; commonly close to memory.current.

container_memory_working_set_bytes
  Commonly usage minus inactive file cache. It is a useful leading indicator,
  but it is not the exact cgroup OOM threshold.
```

Use both `usage` and `working_set` for alerting:

```text
Warning:
  working_set / memory_limit > 80-85% for a sustained period

Critical:
  usage / memory_limit > 90-95%
  AND working_set / memory_limit > 80-85%

Fast-growth alert:
  Recent usage growth predicts exhaustion of the memory limit soon.
```

If `usage` is high but working set is much lower, inactive file cache may be responsible and the kernel may reclaim it. This can be alert noise, but it is still useful as a capacity signal. If both are high, the cgroup is much closer to OOM.

At extremely high usage (for example, 98-99% of the container limit), alert even when working set is lower because sudden anonymous or socket-buffer allocations can reach the cgroup limit before reclaim completes.

For Redis-specific diagnosis, correlate container memory with:

```text
Redis INFO MEMORY: used_memory, used_memory_rss, mem_clients_*
Redis CLIENT LIST: qbuf, omem, tot-mem
cgroup memory.stat: sock, anon, file, kernel, slab
cgroup memory.events: oom, oom_kill
Redis INFO STATS: evicted_keys, expired_keys
```

When monitoring Kubernetes, compare a specific Redis container's metrics with that container's memory limit. Avoid accidentally alerting on duplicate pod-level or pause-container cAdvisor series.

## 12. Operational checklist

1. Set `maxmemory` below the container memory limit, leaving headroom for fragmentation, replication/AOF buffers, fork Copy-on-Write, and kernel memory.
2. Select a `maxmemory-policy` that matches data-loss semantics; use `noeviction` for data that must not be evicted.
3. Configure `maxmemory-clients` for production instances with many clients.
4. Set per-client output-buffer and query-buffer limits based on valid request/response sizes.
5. Bound application connection counts and in-flight asynchronous requests.
6. Separate Pub/Sub, blocking, and bulk-data workloads from latency-sensitive command traffic when appropriate.
7. Monitor client `qbuf`, `omem`, and `tot-mem` before they become an eviction or OOM problem.
8. Monitor cgroup `memory.current`, `memory.stat:sock`, and `memory.events`, not only Redis RSS.
9. Add TTL jitter when many keys would otherwise expire simultaneously.
10. Treat reconnect/retry after a connection close as an in-doubt operation for non-idempotent writes.

