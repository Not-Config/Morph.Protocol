# Morph Protocol v0.1 — Terminology

This document defines common terms used by the Morph Protocol specification.

## Client

An application that connects to a Morph server and implements Morph Protocol. A client may be an official Morph application or a third-party implementation.

## Server

A system that accepts Morph Protocol connections and maintains authoritative application state.

## Connection

A single WebSocket connection between a client and server.

## Session

A logical authenticated relationship between a client instance and the server. A session may outlive an individual network connection when resume support is available.

## Device

A registered client installation associated with a Morph account. Device cryptographic identity is not defined by v0.1, but the protocol is designed so that it can be added later.

## Packet

A single Morph Protocol message transferred over the underlying transport.

## Command

A client request to perform an operation that may change authoritative server state.

## Result

The server response corresponding to a command. It reports whether the command was accepted, rejected, or failed.

## Event

An authoritative notification that state has changed. Events are ordered by sequence identifiers within the synchronization scope defined by the server.

## Sequence

A monotonically increasing identifier associated with ordered events. Clients use sequence values to detect missing events and resume synchronization.

## Request ID

A client-generated identifier used to correlate a `COMMAND` with its `RESULT`.

## Capability

A named optional feature supported by a client or server. Capability negotiation allows compatible behavior without assuming that every implementation supports every feature.

## Presentation

The visual or interactive representation of protocol state. Presentation is explicitly outside the scope of Morph Protocol.
