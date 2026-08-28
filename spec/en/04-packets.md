# Morph Protocol v0.1 — Core Packets

Morph Protocol v0.1 defines a small set of top-level packet operations. Application-specific actions are carried inside `COMMAND` and `EVENT` packets rather than being assigned new top-level opcodes.

## Core operations

The initial operation set is:

- `HELLO`
- `HELLO_ACK`
- `AUTH`
- `READY`
- `COMMAND`
- `RESULT`
- `EVENT`
- `ACK`
- `RESUME`
- `HEARTBEAT`
- `ERROR`

## Common rules

Every packet MUST be a JSON object containing an `op` field.

Unknown top-level fields SHOULD be ignored by receivers unless a negotiated capability specifies otherwise. This permits additive evolution within a protocol version.

Unknown `op` values MUST result in an `ERROR` response with code `UNKNOWN_OPERATION`.

## COMMAND

A `COMMAND` asks the server to perform an application operation.

```json
{
  "op": "COMMAND",
  "request_id": "01K00000000000000000000001",
  "type": "message.create",
  "data": {
    "conversation_id": "01J00000000000000000000001",
    "content": {
      "type": "text",
      "text": "Hello!"
    }
  }
}
```

`request_id` MUST uniquely identify an outstanding request within the client session.

A server MUST produce at most one terminal `RESULT` for a given accepted `request_id`.

## RESULT

`RESULT` correlates with a previous command.

Successful example:

```json
{
  "op": "RESULT",
  "request_id": "01K00000000000000000000001",
  "status": "ok"
}
```

Rejected example:

```json
{
  "op": "RESULT",
  "request_id": "01K00000000000000000000001",
  "status": "error",
  "error": {
    "code": "PERMISSION_DENIED",
    "message": "The current user cannot perform this operation."
  }
}
```

A successful `RESULT` means the command was accepted and processed according to server semantics. Clients MUST use authoritative `EVENT` packets, not `RESULT`, to apply shared state transitions unless an operation is explicitly documented as query-only.

## EVENT

An `EVENT` describes an authoritative state transition.

```json
{
  "op": "EVENT",
  "seq": 1582,
  "type": "message.created",
  "data": {
    "id": "01K00000000000000000000002",
    "conversation_id": "01J00000000000000000000001",
    "author_id": "01H00000000000000000000001",
    "content": {
      "type": "text",
      "text": "Hello!"
    }
  }
}
```

Within a synchronization scope, `seq` values MUST increase monotonically.

A client that observes a gap in a sequence MUST NOT assume the missing state does not exist. It SHOULD request synchronization or resume before continuing to treat its local state as authoritative.

## ACK

`ACK` confirms that the client has successfully applied events up to and including a sequence value.

```json
{
  "op": "ACK",
  "seq": 1582
}
```

ACK semantics are cumulative in v0.1.

## RESUME

`RESUME` requests continuation after the client's latest applied sequence.

```json
{
  "op": "RESUME",
  "after": 1582
}
```

The server may replay subsequent events or return an error indicating that full synchronization is required.

## HEARTBEAT

Either endpoint may send:

```json
{
  "op": "HEARTBEAT",
  "nonce": "01K00000000000000000000003"
}
```

The peer SHOULD respond with the same nonce:

```json
{
  "op": "HEARTBEAT",
  "nonce": "01K00000000000000000000003",
  "reply": true
}
```

Implementations MUST prevent heartbeat request loops. A packet containing `"reply": true` is a response and MUST NOT itself trigger another heartbeat response.

## ERROR

`ERROR` represents a protocol-level error rather than an application command failure.

```json
{
  "op": "ERROR",
  "code": "INVALID_STATE",
  "message": "Packet is not valid in the current connection state."
}
```

Application failures associated with a `COMMAND` SHOULD normally be represented through its `RESULT` packet instead.

## Naming conventions

Top-level protocol operations use uppercase names such as `COMMAND` and `EVENT`.

Application command types use dotted lowercase namespaces such as:

```text
message.create
message.edit
member.ban
reaction.add
```

Authoritative event types SHOULD use completed-state naming where practical:

```text
message.created
message.edited
member.banned
reaction.added
```
