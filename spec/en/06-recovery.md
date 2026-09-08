# Morph Protocol v0.1 — Event recovery

Official translation; the [Russian specification](../ru/06-recovery.md) is canonical.
Capability: `sync-cursor-v1`.

Connections of one user in one API deployment share an EVENT stream. seq is a safe JSON
integer ≥ 0; the first EVENT has seq = 1. Cursors MUST NOT be reused across accounts or
deployments. Assigning seq, appending and bounding the log MUST be atomic. Reading current
seq and the log MUST return a consistent snapshot. Morph uses Redis Lua. Pub/sub and replay
may duplicate or reorder delivery; clients restore order using seq.

## Initial synchronization

When negotiated, READY includes the user's current seq and MUST precede live EVENT packets.
Without local state, retain this temporary boundary, fetch HTTPS snapshots of conversations,
loaded message history and calls, then apply events above the boundary. Buffer events during
the snapshot. Accept the cursor only after successful synchronization; it MUST NOT be retained
without corresponding state. Closing the connection cancels its snapshot requests.

## Reconnection

With retained state, RESUME after READY from the last applied seq. Lower numbers are duplicates.
Gaps require buffering and RESUME; ACK MUST NOT skip a gap. Apply events sequentially, ACK after
successful application. Recovery completes at READY.seq; an established connection's gap target
is the highest received seq. If a contiguous range is unavailable, the server returns:

```json
{"op":"ERROR","code":"SYNC_FULL_REQUIRED","message":"Stored event history is insufficient for resume.","seq":2500}
```

Fetch a full snapshot at this boundary following initial synchronization rules. A lower seq
after Redis reset also requires full synchronization. REALTIME_UNAVAILABLE requires closing
and performing full synchronization after service recovery. The API database is the source
of full state; Redis is a bounded replay log.

Defaults: last 1000 events, log TTL 24 hours after the last append. The web client buffers up
to 1000 events and waits up to 30 seconds for synchronization. Exceeding either limit requires
a new connection and full snapshot. This is not an unlimited offline guarantee. Old request
results and cursors MUST NOT overwrite successor connection state.

## Heartbeat and limits

HEARTBEAT is supported after HELLO_ACK. Replies MUST repeat nonce and include reply: true.
Replies never trigger replies. Different nonces and unrelated events do not acknowledge an
outstanding heartbeat. Default interval: 25 seconds. The client closes if its heartbeat remains
unanswered at the next interval. The server waits up to three intervals for a packet; before
READY, each receive timeout is 10 seconds.

HELLO_ACK.max_packet_bytes advertises the maximum UTF-8 JSON packet size accepted by the server,
default 65,536 bytes. Exceeding it causes PACKET_TOO_LARGE and close 1009. Unknown fields are
ignored; unknown operations produce UNKNOWN_OPERATION.
