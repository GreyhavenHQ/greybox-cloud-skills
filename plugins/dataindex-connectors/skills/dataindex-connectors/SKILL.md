---
name: dataindex-connectors
description: >
  Operate the DataIndex ingesters (slack, email/mbsync, calendar, contactdb,
  api_document) running in `apps/dataindex`. Covers connector lifecycle
  (configure, sync, reset, reset-backfill), entity/search queries via REST + MCP,
  log diagnostics, and recovery from common stuck states. Use when the user
  asks to inspect, query, restart, reset, or debug a DataIndex connector, or
  to search/query indexed data (emails, calendar events, slack messages,
  meetings, documents).
---

# DataIndex Connectors

## What this is

DataIndex (`apps/dataindex/`) is a FastAPI service that periodically pulls data from
external sources via "ingesters" (= connectors) and stores `Entity` rows in
PostgreSQL plus chunks in Qdrant. Two equivalent query surfaces are exposed:

- **REST API** at `http://localhost:42000/dataindex/api/v1/*` — easy to curl.
- **MCP** at `http://localhost:42000/dataindex/mcp/` (FastMCP streamable-http) —
  same data, designed for LLM/agent consumption.

For ad-hoc work from a shell, **prefer the REST API**. The MCP transport requires
a session-ID handshake that is awkward over curl.

## Topology cheatsheet

| Container | Role | How to reach |
|---|---|---|
| `internalai-dataindex-backend-1` | API + ingesters | `http://localhost:42000/dataindex/` (via Caddy) |
| `internalai-dataindex-postgres-1` | entities + sync state | `psql -U dataindex -d dataindex` (inside container) |
| `internalai-dataindex-qdrant-1` | vector chunks | internal only |
| `internalai-dataindex-redis-1` | contact cache | internal only |

Internal port is `42180`; Caddy routes `/dataindex/*` to it. Always go through
`http://localhost:42000/dataindex/...`.

## Entity types

`EntityType` enum (`src/dataindex/core/models.py`):

```
calendar_event, email, meeting, conversation, conversation_message,
threaded_conversation, document, contact, webpage
```

Per-connector typical types:

| Connector | Produces |
|---|---|
| `slack` | `conversation`, `conversation_message`, `threaded_conversation` |
| `email` (mbsync) | `email` |
| `calendar` (ICS) | `calendar_event` |
| `contactdb` | `contact` |
| `api_document` | `document` |

Entity IDs are formatted `<connector_id>:<native_id>` (e.g.
`slack:1777496398.194199`, `slack:channel:C08SU582EAX`).

## Common operations (REST)

All examples assume the platform is running locally. Replace the base URL if
not.

### Connector status (sync state, entity counts, health)

```bash
curl -sS http://localhost:42000/dataindex/api/v1/connectors/status \
  | python3 -m json.tool
```

Returns one entry per connector with `sync_status`, `last_sync_at`,
`next_sync_at`, `entity_count`, `error`, `ingester_health`.

### Connector configs (DB-backed, sensitive fields masked)

```bash
curl -sS http://localhost:42000/dataindex/api/v1/connectors \
  | python3 -m json.tool
```

Editable connectors (`source: "database"`) can be patched; environment-sourced
ones (`editable: false`) cannot.

### Update a connector config

`PATCH /api/v1/connectors/{name}` with partial `config`. Masked sensitive fields
(`"********"`) are preserved as-is — only send keys you want to change.

```bash
curl -sS -X PATCH http://localhost:42000/dataindex/api/v1/connectors/email \
  -H "Content-Type: application/json" \
  -d '{"config":{"imap_host":"imap.gmail.com"}}'
```

### Trigger an immediate sync

```bash
curl -sS -X POST http://localhost:42000/dataindex/api/v1/connectors/trigger-sync \
  -H "Content-Type: application/json" \
  -d '{"connector_name":"calendar"}'
```

### Reset a connector (DESTRUCTIVE — wipes all entities for that connector)

```bash
curl -sS -X POST http://localhost:42000/dataindex/api/v1/connectors/reset-ingestion \
  -H "Content-Type: application/json" \
  -d '{"connector_name":"slack"}'
```

Or via CLI inside the container (same effect, plus tabular output):

```bash
docker exec internalai-dataindex-backend-1 \
  uv run dataindex reset-ingestion --connector slack --force
```

### Reset only the backfill state (keeps entities)

```bash
docker exec internalai-dataindex-backend-1 \
  uv run dataindex reset-backfill slack --force
```

Use this when the connector is stuck on `backfill_completed_no_new_channels` /
similar but you don't want to re-fetch already-stored data.

### Query entities (filterable, paginated)

```bash
# Latest 5 slack messages
curl -sS "http://localhost:42000/dataindex/api/v1/query?limit=5&entity_types=conversation_message&connector_ids=slack" \
  | python3 -m json.tool
```

Supported query params (see `src/dataindex/api/routes/query.py`):

- `entity_types` — repeatable, e.g. `&entity_types=email&entity_types=meeting`
- `connector_ids` — comma-separated
- `contact_ids` — comma-separated
- `date_from`, `date_to` — ISO 8601
- `search_text` — naive text filter (NOT semantic; use `/search` for semantic)
- `parent_id` — e.g. for threaded conversations
- `limit` (default 50), `offset` (default 0)
- `sort_by` (default `timestamp`), `sort_order` (`asc`|`desc`)
- `include_raw_data=true` to include the source payload
- `contact_recent_messages=N` (threaded conversations only) — only return threads
  where the named contact appears in the last N messages

Response shape: `{items, total, page, size, pages, sources_queried, partial_failure, errors}`.

### Semantic search

```bash
curl -sS -X POST http://localhost:42000/dataindex/api/v1/search \
  -H "Content-Type: application/json" \
  -d '{"search_text":"poker dinner plans","limit":5}' \
  | python3 -m json.tool
```

Notes:

- Body field is `search_text` (REST), NOT `query`. The MCP tool uses `query`.
  Mixing them up returns HTTP 422.
- Returns `Chunk` objects (not entities) — chunked content with relevance
  scores. Use the `entity_ids` on each chunk to fetch full entities via
  `/entities/{id}`.
- Uses hybrid dense+sparse retrieval against Qdrant. Quality depends on the
  search ingester having indexed the entities — newly ingested data takes one
  search-sync interval (default 300s) to appear in results.
- Same filters as `/query` are accepted in the JSON body
  (`entity_types`, `connector_ids`, `contact_ids`, `date_from`, `date_to`,
  `parent_id`).
- Optional: `search_modes=["semantic"]` or `["keyword"]`, `rerank: true`.

### Get a specific entity (with raw payload)

```bash
curl -sS "http://localhost:42000/dataindex/api/v1/entities/slack:1777496398.194199?include_raw_data=true" \
  | python3 -m json.tool
```

Note the `:path` converter: colons in IDs are fine.

## MCP usage

Endpoint: `http://localhost:42000/dataindex/mcp/` (note the trailing slash).
Transport: FastMCP streamable-http with SSE responses. Tools defined in
`src/dataindex/mcp.py`:

- `list_connectors()` → list of connector IDs
- `query_entities(...)` → same filters as REST `/query`, returns paginated entities
- `get_entity_by_id(entity_id, include_raw_data=False, max_content_length=4096)`
- `search(query, limit=10, ...)` → semantic search, returns chunks

For programmatic clients use the FastMCP/MCP SDK. For curl, only `initialize`
works in one shot:

```bash
curl -sS -X POST http://localhost:42000/dataindex/mcp/ \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"0.0.0"}}}'
```

Anything beyond `initialize` requires session-ID tracking — fall back to REST
for one-shot inspection.

## Slack ingester (deep dive)

Source: `src/dataindex/ingestion/slack.py`.

### Auth

Slack ingester takes a **user token** (`xoxp-...`) as `slack_token` (required)
and optionally a **bot token** (`xoxb-...`) as `slack_bot_token` (used for bot
DMs only). The user token is what fetches channel history; without it the
connector can't start.

### Sync model

Three concurrent loops driven by the orchestrator:

1. **`initial_sync` / `incremental_sync`** — runs every `sync_interval_seconds`
   (clamped to [30, 3600]; default 120 → 60 in production config).
   - First run with no `last_message_id` → `initial_sync` (last
     `initial_sync_hours`, default 48).
   - Subsequent runs → `incremental_sync` reads only messages newer than
     `state_data.channel_latest_ts` per channel.
2. **`_backfill_historical_data`** — long-running task spawned in `start()`.
   Walks all channels back to `historical_backfill_days` (clamped [1, 365],
   default 90). Yields to incremental via `_backfill_may_run` event.

### Config knobs

| Key | Default | Range | Meaning |
|---|---|---|---|
| `slack_token` | — | required | `xoxp-...` user token |
| `slack_bot_token` | none | optional | `xoxb-...` for bot DM ingestion |
| `sync_interval_seconds` | 120 | 30–3600 | incremental cadence |
| `initial_sync_hours` | 48 | 1–720 | how far back the first incremental looks |
| `historical_backfill_days` | 90 | 1–365 | backfill window |
| `channel_message_keep_count` | 50 | 10–500 | recent_messages preview size on the conversation entity |
| `sync_dms` | `false` | bool | sync user DMs |
| `sync_bot_dms` | `false` | bool | sync DMs with bots (needs `slack_bot_token`) |
| `sync_private_channels` | `true` | bool | include private channels |

### Stuck-state recipes

**Symptom:** logs show `backfill_completed_no_new_channels` immediately on
startup AND `no_sync_state_skipping_incremental`, `entity_count=0` for slack.

**Cause:** `state_data.backfill_completed=true` was persisted but no entities
were stored (e.g. earlier broken run). The current backfill loop marks each
channel "backfilled" as soon as `_fetch_channel_history` returns, even if the
upsert that would follow never happened — so a half-broken run can poison
state.

**Fix:**

```bash
docker exec internalai-dataindex-backend-1 \
  uv run dataindex reset-ingestion --connector slack --force
```

Then verify:

```bash
sleep 30 && curl -sS http://localhost:42000/dataindex/api/v1/connectors/status \
  | python3 -c 'import sys,json; d=json.load(sys.stdin)["connectors"]["slack"]; print(d["sync_status"], d["entity_count"], d["ingester_health"])'
```

Expect entity_count > 0 within a couple of minutes.

**Symptom:** `slack_rate_limited` repeating, sync stalls.

**Cause:** Slack tier-2/3 limits hit. The ingester pauses the whole tier
correctly via `RateLimiter.pause_for`. No action needed unless retries exceed
`SLACK_MAX_RATE_LIMIT_RETRIES` — then check `slack_rate_limit_max_retries_exceeded`
errors and consider raising `sync_interval_seconds`.

**Symptom:** `ingester_health: degraded` for slack but entities still flow.

`auth.test` likely returned ok but `last_sync_at` is older than the threshold,
or rate-limit retries are happening. Check logs:

```bash
docker logs internalai-dataindex-backend-1 --since 5m 2>&1 \
  | grep -E "slack" | tail -50
```

## Email / Gmail (mbsync) ingester

Source: `src/dataindex/ingestion/mbsync_email.py`. Uses `mbsync` to fetch
IMAP into a maildir, then `notmuch` to index, then converts to `EmailEntity`.

### Critical config gotcha

`imap_host` becomes the literal `Host` directive in `/data/email/.mbsyncrc`.
**Do not include the port** — mbsync configures the port separately and will
DNS-resolve `Host imap.gmail.com:993` as a hostname (and fail with
`Cannot resolve server 'imap.gmail.com:993': Name or service not known`).

Correct:

```json
{"imap_host": "imap.gmail.com", "imap_port": "993"}
```

Wrong (DNS-resolution error):

```json
{"imap_host": "imap.gmail.com:993"}
```

If you fix this via `PATCH /api/v1/connectors/email`, the running ingester
regenerates `.mbsyncrc` on its next sync cycle — usually no restart needed.

### Required fields

`imap_host`, `imap_user`, `imap_pass`. Optional: `imap_port` (993),
`imap_tls_type` (`IMAPS`), `folder_patterns` (default `INBOX`),
`folder_sync_mode` (`include`/`exclude`), `folder_subfolders` (`true`/`false`).

### Verifying

```bash
docker exec internalai-dataindex-backend-1 cat /data/email/.mbsyncrc
docker exec internalai-dataindex-backend-1 mbsync -c /data/email/.mbsyncrc -l email
```

`-l` lists channels (dry run); succeeds = config is valid.

## CLI inside the container

All commands accept a connector name and run against the live DB.

```bash
# Status table
docker exec internalai-dataindex-backend-1 uv run dataindex list-connectors

# Trigger immediate sync
docker exec internalai-dataindex-backend-1 uv run dataindex re-ingest slack

# Reset (delete entities + sync state)
docker exec internalai-dataindex-backend-1 \
  uv run dataindex reset-ingestion --connector slack --force

# Reset only backfill markers (keep entities)
docker exec internalai-dataindex-backend-1 \
  uv run dataindex reset-backfill slack --force

# Decrypt a stored secret (debugging only)
docker exec internalai-dataindex-backend-1 uv run dataindex read-password slack
```

If using `docker-compose run` instead, override the entrypoint or it tries to
boot the API server: `docker-compose run --rm --entrypoint "" dataindex-backend ...`.

## Inspecting state directly (Postgres)

```bash
# Sync-state for one connector (state_data is jsonb)
docker exec internalai-dataindex-postgres-1 psql -U dataindex -d dataindex -c \
  "SELECT connector_id, sync_status, last_sync_at, last_message_id,
          LEFT(state_data::text, 300) AS state
     FROM connector_sync_state WHERE connector_id = 'slack';"

# Entity counts per connector + type
docker exec internalai-dataindex-postgres-1 psql -U dataindex -d dataindex -c \
  "SELECT connector_id, entity_type, COUNT(*)
     FROM entities GROUP BY connector_id, entity_type ORDER BY connector_id;"

# Sample N rows from a connector
docker exec internalai-dataindex-postgres-1 psql -U dataindex -d dataindex -c \
  "SELECT id, entity_type, timestamp FROM entities
     WHERE connector_id = 'slack' ORDER BY timestamp DESC LIMIT 5;"
```

Tables: `entities`, `connector_sync_state`, `connector_configs`, `webhooks`,
`webhook_delivery_logs`, `schema_migrations`.

## Reading logs

```bash
# Just the latest activity for one connector
docker logs internalai-dataindex-backend-1 --since 2m 2>&1 \
  | grep -E "ingester=slack|connector_id=slack" | tail -40

# Errors only
docker logs internalai-dataindex-backend-1 2>&1 \
  | grep -E "\[error|\[warning" | tail -30
```

Structured-log keys to scan for:

- `mbsync_failed`, `mbsync_stderr` — email/IMAP problems
- `backfill_started`, `backfill_completed_no_new_channels`, `new_channels_detected`
  — slack backfill state transitions
- `no_sync_state_skipping_incremental` — first incremental run, expected once
- `slack_rate_limited`, `slack_rate_limit_max_retries_exceeded`
- `sync_state_upserted` — heartbeat that the ingester completed a cycle
- `ingester_loop_started` / `ingester_loop_cancelled` — lifecycle

## Default operating procedure

When the user says "X connector isn't ingesting / is broken / is stuck":

1. `connectors/status` → check `entity_count`, `ingester_health`, `error`.
2. Tail logs filtered for that connector (last 5 min).
3. Inspect `connector_sync_state.state_data` — many bugs leave bogus markers.
4. Cross-check `connector_configs.config` for typos (the IMAP host gotcha is
   a recurring one).
5. If state is corrupt but data was real, prefer `reset-backfill` over
   `reset-ingestion` to avoid re-downloading.

When the user asks "find / search for X":

- For exact filtering by attributes (date, contact, connector, type) → REST
  `/query`.
- For natural-language / semantic queries → REST `/search` (or MCP `search`).
- For threaded conversations belonging to a contact → `/query` with
  `entity_types=threaded_conversation&contact_ids=<id>&contact_recent_messages=50`.

## Adding a new ingester

1. Implement in `src/dataindex/ingestion/<name>.py` with the contract:
   `name`, `sync_interval_seconds`, `initial_sync()`, `incremental_sync()`,
   `health_check()` (optional `start`, `stop`, `reset`).
2. Register the type in the `INGESTER_TYPE_TO_CLASS` mapping
   (`src/dataindex/api/main.py`).
3. Add required-fields/sensitive-fields entries to
   `src/dataindex/api/routes/config.py` (`REQUIRED_FIELDS`, `SENSITIVE_FIELDS`).
4. Tests under `tests/ingesters/test_<name>.py`. Run from `apps/dataindex/`:
   ```bash
   cd apps/dataindex && uv run -m pytest tests/ingesters/test_<name>.py -v
   ```
