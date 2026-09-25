# Lettuce Redis Cluster `ReadFrom`

`ReadFrom` is a connection-level policy that determines where Lettuce sends read-only commands in a Redis Cluster. It does not change write routing: writes always go to the primary that owns the key's hash slot.

For a key-based read, Lettuce first resolves the key's hash slot and then selects an eligible primary or replica for that slot according to the configured policy. The default is `ReadFrom.MASTER`.

```java
StatefulRedisClusterConnection<String, String> connection = client.connect();
connection.setReadFrom(ReadFrom.REPLICA_PREFERRED);
```

## Read Policies

| Policy | Selection behavior | Consistency and availability implications |
| --- | --- | --- |
| `MASTER` | Read from the current primary only. | Default. Provides the current primary's view and is the correct choice for read-after-write and consistency-sensitive data. |
| `MASTER_PREFERRED` | Prefer the primary; use replicas only when the primary is unavailable. | Normally current reads; during primary unavailability, stale replica reads are possible. Useful for a deliberate read-only degradation mode. |
| `REPLICA` | Read from replicas only. | Reduces primary read load, but reads fail when no eligible replica is available. Stale data is possible. |
| `REPLICA_PREFERRED` | Prefer replicas; fall back to the primary when no replica is available. | Common read-scaling policy. Stale data is possible during normal operation. |
| `LOWEST_LATENCY` | Select the eligible node with the lowest topology-refresh latency snapshot. | Can use either primary or replica; stale and non-monotonic reads are possible. |
| `ANY` | Select any eligible node. | Primary or replica selection is not a consistency policy; generally avoid for authoritative request paths. |
| `ANY_REPLICA` | Select any eligible replica. | Distributes reads across replicas; stale data is possible and reads fail without an eligible replica. |

All policies except `MASTER` can return stale data because Redis replication is asynchronous. Replica reads use Redis `READONLY` mode on the corresponding replica connection.

## Consistency Effects

Replica reads do not provide read-after-write or monotonic-read guarantees.

```text
SET user:1 v2  -> primary accepts the write
GET user:1     -> replica can still return v1 or nil until replication catches up
```

Even `REPLICA_PREFERRED` can return different versions across consecutive reads because requests can be served by a replica at one point and the primary at another. A failover can also expose a promoted primary that did not receive the previous primary's final writes.

Use `MASTER` for sessions, authentication and authorization state, inventory, payment state, distributed locks, and reads immediately following a write. Use replica policies only where stale data is explicitly acceptable, such as cache-like reads, search results, reporting, or derived statistics.

`WAIT` can confirm that a write reached a requested number of replicas, but it does not guarantee that a later `ReadFrom` selection will use one of those replicas. It is not a general replica-read consistency guarantee.

## Recommended Traffic Separation

Avoid changing `ReadFrom` frequently on a shared connection. Separate traffic with different consistency requirements into distinct connections or client beans.

```java
// Consistency-sensitive path
primaryConnection.setReadFrom(ReadFrom.MASTER);

// Stale-tolerant, read-heavy path
replicaConnection.setReadFrom(ReadFrom.REPLICA_PREFERRED);
```

Separate clients or connections increase Redis node connection count and topology-refresh work, so use the split only when the different semantics are intentional.

## `LOWEST_LATENCY` and Topology Refresh

`LOWEST_LATENCY` was introduced as the clearer name for the old `NEAREST` option. Since Lettuce 6.1.7, `NEAREST` is deprecated and is an alias for the same object:

```java
public static final ReadFrom LOWEST_LATENCY =
        new ReadFromImpl.ReadFromLowestCommandLatency();

@Deprecated
public static final ReadFrom NEAREST = LOWEST_LATENCY;
```

The policy does not calculate a rolling p50/p95/p99 from application commands, and it does not select the geographically nearest node directly. Its latency value is a point-in-time measurement obtained during cluster topology discovery or refresh.

For each topology source node, Lettuce sends `CLUSTER NODES` and `INFO`. The latency used for this policy is specifically the duration of the `CLUSTER NODES` command:

```text
`CLUSTER NODES` encoded by Lettuce
  -> outbound network transport
  -> Redis queueing and command execution
  -> inbound network transport
  -> response completion in Lettuce
```

The measured value is recorded in nanoseconds from command encoding until command completion. `INFO` provides topology-related supplementary data such as client count and replication offset, but its response time is not the `LOWEST_LATENCY` selection value.

After refresh, Lettuce stores this value on the node snapshot, orders eligible nodes by ascending latency, and `LOWEST_LATENCY` chooses the first candidate. The ordering remains a snapshot until the next topology refresh; it is not recalculated for every application read.

The value does not include TCP connection establishment or TLS handshake time because the refresh connections are opened before the topology commands are timed. It should therefore be understood as a command RTT sample, not as a complete end-to-end connection cost or a durable node-performance score.

### Dynamic Refresh Sources

`LOWEST_LATENCY` requires `dynamicRefreshSources=true` to obtain topology and latency samples from all discovered cluster nodes. This is the default.

- With dynamic sources enabled, Lettuce discovers nodes and queries each discovered node during topology refresh.
- With dynamic sources disabled, only the initial seed nodes are topology sources, so latency information for other nodes is incomplete or unavailable. `LOWEST_LATENCY` then cannot reliably represent the entire cluster.

Each refresh creates temporary topology-refresh connections to its source nodes. Lettuce issues `CLUSTER NODES` and `INFO` through these connections and closes them after refresh completion. These are distinct from long-lived command-routing connections, which are allocated on demand when application traffic first needs a node. Short periodic refresh intervals multiply this temporary connection and command load by the number of nodes and application instances.

## Operations Guidance

- Use `MASTER` by default unless the business behavior explicitly tolerates stale reads.
- Use `REPLICA_PREFERRED` for read scaling where primary fallback is acceptable.
- Use `REPLICA` only when avoiding primary reads is more important than read availability.
- Use `MASTER_PREFERRED` to keep an intentionally stale read-only mode during a primary outage.
- Use `LOWEST_LATENCY` only when geographic/network latency differences justify it and stale/non-monotonic reads are safe.
- When using `LOWEST_LATENCY`, keep dynamic refresh sources enabled and choose periodic/adaptive refresh settings that balance routing freshness against topology-refresh connection load.
