# Morph Protocol v0.1 — Overview

Morph Protocol is the application protocol used by Morph clients and servers to exchange commands, results, events, synchronization state, and connection metadata.

The protocol defines data, state transitions, and behavior. It does not define how clients render or present that data.

## Design goals

Morph Protocol v0.1 is designed around the following principles:

- event-driven state synchronization;
- device-aware sessions;
- transport independence at the logical protocol level;
- presentation independence;
- explicit versioning;
- resumable realtime sessions;
- extensibility without breaking older clients;
- security provided by established cryptographic standards rather than custom primitives.

## v0.1 scope

The initial experimental version uses:

- WebSocket as the realtime transport;
- TLS 1.3 for transport security;
- UTF-8 JSON for packet encoding;
- a command / result / event model;
- sequence-based synchronization and resume;
- heartbeat messages for connection liveness.

The following are intentionally out of scope for v0.1 and may be added later:

- binary encoding;
- Morph Secure Session;
- device cryptographic identity;
- end-to-end encryption;
- encrypted attachment format;
- QUIC transport;
- extension namespaces.

## Protocol lifecycle

A typical connection follows this flow:

```text
Client                                Server
  |                                     |
  |---- WebSocket over TLS ------------>|
  |---- HELLO -------------------------->|
  |<--- HELLO_ACK ----------------------|
  |---- AUTH --------------------------->|
  |<--- READY --------------------------|
  |                                     |
  |---- COMMAND ------------------------>|
  |<--- RESULT -------------------------|
  |<--- EVENT --------------------------|
  |---- ACK ---------------------------->|
  |                                     |
```

A reconnecting client may use `RESUME` after authentication to request events after its last known event sequence.

## Core semantic model

Morph Protocol distinguishes three concepts:

- **COMMAND** — the client asks the server to perform an operation.
- **RESULT** — the server reports whether that command was accepted or rejected.
- **EVENT** — the server reports that authoritative state has changed.

A successful `COMMAND` does not itself represent the state change. The corresponding `EVENT` does.

Example:

```text
COMMAND member.ban
RESULT  ok
EVENT   member.banned
```

This separation allows clients to reconcile state consistently across multiple devices and reconnects.

## Version status

All `0.x` protocol versions are experimental and may contain breaking changes.

`1.0` will represent the first stable protocol specification.
