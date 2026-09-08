# Morph Protocol v0.1 — Прикладной API

Каноническая спецификация. Capability: `application-api-v1`.

## Подключение

Клиент получает одноразовый ticket через авторизованный `POST /auth/ws-ticket`,
открывает `/ws` с WebSocket subprotocol `morph.v0.1` и выполняет
`HELLO → HELLO_ACK → AUTH → READY`. Ticket передаётся только в `AUTH.token`.
Также доступен `/protocol/ws`. Перед адресами добавляется префикс развёртывания,
например `/api/dev`. Сервер MUST проверять Origin по списку разрешённых адресов.

Клиент запрашивает `commands-v1`, `resume-v1`, `sync-cursor-v1`, `application-api-v1`.
HELLO_ACK возвращает пересечение запрошенных и поддерживаемых capabilities.
Клиент MUST проверить обязательные capabilities перед выполнением команд.

Регистрация, вход, refresh cookies, tickets, загрузка и скачивание файлов используют
HTTPS. Начальные снимки и полная синхронизация также доступны через HTTPS.
Аудио и видео передаются через LiveKit/WebRTC. Перечисленные ниже операции после
готовности соединения выполняются через Morph Protocol.

## Команды

Идентификаторы ресурсов — строки UUID. Сервер MUST проверять обязательные поля,
типы, границы и права доступа. Неизвестные дополнительные поля могут игнорироваться.
`limit`: целое 1–100, по умолчанию 50; `offset`: целое ≥ 0, по умолчанию 0.
Для поиска limit: 1–20, по умолчанию 10. `?` обозначает необязательное поле.

| COMMAND.type | data | Результат или авторитетное событие |
| --- | --- | --- |
| `conversation.list` | `limit?`, `offset?` | RESULT.data: массив Conversation |
| `conversation.get` | `conversation_id` | RESULT.data: Conversation |
| `conversation.open` | `username` | `conversation.updated`, data.conversation |
| `conversation.create_group` | `title`, `member_usernames` | `conversation.updated`, data.conversation |
| `message.list` | `conversation_id`, `limit?`, `before?` | RESULT.data: массив Message; before — ISO 8601 или null |
| `message.create` | `conversation_id`, `content: {type: "text", text}`, `reply_to_id?`, `client_id?` | `message.created`, data.message |
| `search.query` | `q`, `limit?` | RESULT.data: `{people, conversations, servers, channels}` |
| `profile.get` | `user_id?` | RESULT.data: Profile; null/отсутствие ID означает свой профиль |
| `profile.update` | `display_name?`, `bio?` | `profile.updated`, data.profile |
| `call.list` | `limit?` | RESULT.data: массив активных Call |
| `call.get` | `call_id` | RESULT.data: Call |
| `call.get_conversation` | `conversation_id` | RESULT.data: Call или null |
| `call.get_channel` | `channel_id` | RESULT.data: Call или null |
| `call.join_conversation` | `conversation_id` | Данные подключения в RESULT, Call в EVENT |
| `call.join_channel` | `channel_id` | Данные подключения в RESULT, Call в EVENT |
| `call.join` | `call_id` | Данные подключения в RESULT, Call в EVENT |
| `call.decline` | `call_id` | `call.ended`, data.call |
| `call.end` | `call_id` | `call.ended`, data.call |

conversation.open выбирает существующий личный диалог, если он уже создан.
Название группы содержит 1–128 символов; при создании передаются 1–99 участников.
Текст сообщения содержит 1–4000 символов после удаления внешних пробелов.
client_id содержит 1–128 символов и связывает локальное сообщение с EVENT;
это не гарантия идемпотентности повторных команд. q содержит 1–128 символов
и MUST NOT состоять только из пробелов.

Ресурсы используют поля соответствующих JSON-ресурсов HTTPS API:

- Conversation: id, kind, title, participants, created_at, updated_at.
  Участник: id, username, display_name, avatar_url, banner_url.
- Message: id, conversation_id, author, строковый content, is_hidden, hidden_reason,
  reply_to_id, attachments, created_at, edited_at. Событие сохраняет client_id в data.client_id.
- Profile: id, username, display_name, bio, avatar, banner, joined_at, updated_at.
  Изображение — null либо `{url, content_type, size_bytes, sha256, updated_at}`.
- Call: id, scope_type, conversation_id, channel_id, initiated_by_id, ended_by_id,
  status, join_deadline, created_at, started_at, ended_at, end_reason.

Скрытые сообщения сохраняют правила модерации HTTPS API. URL изображений MUST
содержать префикс развёртывания и версию изображения. Файлы не передаются base64 JSON.

## RESULT и EVENT

Изменение создаёт один RESULT и авторитетное EVENT. Порядок их доставки не задан.
Успешный RESULT сам по себе MUST NOT изменять общее состояние клиента.
Клиент дожидается EVENT и применяет его перед завершением операции.

События команды содержат data.origin:

```json
{"session_id":"connection-id-from-READY","user_id":"authenticated-user-uuid","request_id":"client-request-uuid"}
```

Сервер формирует origin из проверенной сессии и MUST NOT доверять origin из COMMAND.data.
session_id — идентификатор WS-соединения из READY, не секрет и не ID сессии авторизации
аккаунта. RESULT сопоставляется по request_id, EVENT — по паре session_id/request_id.
События других соединений обновляют состояние, но не завершают локальные команды.

RESULT.data при входе в звонок содержит livekit_url, краткоживущий token,
token_expires_at, created. Call приходит через call.created или call.updated.
Токен MUST NOT попадать в EVENT, журнал событий, логи или постоянное хранилище клиента.
Остальные изменения из таблицы не возвращают общее состояние в RESULT.
conversation.updated получают участники; profile.updated — соединения владельца профиля.
Webhooks LiveKit также создают call.started, call.updated и call.ended.

## Доступ и ошибки

Перед COMMAND и RESUME сервер MUST повторно проверять сессию аккаунта.
Членство в диалогах/серверах, права на звонки и ограничения аккаунта совпадают с HTTPS.
Данные команды не позволяют выбрать другого действующего пользователя.

Ошибка команды: RESULT.status = error и `error: {code, message, status?, detail?}`.
status — числовой аналог HTTP-статуса, detail — описание или объект ограничения.
Ошибки протокола передаются через ERROR; неизвестная команда — UNKNOWN_COMMAND.

Сервер хранит использованные request_id на время соединения. Повтор вызывает ERROR
DUPLICATE_REQUEST: операция MUST NOT выполняться снова или создавать второй RESULT.
После 10 000 команд реализация возвращает SESSION_LIMIT и требует переподключения.
При потере RESULT/EVENT исход операции может быть неизвестен. Клиент MUST NOT
автоматически повторять изменение через другой WS или HTTP; сначала нужна синхронизация.
Отмена ожидания клиентом не отменяет уже принятую сервером операцию.
