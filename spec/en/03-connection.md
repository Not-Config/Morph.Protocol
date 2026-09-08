# Morph Protocol v0.1 — Connection State

> This document is an official English translation of the canonical Russian specification. If translations differ, the version under `spec/ru/` is authoritative.

A Morph Protocol connection is governed by an explicit state machine.

## States

```text
CONNECTED
   |
   | HELLO
   v
NEGOTIATED
   |
   | AUTH
   v
AUTHENTICATED
   |
   | READY received
   v
READY
```

A reconnecting authenticated client may transition into synchronization using `RESUME` before normal event processing continues.

## 1. Transport connected

After the WebSocket transport is established, the server waits for `HELLO`.

Before a valid `HELLO` is accepted, the client MUST NOT send application commands.

## 2. HELLO

The client sends:

```json
{
  "op": "HELLO",
  "protocol": "0.1",
  "client": {
    "name": "Morph Desktop",
    "version": "0.1.0"
  },
  "capabilities": []
}
```

The server validates protocol compatibility and responds with `HELLO_ACK`.

Example:

```json
{
  "op": "HELLO_ACK",
  "protocol": "0.1",
  "session_id": "01K00000000000000000000000",
  "heartbeat_interval_ms": 30000,
  "capabilities": []
}
```

If no compatible protocol version exists, the server returns `ERROR` with code `UNSUPPORTED_PROTOCOL` and SHOULD close the connection.

## 3. AUTH

After `HELLO_ACK`, the client authenticates.

For v0.1, the authentication credential format is intentionally abstract. Existing Morph account authentication may be used by an implementation.

Example:

```json
{
  "op": "AUTH",
  "token": "opaque-authentication-token"
}
```

Credentials MUST NOT be logged in plaintext by clients, servers, gateways, or debugging middleware.

## 4. READY

After successful authentication, the server sends `READY`.

```json
{
  "op": "READY",
  "session_id": "01K00000000000000000000000",
  "user_id": "01J00000000000000000000000"
}
```

Receipt of `READY` means the client may issue application commands.

## Invalid ordering

The server MUST reject packets that are invalid for the current connection state.

Examples:

- `COMMAND` before `READY`;
- `AUTH` before `HELLO_ACK`;
- a second `HELLO` after negotiation;
- application packets after authentication has been revoked.

The server SHOULD return:

```json
{
  "op": "ERROR",
  "code": "INVALID_STATE",
  "message": "Packet is not valid in the current connection state."
}
```

Repeated or clearly malicious state violations MAY cause immediate connection closure.

## Heartbeat

After `HELLO_ACK`, the server communicates the expected heartbeat interval.

The precise heartbeat request/response shape is defined in the packet specification. A peer that does not satisfy liveness requirements within implementation-defined tolerance may be disconnected.

## Resume

After authentication, a client that has retained synchronization state may send `RESUME` with its last successfully applied event sequence.

Example:

```json
{
  "op": "RESUME",
  "after": 1582
}
```

The server may then replay missing events. If the requested sequence can no longer be resumed safely, the server MUST indicate that a full synchronization is required rather than silently skipping state.
