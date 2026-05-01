---
name: dataindex-connectors
description: >
  Use the per-Greybox DataIndex + ContactDB MCP servers to query unified
  data (emails, calendar events, meetings, chat, documents) and resolve
  people to contact_ids. Apply when the user asks to find/search/list/inspect/
  summarize/count emails, meetings, calendar events, chat conversations,
  or documents in their Greybox; or to look up a person, get their contact
  ID, or filter data by who's involved.
---

# DataIndex + ContactDB MCP

A greybox-cloud Greybox exposes two MCP servers:

- **`dataindex`** — unified query interface over emails, meetings, calendar events, chat conversations, documents, and webpage history. Every piece of data is an **entity** with shared base fields plus type-specific ones.
- **`contactdb`** — the people directory. Every person across all data sources resolves to a single `contact_id`. Use it to convert names/emails into IDs that filter DataIndex queries.

Both are reached through the user's per-Greybox MCP install (e.g. `gb-XXXXXX-dataindex`, `gb-XXXXXX-contactdb`). Tool names below are unqualified — apply them via whichever MCP namespace your client surfaces.

## Tools at a glance

### DataIndex MCP

| Tool | When to use |
|---|---|
| `list_connectors` | Discover which connectors (and therefore entity types) are actually configured in this Greybox. Call before assuming a connector exists. |
| `query_entities` | **Exhaustive filtered enumeration** with pagination. Use when the user wants *all* matching entities ("list every email from Alice this week", "all meetings in Q1", counts). |
| `search` | **Semantic/hybrid search** — ranked relevance, no pagination. Use for natural-language questions ("what was discussed about hiring", "find anything about the product roadmap"). |
| `get_entity_by_id` | Fetch one entity in full detail. ID format: `connector_name:native_id`. |

### ContactDB MCP

| Tool | When to use |
|---|---|
| `get_me` | Get the operator's own `contact_id`. **Call this first** when the user references themselves ("meetings I attended", "emails I sent"). |
| `query_contacts` | Search/filter people by name, hotness score (0–100 engagement metric), platform, or last-interaction window. |
| `get_contact_by_id` | Fetch one contact's full record by numeric ID. |

## Required sequence

Follow this order unless the user gives a constraint that requires otherwise:

1. **Resolve people first.** Named person → `query_contacts(search=...)`. Self-reference ("me/my") → `get_me`. Never guess a `contact_id`.
2. **Verify connectors.** Call `list_connectors` once per request flow; confirm the source you need is configured.
3. **Choose query mode.** Exhaustive listing/counting → `query_entities`. Natural-language relevance → `search`.
4. **Fetch full detail only when asked.** `get_entity_by_id`; if content is truncated, pass `max_content_length=null`.
5. **Check response integrity.** Inspect `partial_failure` and `errors` before claiming "no results".

## Routing decisions

**`search` vs `query_entities`:** "find/list/every/all/count X" → `query_entities`. "what about / tell me about / what was said / discussed" → `search`. When unsure on a natural-language question, prefer `search` — it's faster and better-ranked.

**Resolving people first:** if the user mentions someone by name, **first** call `query_contacts(search="...")` to get the contact_id, then pass that ID into DataIndex via the `contact_ids` filter.

**Self-references:** "my meetings", "emails I got", "calls I joined" → `get_me` first, then filter by your own `contact_id`.

**Keyword → workflow** (see workflows below):

| Trigger words | Workflow |
|---|---|
| "during", "while", "at the same time", "in that meeting" | Anchor + Interval Correlation |
| "after", "follow-up", "later", "what happened next" | Anchor + Follow-up Window |
| "everything X sent", "timeline", "across tools", "all activity" | Person-Centric Timeline |
| "about X", "what do we know", "mentions of", "discussion on" | Topic-Centric Evidence Pack |
| "summarize meeting", "recap with updates", "summary plus follow-up" | Narrative Reconstruction |
| "did we do", "was this implemented", "follow-through" | Decision-to-Execution Trace |
| "was this answered", "resolved?", "open question" | Question Resolution Tracker |
| "context around this", "full thread", "what led to this" | Entity Neighborhood Expansion |

If multiple match, prefer (1) user-stated intent over keyword frequency, (2) the narrowest workflow that answers the question, (3) Narrative Reconstruction for mixed "during + after + summarize" requests.

## Workflows

Pick the smallest workflow that fits. Each is a recipe — combine `search` / `query_entities` / `get_entity_by_id` per the steps.

1. **Anchor + Interval Correlation** — *"what happened during X"*. Pick anchors (often meetings) → derive time windows (with `sync_buffer_minutes`) → `query_entities` for target types within each window → group by anchor.

2. **Anchor + Follow-up Window** — *"what happened after X"*. From each anchor's end time, query the next `followup_window_hours` for related entities → rank by relevance to the anchor topic/participants.

3. **Person-Centric Timeline** — *"everything X sent/discussed this week"*. Resolve contacts → `query_entities(contact_ids=…, entity_types=…, date_from/to=…)` paginated → sort chronologically.

4. **Topic-Centric Evidence Pack** — *"what do we know about Y"*. Seed with `search(query=topic, …)`. If the user wants exhaustive coverage, follow with `query_entities(search=topic_terms, …)` paginated and dedupe.

5. **Narrative Reconstruction (meeting + follow-ups)** — anchor meeting → "during" entities (window ± buffer) → "after" entities (post-window). Synthesize: decisions, action items, unresolved points, later changes.

6. **Decision-to-Execution Trace** — *"did we ship what we agreed"*. Extract decisions from anchor → `search` each decision over `[anchor.start, anchor.end + followup_window]` → map decision → evidence.

7. **Question Resolution Tracker** — extract open questions from anchor → `search` each over the post-window → classify resolved / partial / unresolved.

8. **Entity Neighborhood Expansion** — seed = `get_entity_by_id`. Expand via `parent_id`, `contact_ids` overlap within ±Δ time, and a `search` over key terms in the same window. Rank context.

### Standard defaults

- `sync_buffer_minutes`: 5
- `followup_window_hours`: 72
- `sort_by`: `timestamp`, `sort_order`: `desc` (unless user asks otherwise)
- `limit`: start at 20–50; paginate for exhaustive requests

## Entity types

All entities share these base fields:

| Field | Type | Notes |
|---|---|---|
| `id` | string | Format: `connector_name:native_id` |
| `entity_type` | string | One of the types below |
| `timestamp` | datetime | When the entity occurred |
| `contact_ids` | string[] | ContactDB IDs of people involved |
| `connector_id` | string | Which connector produced this |
| `title` | string? | Display title |
| `parent_id` | string? | Parent (e.g. thread for a message) |
| `raw_data` | dict | Original source data (excluded by default; pass `include_raw_data=true` to include) |

### `email`

From mbsync/IMAP sync.

| Field | Type | Notes |
|---|---|---|
| `thread_id` | string? | Email thread grouping |
| `text_content` | string? | Plain text body |
| `html_content` | string? | HTML body |
| `snippet` | string? | Preview snippet |
| `from_contact_id` | string? | Sender's contact_id |
| `to_contact_ids` | string[] | Recipient contact_ids |
| `cc_contact_ids` | string[] | CC contact_ids |
| `has_attachments` | bool | |
| `attachments` | dict[] | Attachment metadata |

### `calendar_event`

From ICS calendar feeds.

| Field | Type | Notes |
|---|---|---|
| `start_time` / `end_time` | datetime? | |
| `all_day` | bool | |
| `description` | string? | |
| `location` | string? | |
| `attendees` | dict[] | |
| `organizer_contact_id` | string? | |
| `status` | string? | |
| `calendar_name` | string? | Source calendar |
| `meeting_url` | string? | Video call link |

### `meeting`

From Reflector (recorded meetings + transcripts).

| Field | Type | Notes |
|---|---|---|
| `start_time` / `end_time` | datetime? | |
| `participants` | MeetingParticipant[] | `display_name`, `contact_id?`, `platform_user_id?`, `email?`, `speaker?` |
| `meeting_platform` | string? | e.g. `"jitsi"` |
| `transcript` | string? | Full speaker-diarized transcript |
| `summary` | string? | AI-generated summary |
| `meeting_url` / `recording_url` | string? | |
| `location` / `room_name` | string? | `room_name` often encodes location (e.g. `standup-office-bogota`); fall back to it when `location` is null |

> **Reflector participant coverage is incomplete.** Reflector only sees logged-in users.
> - `contact_ids` is a **subset** of actual attendees — only those resolved to a known contact.
> - `participants` is more complete but still misses anyone Reflector didn't detect.
> - `participant.contact_id` may be `null` if detected but unmatched.
>
> **Consequence:** filtering meetings by `contact_ids` will miss meetings someone attended but wasn't logged in for. To improve coverage, combine: (1) filter by `contact_ids`, (2) also `search` the transcript/summary by name.

### `conversation` / `conversation_message` / `threaded_conversation`

From Zulip / Slack / Babelfish / Notion (comments).

- `conversation` — a stream/channel with `recent_messages: dict[]`.
- `conversation_message` — single message with `message: string?` and `mentioned_contact_ids: string[]`. Notion comments use this type with `parent_id` pointing at the parent Notion page (`document`).
- `threaded_conversation` — a topic thread under a stream with `recent_messages: dict[]`.

For "discussions about X" use `threaded_conversation` + `search`. For "messages mentioning person Y" use `conversation_message` filtered by `contact_ids`.

### `document`

From HedgeDoc, API ingestion, Notion pages, Float Financial records, etc. Fields: `content`, `description`, `mimetype`, `url`, `revision_id`. Prefer `search` over `query_entities`-with-text-filter for body-content matching.

- **Notion pages** are rendered to markdown (`mimetype="text/markdown"`), have a `url` back to notion.so, and may set `parent_id` to another Notion page when nested.
- **Float Financial records** are flattened to `text/plain` documents, one per record across `card-transactions`, `account-transactions`, `bills`, `payments`, `receipts`. `connector_metadata.resource_type` distinguishes them; raw fields live in `raw_data`. Filter to a specific kind by combining `connector_ids=["float_financial"]` with a `search` substring or post-filter on `connector_metadata.resource_type`.

### `webpage`

From browser-history extension. Fields: `url`, `visit_time`, `text_content`.

### `contact`

Contacts mirrored from ContactDB into DataIndex. **Read-only mirror** — for contact operations use the ContactDB MCP directly (`query_contacts`, `get_contact_by_id`, `get_me`).

## Connectors → entity types

`list_connectors` returns IDs; not every Greybox has all of them. Common mappings:

| Connector ID | Produces | Notes |
|---|---|---|
| `mbsync_email` | `email` | IMAP sync. Filter by `from_contact_id` / `to_contact_ids` via the `contact_ids` filter. |
| `ics_calendar` | `calendar_event` | Multiple feeds may exist as separate connectors (e.g. `personal_calendar`, `work_calendar`). |
| `reflector` | `meeting` | Transcripts + summaries; see participant-coverage caveat above. |
| `zulip` | `conversation`, `conversation_message`, `threaded_conversation` | |
| `slack` | `conversation`, `conversation_message`, `threaded_conversation` | |
| `babelfish` | `conversation_message`, `threaded_conversation` | Translated cross-language chat. Query alongside `zulip` / `slack` for full coverage. |
| `hedgedoc` | `document` | Use `search` for body content, not `query_entities` text filter. |
| `api_document` | `document` | API-ingested documents (uploads, etc). |
| `notion` | `document`, `conversation_message` | Pages → markdown documents (`mimetype="text/markdown"`); comments → `conversation_message` with `parent_id` set to the page. ID forms: `notion:page:<uuid>`, `notion:comment:<uuid>`. |
| `float_financial` | `document` | Corporate financial records (card/account transactions, bills, payments, receipts) flattened into documents. Resource kind in `connector_metadata.resource_type`; raw API fields in `raw_data`. ID form: `float_financial:<resource_type>:<id>`. |
| `browser_history` | `webpage` | |
| `contactdb` | `contact` | Read-only mirror. Use ContactDB MCP for actual contact operations. |

## query_entities — key parameters

| Param | Type | Notes |
|---|---|---|
| `entity_types` | string\|list | E.g. `["email", "meeting"]`. |
| `contact_ids` | int\|list | Filter to entities involving these contacts. |
| `connector_ids` | string\|list | Filter to specific connectors. Discover via `list_connectors`. |
| `date_from` / `date_to` | ISO string | UTC if no timezone. |
| `search` | string | Substring filter on content fields. **Not** semantic — use the `search` tool for that. |
| `parent_id` | string | E.g. messages within a thread. |
| `limit` / `offset` | int | Paginate; `limit` 1–100, default 50. Loop until `offset >= total`. |
| `sort_by` / `sort_order` | string | Default `timestamp` / `desc`. |
| `include_raw_data` | bool | Default false. Only include when the user wants original-source detail. |
| `max_content_length` | int | Default 1024 — content fields auto-truncate beyond this. Pass `null` to disable. |

Response shape: `{items, total, page, size, pages, sources_queried, partial_failure, errors}`. Always check `partial_failure` and `errors` before claiming "no results".

## search — key parameters

| Param | Type | Notes |
|---|---|---|
| `query` | string | The natural-language question. |
| `limit` | int | 1–100, default 10. **No pagination** — set higher if you need more. |
| `entity_types` / `connector_ids` / `contact_ids` / `date_from` / `date_to` / `parent_id` | — | Same semantics as `query_entities`. |

Response shape: `{results: chunk[], total_count}`. Each chunk has `entity_ids`, `entity_type`, `connector_id`, `content`, `timestamp`. To get the full source, follow up with `get_entity_by_id(entity_ids[0])`.

## query_contacts — key parameters

| Param | Type | Notes |
|---|---|---|
| `search` | string | Matches name + bio. The most common entry point. |
| `sort_by` | string | `"hotness"` (engagement, 0–100), `"name"`, or `"updated_at"`. |
| `min_hotness` / `max_hotness` | float | 0–100 composite engagement score. |
| `platforms` | list | Contacts present on **all** specified platforms (AND). |
| `is_placeholder` / `is_service_account` | bool | Stubs / no-reply bots. |
| `last_interaction_from` / `last_interaction_to` | ISO string | |
| `limit` / `offset` | int | Default 50, max 100. |

## Common recipes

| Question | Approach |
|---|---|
| "Emails from Alice this week" | `query_contacts(search="Alice")` → grab `id` → `query_entities(entity_types=["email"], contact_ids=[id], date_from=…)` |
| "Meetings I attended" | `get_me` → `query_entities(entity_types=["meeting"], contact_ids=[my_id])` (then double-check via name search per the Reflector caveat) |
| "What was discussed about the roadmap" | `search(query="product roadmap decisions", entity_types=["meeting","threaded_conversation","email"])` |
| "Active contacts I haven't talked to recently" | `query_contacts(min_hotness=50, last_interaction_to=…, sort_by="hotness")` |
| "All Zulip/Slack threads about hiring" | `search(query="hiring", entity_types=["threaded_conversation"], connector_ids=["zulip","slack"])` |
| "Upcoming calendar events" | `query_entities(entity_types=["calendar_event"], date_from=now, sort_order="asc")` |
| "Show me the full email" | get the `id` from a search/query result, then `get_entity_by_id(id, max_content_length=null)` |
| "Notion pages about onboarding" | `search(query="onboarding", entity_types=["document"], connector_ids=["notion"])` |
| "Comments on a Notion page" | `query_entities(entity_types=["conversation_message"], parent_id="notion:page:<uuid>")` |
| "Float card transactions this month" | `query_entities(entity_types=["document"], connector_ids=["float_financial"], date_from=…, search="card-transactions")` then post-filter on `connector_metadata.resource_type` if needed |
| "All Float spend by Alice" | `query_contacts(search="Alice")` → `query_entities(entity_types=["document"], connector_ids=["float_financial"], contact_ids=[id])` |

## Correlating across sources

When stitching results from different connectors, weight evidence by:

- temporal overlap / proximity
- participant overlap (`contact_ids`)
- mention overlap (`mentioned_contact_ids`)
- thread / `parent_id` linkage
- semantic similarity to the anchor topic

Confidence:

- **High** — strong temporal overlap **plus** at least one identity/linkage signal
- **Medium** — temporal proximity **plus** semantic relevance
- **Low** — semantic relevance only, no identity linkage

## Output contract

Unless the user asks for a raw dump, include in the reply:

1. Matched person/contact used (or note none)
2. Workflow used (from the catalog)
3. Tool mode(s) used (`search`, `query_entities`, `get_entity_by_id`)
4. Filters/windows used (entity types, connectors, date range, contact scope)
5. Result count and notable caveats (truncation, `partial_failure`, missing connector coverage)

Keep results concise.

## Conventions and failure handling

- ID prefixes are stable: `mbsync_email:...`, `reflector:...`, `zulip:...`, `slack:...`, `notion:page:...` / `notion:comment:...`, `float_financial:<resource_type>:...`, etc. You can predict them from the connector ID.
- All dates are ISO-8601; treat naive datetimes as UTC.
- Content fields (`text_content`, `transcript`, `summary`, `description`, `message`, `html_content`) auto-truncate at `max_content_length`. When the user asks for the full body, pass `max_content_length=null` (`query_entities` / `get_entity_by_id`).
- The `contactdb` connector pseudo-ID inside DataIndex only mirrors contacts for unified search; **don't** use it for contact operations. Use the ContactDB MCP instead.
- Reflector data is **incomplete by design** — always layer a transcript `search` on top of `contact_ids` filtering when the user is asking about meetings involving a person.
- If multiple contacts plausibly match a name, ask the user to disambiguate before deep querying.
- If an expected connector is missing, say so and suggest available alternatives from `list_connectors`.
- If zero results, report the filters used and suggest the next broadening step (wider window, drop a filter, switch `search`↔`query_entities`).
- Never fabricate content — only report retrieved data.
