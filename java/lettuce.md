# Lettuce Redis Cluster: Disconnected Commands and Request Queues

This note summarizes Lettuce behavior relevant to Redis Cluster high availability. It is based on the current source implementation, especially `ClientOptions` and `DefaultEndpoint`.

## Key Options

### `autoReconnect`

`autoReconnect` controls whether Lettuce attempts to restore a connection that was closed or reset unexpectedly.

- `true` (default): Lettuce retries the connection and can replay queued commands after a successful reconnect.
- `false`: Lettuce does not reconnect. Commands are handled with at-most-once semantics and pending commands fail when the connection is lost.

When it is enabled, a command can have **at-least-once** execution semantics: Redis may have executed the command before the connection failure, while the client did not receive the response; replay can then execute it again. Treat non-idempotent writes accordingly.

### `disconnectedBehavior`

This option controls whether a *new* command is accepted while the relevant connection is disconnected.

- `ACCEPT_COMMANDS`: Accept the command and buffer it.
- `REJECT_COMMANDS`: Fail the command immediately with `RedisException`.
- `DEFAULT` (default): Derive the policy from `autoReconnect`.

The source implements `DEFAULT` as follows:

```java
public boolean isRejectCommandsWhileDisconnected() {
    switch (disconnectedBehavior) {
        case REJECT_COMMANDS:
            return true;
        case ACCEPT_COMMANDS:
            return false;
        case DEFAULT:
        default:
            return !autoReconnect;
    }
}
```

Therefore:

| Configuration | New commands while disconnected | Reconnection |
| --- | --- | --- |
| `DEFAULT` + `autoReconnect(true)` | Accepted and buffered | Retried |
| `DEFAULT` + `autoReconnect(false)` | Rejected immediately | Not attempted |
| `REJECT_COMMANDS` + `autoReconnect(true)` | Rejected immediately | Retried |
| `ACCEPT_COMMANDS` + `autoReconnect(false)` | Accepted and buffered | Not attempted; generally unsafe |

`autoReconnect(true)` and `REJECT_COMMANDS` are not contradictory:

1. Lettuce attempts reconnect in the background.
2. During the disconnected period, new commands fail fast instead of accumulating.
3. After the connection is active again, subsequent commands are sent normally.

For typical online Redis Cluster clients, `autoReconnect(true)` with `REJECT_COMMANDS` is often a safer availability posture than default buffering. It avoids an unbounded backlog and makes application-level retry, circuit-breaker, and fallback policies explicit.

## Internal Command Storage

`DefaultEndpoint` keeps separate internal data structures:

```java
private final Queue<RedisCommand<?, ?, ?>> disconnectedBuffer;
private final Queue<RedisCommand<?, ?, ?>> commandBuffer;
private volatile int queueSize;
```

| Internal path | Purpose |
| --- | --- |
| `disconnectedBuffer` | Commands accepted while the connection is disconnected; flushed after activation following reconnect. |
| `commandBuffer` | Commands held instead of immediately writing to the channel, such as when auto-flush is disabled. |
| `queueSize` / `QUEUE_SIZE` | Count of commands written to the channel and tracked while awaiting completion or replay handling. |

`disconnectedBuffer` and `commandBuffer` are different queues. The documentation term “request queue” is not the name of one single internal queue.

## What `requestQueueSize` Limits

`requestQueueSize` is a per-`DefaultEndpoint` capacity value. The default is `Integer.MAX_VALUE`.

It is **not** a single aggregate cap across all pending commands. The source validates each relevant path independently:

```java
if (QUEUE_SIZE.get(this) + commands > requestQueueSize) {
    // reject
}

if (!connected && disconnectedBuffer.size() + commands > requestQueueSize) {
    // reject
}

if (connected && commandBuffer.size() + commands > requestQueueSize) {
    // reject
}
```

Consequences:

- The same configured number is reused as a limit for each internal path.
- It does not mean that all pending commands combined can never exceed `requestQueueSize`.
- For example, after a disconnect, replay-tracked commands and newly buffered disconnected commands can coexist.
- In Redis Cluster, there are multiple node connections/endpoints, so total process-wide backlog can be far higher than one endpoint's configured value.

Lettuce does not expose separate public size settings for `disconnectedBuffer`, `commandBuffer`, and in-flight/replay-tracked commands. `requestQueueSize(int)` is the only queue-capacity option.

## Operational Guidance for Redis Cluster

```java
ClusterClientOptions options = ClusterClientOptions.builder()
        .autoReconnect(true)
        .disconnectedBehavior(ClientOptions.DisconnectedBehavior.REJECT_COMMANDS)
        .requestQueueSize(10_000)
        .build();
```

Use a queue size appropriate to the service's concurrency and heap budget; `10_000` is only an example, not a universal recommendation.

Additional recommendations:

- Enable adaptive topology refresh triggers so failover and slot ownership changes are learned promptly (for example `MOVED_REDIRECT`, `ASK_REDIRECT`, `PERSISTENT_RECONNECTS`, and `UNCOVERED_SLOT`).
- Use `TimeoutOptions` and application-level concurrency limits to bound in-flight work.
- Use idempotency keys, conditional writes, or Lua scripts where duplicate execution of writes would be harmful.
- Avoid `setAutoFlushCommands(false)` on ordinary shared request paths unless deliberate batching and explicit flush behavior are required.
- If policies genuinely differ by traffic class, separate clients or connections can use different option sets. Account for the resulting extra Redis node connections and topology-refresh work.

## Source Locations

- `io.lettuce.core.ClientOptions`
  - `isRejectCommandsWhileDisconnected()` derives `DEFAULT` from `autoReconnect`.
  - `DisconnectedBehavior` defines `DEFAULT`, `ACCEPT_COMMANDS`, and `REJECT_COMMANDS`.
- `io.lettuce.core.protocol.DefaultEndpoint`
  - Defines `disconnectedBuffer`, `commandBuffer`, and `queueSize`.
  - Applies independent `requestQueueSize` checks in `validateWrite(int commands)`.
  - Uses at-least-once reliability when auto-reconnect is enabled.
