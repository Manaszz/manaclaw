# ManaClaw HTTP API Reference

This document describes the REST API exposed by ManaClaw for webhooks, integrations, and automation. The server is implemented with **Fastify** and listens on a configurable port (default **8080**).

**Base URL:** `http://localhost:8080/api/v1`

Replace the host and port when deploying to another environment.

---

## Authentication

When a Bearer token is configured in `config.toml`, every request must include:

```http
Authorization: Bearer <token>
```

If the configured token is empty, authentication is disabled and the `Authorization` header is not required.

Invalid or missing credentials (when auth is enabled) result in **401 Unauthorized**.

---

## Common conventions

### Content type

Unless noted otherwise, request bodies use **JSON** with:

```http
Content-Type: application/json
```

### Timestamps

Query parameters such as `since` and `until` use **ISO 8601** strings (e.g. `2025-03-24T12:00:00.000Z`).

### Error responses

Errors use a JSON body when possible. Typical HTTP status codes:

| Code | Meaning |
|------|---------|
| `400` | Bad request (validation, malformed body or query) |
| `401` | Unauthorized (missing or invalid Bearer token when auth enabled) |
| `404` | Resource not found |
| `409` | Conflict (e.g. duplicate or invalid state transition) |
| `422` | Unprocessable entity (semantic validation failure) |
| `500` | Internal server error |

Exact error payload shapes may include `message`, `code`, or `details` fields depending on implementation version.

---

## Data types

### `Message`

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Internal message identifier |
| `chatJid` | string | Chat / conversation identifier (JID or channel-specific) |
| `sender` | string | Sender identifier or display handle |
| `content` | string | Message text or serialized content |
| `timestamp` | string | ISO 8601 timestamp |
| `messageId` | string | External or upstream message correlation id |

---

## Messages

### Send a message

Queues a message to a workspace for processing.

| | |
|---|---|
| **Method** | `POST` |
| **Path** | `/messages` |

**Description:** Submits user or system content to the specified workspace. The message is accepted asynchronously; the response indicates it was queued.

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workspaceId` | string | yes | Target workspace id |
| `content` | string | yes | Message body |
| `sender` | string | no | Optional sender label for attribution |

**Response** `200 OK` (or `202 Accepted` if the implementation uses accepted semantics)

```json
{
  "messageId": "msg_01hxyz...",
  "status": "queued"
}
```

**Example**

```bash
curl -sS -X POST "http://localhost:8080/api/v1/messages" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"workspaceId":"ws_abc","content":"Hello","sender":"api"}'
```

**Errors:** `400` (validation), `401` (auth), `404` (unknown workspace), `500`.

---

### Poll messages for a workspace

Retrieves messages (e.g. assistant replies) after a given time.

| | |
|---|---|
| **Method** | `GET` |
| **Path** | `/messages/:workspaceId` |

**Description:** Returns a batch of messages for polling clients. Use `since` to avoid duplicates.

**Path parameters**

| Name | Type | Description |
|------|------|-------------|
| `workspaceId` | string | Workspace id |

**Query parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `since` | string (ISO 8601) | no | Return messages newer than this timestamp |
| `limit` | integer | no | Max messages (default server-defined; typical max **50**) |

**Response** `200 OK`

```json
{
  "messages": [
    {
      "id": "m1",
      "chatJid": "chat@example",
      "sender": "assistant",
      "content": "Reply text",
      "timestamp": "2025-03-24T12:00:00.000Z",
      "messageId": "upstream-123"
    }
  ]
}
```

**Example**

```bash
curl -sS "http://localhost:8080/api/v1/messages/ws_abc?since=2025-03-24T00:00:00.000Z&limit=50" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Errors:** `400`, `401`, `404`, `500`.

---

### Server-Sent Events: message stream

Opens a long-lived stream of events for a workspace.

| | |
|---|---|
| **Method** | `GET` |
| **Path** | `/messages/:workspaceId/stream` |

**Description:** Returns `text/event-stream`. Clients should use an SSE-capable HTTP client or `EventSource` in browsers (subject to CORS configuration).

**Path parameters**

| Name | Type | Description |
|------|------|-------------|
| `workspaceId` | string | Workspace id |

**Response** `200 OK`

- **Header:** `Content-Type: text/event-stream`
- **Events:** Named SSE events (implementation may use `event:` lines). Event types:

| Event | Purpose |
|-------|---------|
| `message` | New or updated chat message payload |
| `tool_call` | Tool invocation lifecycle / result |
| `error` | Recoverable or terminal stream error |
| `done` | Stream or turn completed |

Payloads are typically JSON in the `data:` field of each SSE message.

**Example**

```bash
curl -sSN "http://localhost:8080/api/v1/messages/ws_abc/stream" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Accept: text/event-stream"
```

**Errors:** Connection may close with non-200 on auth failure; `401`, `404`, `500` before stream start.

---

## Workspaces

### List workspaces

| | |
|---|---|
| **Method** | `GET` |
| **Path** | `/workspaces` |

**Description:** Returns all registered workspaces.

**Response** `200 OK`

```json
{
  "workspaces": [
    {
      "id": "ws_abc",
      "name": "My Project",
      "folder": "/path/to/workspace"
    }
  ]
}
```

*(Exact list item shape may include additional fields returned by `GET /workspaces/:id`.)*

**Example**

```bash
curl -sS "http://localhost:8080/api/v1/workspaces" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Errors:** `401`, `500`.

---

### Register workspace

| | |
|---|---|
| **Method** | `POST` |
| **Path** | `/workspaces` |

**Description:** Registers a new workspace with optional trigger, channel, and container configuration.

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Human-readable name |
| `folder` | string | yes | Filesystem path to workspace root |
| `trigger` | string | no | Trigger configuration (format depends on deployment) |
| `channel` | string | no | Default channel or routing key |
| `containerConfig` | object | no | Container / runtime overrides |

**Response** `201 Created` or `200 OK` with created workspace object (including `id`).

**Example**

```bash
curl -sS -X POST "http://localhost:8080/api/v1/workspaces" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"Docs","folder":"/data/workspaces/docs","channel":"slack:#docs"}'
```

**Errors:** `400`, `401`, `409`, `500`.

---

### Get workspace

| | |
|---|---|
| **Method** | `GET` |
| **Path** | `/workspaces/:id` |

**Path parameters**

| Name | Type | Description |
|------|------|-------------|
| `id` | string | Workspace id |

**Response** `200 OK` — workspace details object.

**Example**

```bash
curl -sS "http://localhost:8080/api/v1/workspaces/ws_abc" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Errors:** `401`, `404`, `500`.

---

### Delete workspace

| | |
|---|---|
| **Method** | `DELETE` |
| **Path** | `/workspaces/:id` |

**Description:** Removes the workspace registration (and may tear down associated resources per server policy).

**Path parameters**

| Name | Type | Description |
|------|------|-------------|
| `id` | string | Workspace id |

**Response** `204 No Content` or `200 OK` with confirmation body.

**Example**

```bash
curl -sS -X DELETE "http://localhost:8080/api/v1/workspaces/ws_abc" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Errors:** `401`, `404`, `409`, `500`.

---

## Tasks

### List tasks

| | |
|---|---|
| **Method** | `GET` |
| **Path** | `/tasks` |

**Description:** Lists scheduled or ad-hoc tasks, optionally filtered by workspace.

**Query parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `workspaceId` | string | no | Filter by workspace |

**Response** `200 OK`

```json
{
  "tasks": []
}
```

**Example**

```bash
curl -sS "http://localhost:8080/api/v1/tasks?workspaceId=ws_abc" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Errors:** `400`, `401`, `500`.

---

### Create task

| | |
|---|---|
| **Method** | `POST` |
| **Path** | `/tasks` |

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workspaceId` | string | yes | Workspace to run against |
| `prompt` | string | yes | Task instruction or template |
| `scheduleType` | string | yes | One of: `cron`, `interval`, `once` |
| `scheduleValue` | string | yes | Cron expression, ISO datetime, or interval spec per `scheduleType` |
| `maxCalls` | integer | no | Cap on executions (if supported) |

**Response** `201 Created` or `200 OK` with task object including `id`.

**Example**

```bash
curl -sS -X POST "http://localhost:8080/api/v1/tasks" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "workspaceId":"ws_abc",
    "prompt":"Summarize open issues",
    "scheduleType":"cron",
    "scheduleValue":"0 9 * * *",
    "maxCalls":100
  }'
```

**Errors:** `400`, `401`, `404`, `422`, `500`.

---

### Get task

| | |
|---|---|
| **Method** | `GET` |
| **Path** | `/tasks/:id` |

**Description:** Returns task definition and run history.

**Path parameters**

| Name | Type | Description |
|------|------|-------------|
| `id` | string | Task id |

**Response** `200 OK` — task plus `runs` or embedded history per implementation.

**Example**

```bash
curl -sS "http://localhost:8080/api/v1/tasks/task_01hxyz" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Errors:** `401`, `404`, `500`.

---

### Update task

| | |
|---|---|
| **Method** | `PATCH` |
| **Path** | `/tasks/:id` |

**Description:** Updates task state or definition: pause, resume, cancel, or modify schedule/prompt fields supported by the server.

**Path parameters**

| Name | Type | Description |
|------|------|-------------|
| `id` | string | Task id |

**Request body** (partial update; common fields)

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | e.g. `paused`, `active`, `cancelled` |
| `prompt` | string | New prompt text |
| `scheduleType` | string | `cron` \| `interval` \| `once` |
| `scheduleValue` | string | Updated schedule |
| `maxCalls` | integer | Updated cap |

**Response** `200 OK` — updated task object.

**Example**

```bash
curl -sS -X PATCH "http://localhost:8080/api/v1/tasks/task_01hxyz" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status":"paused"}'
```

**Errors:** `400`, `401`, `404`, `409`, `422`, `500`.

---

## Memory

### Search memory facts

| | |
|---|---|
| **Method** | `GET` |
| **Path** | `/memory/:workspaceId/search` |

**Path parameters**

| Name | Type | Description |
|------|------|-------------|
| `workspaceId` | string | Workspace id |

**Query parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `q` | string | yes | Search query |

**Response** `200 OK`

```json
{
  "facts": [],
  "query": "user preference theme"
}
```

**Example**

```bash
curl -sS "http://localhost:8080/api/v1/memory/ws_abc/search?q=deployment" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Errors:** `400`, `401`, `404`, `500`.

---

### List recent facts

| | |
|---|---|
| **Method** | `GET` |
| **Path** | `/memory/:workspaceId/facts` |

**Path parameters**

| Name | Type | Description |
|------|------|-------------|
| `workspaceId` | string | Workspace id |

**Response** `200 OK` — list of recent memory facts.

**Example**

```bash
curl -sS "http://localhost:8080/api/v1/memory/ws_abc/facts" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Errors:** `401`, `404`, `500`.

---

### Trigger memory compaction

| | |
|---|---|
| **Method** | `POST` |
| **Path** | `/memory/:workspaceId/compact` |

**Description:** Requests consolidation or pruning of stored memory for the workspace (long-running; may return immediately with a job id depending on implementation).

**Path parameters**

| Name | Type | Description |
|------|------|-------------|
| `workspaceId` | string | Workspace id |

**Response** `200 OK` or `202 Accepted` with status payload.

**Example**

```bash
curl -sS -X POST "http://localhost:8080/api/v1/memory/ws_abc/compact" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Errors:** `401`, `404`, `409`, `500`.

---

## Usage

### Aggregate token usage

| | |
|---|---|
| **Method** | `GET` |
| **Path** | `/usage` |

**Description:** Returns aggregated token usage statistics across models and workspaces.

**Query parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `workspaceId` | string | no | Restrict to one workspace |
| `since` | string (ISO 8601) | no | Range start |
| `until` | string (ISO 8601) | no | Range end |

**Response** `200 OK`

```json
{
  "totals": {
    "promptTokens": 0,
    "completionTokens": 0,
    "totalTokens": 0
  },
  "byModel": {},
  "workspaceId": "ws_abc"
}
```

*(Field names are illustrative; align with server metrics export.)*

**Example**

```bash
curl -sS "http://localhost:8080/api/v1/usage?workspaceId=ws_abc&since=2025-03-01T00:00:00.000Z" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Errors:** `400`, `401`, `500`.

---

## Identity

### Link identity

| | |
|---|---|
| **Method** | `POST` |
| **Path** | `/identities/link` |

**Description:** Associates a logical `userId` with a channel-specific account.

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `userId` | string | yes | Internal or canonical user id |
| `channel` | string | yes | Channel type (e.g. `slack`, `discord`, `webhook`) |
| `channelUserId` | string | yes | Opaque id from the channel |
| `displayName` | string | no | Optional display name |

**Response** `200 OK` or `201 Created` with link record.

**Example**

```bash
curl -sS -X POST "http://localhost:8080/api/v1/identities/link" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"userId":"user_1","channel":"slack","channelUserId":"U123","displayName":"Ada"}'
```

**Errors:** `400`, `401`, `409`, `500`.

---

### Get identity links

| | |
|---|---|
| **Method** | `GET` |
| **Path** | `/identities/:userId` |

**Path parameters**

| Name | Type | Description |
|------|------|-------------|
| `userId` | string | User id |

**Response** `200 OK` — linked identities for the user.

**Example**

```bash
curl -sS "http://localhost:8080/api/v1/identities/user_1" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Errors:** `401`, `404`, `500`.

---

## Health

### Service health

| | |
|---|---|
| **Method** | `GET` |
| **Path** | `/health` |

**Description:** Liveness and version metadata. Typically does not require authentication (confirm with deployment); if protected, send `Authorization` as for other routes.

**Response** `200 OK`

```json
{
  "status": "ok",
  "version": "1.0.0",
  "uptime": 3600,
  "activeContainers": 2,
  "dbProvider": "sqlite"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | Overall health (`ok`, `degraded`, etc.) |
| `version` | string | Application version |
| `uptime` | number | Seconds since process start |
| `activeContainers` | number | Running workspace containers (if applicable) |
| `dbProvider` | string | Active database backend identifier |

**Example**

```bash
curl -sS "http://localhost:8080/api/v1/health"
```

**Errors:** `503` if unhealthy (if implemented), `500`.

---

## Versioning

The API is versioned under the `/api/v1` prefix. Breaking changes may introduce `/api/v2`; clients should pin integrations to a major version.
