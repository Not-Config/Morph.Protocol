# Morph Protocol v0.1 — Transport

Morph Protocol v0.1 uses WebSocket as its realtime transport.

## Transport requirements

Production connections MUST use WebSocket over TLS (`wss://`). TLS 1.3 SHOULD be used wherever supported by the deployment.

Plaintext WebSocket (`ws://`) MUST NOT be used across untrusted networks. Implementations MAY allow it for local development or isolated test environments.

## Logical independence

The logical Morph Protocol is not permanently coupled to WebSocket. Packet semantics, connection state, commands, results, and events are defined independently so that future versions may support transports such as QUIC without redesigning the application model.

## Encoding

Morph Protocol v0.1 packets are encoded as UTF-8 JSON text messages.

Each WebSocket message MUST contain exactly one complete Morph Protocol packet.

A v0.1 implementation MUST NOT split one Morph packet across multiple application-level WebSocket messages and MUST NOT concatenate multiple Morph packets into one application-level WebSocket message.

## Connection closure

Either endpoint may close the WebSocket connection when:

- the peer violates the protocol state machine;
- malformed packets are received repeatedly;
- authentication fails according to server policy;
- heartbeat timeout is exceeded;
- an unrecoverable protocol error occurs;
- the endpoint is shutting down.

When practical, the endpoint SHOULD send a Morph `ERROR` packet before closing the transport.

## Message size

The exact maximum packet size is deployment-defined in v0.1. Servers MUST enforce a finite maximum and SHOULD return a protocol error before closing when a packet exceeds the allowed size.

Large binary objects such as media attachments are not transferred directly as JSON payloads in the realtime protocol. Media transport will be specified separately.
