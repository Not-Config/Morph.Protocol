# Morph Protocol v0.1 — Application API

Official translation; the [Russian specification](../ru/05-application-api.md) is canonical.
Capability: `application-api-v1`.

## Connection

Obtain a one-time ticket through authenticated POST /auth/ws-ticket, open /ws with
WebSocket subprotocol morph.v0.1, then exchange HELLO → HELLO_ACK → AUTH → READY.
The ticket appears only in AUTH.token. /protocol/ws is an alias. Prepend the API
deployment prefix, such as /api/dev. Servers MUST validate allowed Origin.

Request commands-v1, resume-v1, sync-cursor-v1 and application-api-v1. HELLO_ACK returns
the intersection of requested and supported capabilities. Clients MUST verify required
capabilities before commands. Registration, login, refresh cookies, tickets, files and
bootstrap/full snapshots use HTTPS. Audio/video use LiveKit/WebRTC.

## Commands

Resource IDs are UUID strings. Servers MUST validate required fields, types, bounds and
permissions; unknown additional fields may be ignored. limit is integer 1–100 (default 50),
offset integer ≥ 0 (default 0). Search limit is 1–20 (default 10). ? marks optional fields.

| COMMAND.type | data | Result or authoritative event |
| --- | --- | --- |
| `conversation.list` | `limit?`, `offset?` | RESULT.data: Conversation array |
| `conversation.get` | `conversation_id` | RESULT.data: Conversation |
| `conversation.open` | `username` | conversation.updated, data.conversation |
| `conversation.create_group` | `title`, `member_usernames` | conversation.updated, data.conversation |
| `message.list` | `conversation_id`, `limit?`, `before?` | RESULT.data: Message array; before is ISO 8601 or null |
| `message.create` | `conversation_id`, `content: {type: "text", text}`, `reply_to_id?`, `client_id?` | message.created, data.message |
| `search.query` | `q`, `limit?` | RESULT.data: {people, conversations, servers, channels} |
| `profile.get` | `user_id?` | RESULT.data: Profile; null/omitted ID means the current user |
| `profile.update` | `display_name?`, `bio?` | profile.updated, data.profile |
| `call.list` | `limit?` | RESULT.data: active Call array |
| `call.get` | `call_id` | RESULT.data: Call |
| `call.get_conversation` | `conversation_id` | RESULT.data: Call or null |
| `call.get_channel` | `channel_id` | RESULT.data: Call or null |
| `call.join_conversation` | `conversation_id` | Credentials in RESULT, Call in EVENT |
| `call.join_channel` | `channel_id` | Credentials in RESULT, Call in EVENT |
| `call.join` | `call_id` | Credentials in RESULT, Call in EVENT |
| `call.decline` | `call_id` | call.ended, data.call |
| `call.end` | `call_id` | call.ended, data.call |

conversation.open selects an existing direct conversation when one exists. Group titles
contain 1–128 characters, creation accepts 1–99 members. Messages contain 1–4000 characters
after trimming outer whitespace. client_id is 1–128 characters and correlates optimistic
messages with EVENT; it does not guarantee idempotence. q is 1–128 characters and MUST NOT
consist entirely of whitespace.

Resources use the same JSON fields as HTTPS:

- Conversation: id, kind, title, participants, created_at, updated_at.
  Participant: id, username, display_name, avatar_url, banner_url.
- Message: id, conversation_id, author, string content, is_hidden, hidden_reason,
  reply_to_id, attachments, created_at, edited_at. EVENT retains data.client_id.
- Profile: id, username, display_name, bio, avatar, banner, joined_at, updated_at.
  Images are null or {url, content_type, size_bytes, sha256, updated_at}.
- Call: id, scope_type, conversation_id, channel_id, initiated_by_id, ended_by_id,
  status, join_deadline, created_at, started_at, ended_at, end_reason.

Hidden messages retain HTTPS moderation semantics. Image URLs MUST include deployment
prefix and version. Files are not carried as base64 JSON.

## Results and events

Mutations produce one RESULT and an authoritative EVENT in either arrival order. A
successful RESULT alone MUST NOT change shared client state. Apply EVENT before completing
the operation. Command events contain data.origin:

```json
{"session_id":"connection-id-from-READY","user_id":"authenticated-user-uuid","request_id":"client-request-uuid"}
```

Servers derive origin from the validated session and MUST NOT trust COMMAND.data.origin.
session_id is the WS connection ID in READY, not a secret or an account authorization
session ID. Correlate RESULT by request_id and EVENT by session_id/request_id. Other
connections' events update state without completing this connection's commands.

Join RESULT.data contains livekit_url, short-lived token, token_expires_at, created.
Call arrives in call.created or call.updated. Tokens MUST NOT enter EVENT, event logs,
application logs or persistent client storage. Other mutations in the table return no
shared state in RESULT. conversation.updated reaches participants; profile.updated reaches
the owner's connections. LiveKit webhooks also emit call.started, call.updated, call.ended.

## Access and errors

Servers MUST revalidate account sessions before COMMAND and RESUME. Conversation/server
membership, call permissions and restrictions match HTTPS. Payloads cannot select another actor.
Command errors use RESULT.status = error with error: {code, message, status?, detail?}.
status is an HTTP analogue; detail is a description or restriction object. Protocol errors
use ERROR; unknown commands return UNKNOWN_COMMAND.

Servers retain used request_id values for the connection. Reuse produces DUPLICATE_REQUEST
and MUST NOT execute again or send a second RESULT. After 10,000 commands SESSION_LIMIT
requires reconnecting. Missing RESULT/EVENT may leave the outcome unknown: clients MUST NOT
automatically retry a mutation over another WS or HTTP; synchronize first. Cancelling a
client wait does not cancel an already accepted server operation.
