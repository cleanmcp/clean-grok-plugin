# Notifications and smart replies (pending tools)

Status: **not live yet.** Every tool on this page depends on an open pull request in Clean's server repo (cleanmcp/Mono-icarus). Each section names the PR it depends on. A tool is usable only after its PR merges and deploys; until then the hosted server does not list it. The hosted server stays the source of truth for names and schemas.

| Tool | API route | Depends on |
| --- | --- | --- |
| `get_notifications` | `GET /v1/w/:workspaceId/notifications/events` | [Mono-icarus #1071](https://github.com/cleanmcp/Mono-icarus/pull/1071) |
| `smart_reply_list` | `GET /v1/w/:workspaceId/smart-replies` | [Mono-icarus #1069](https://github.com/cleanmcp/Mono-icarus/pull/1069) |
| `smart_reply_draft` | `POST /v1/w/:workspaceId/smart-replies/:conversationId/draft` | [Mono-icarus #1069](https://github.com/cleanmcp/Mono-icarus/pull/1069) |
| `smart_reply_approve_send` | `POST /v1/w/:workspaceId/smart-replies/:conversationId/send` | [Mono-icarus #1069](https://github.com/cleanmcp/Mono-icarus/pull/1069) |
| Webhooks | coming | Child #74 PR, not open yet |
| Remaining parity tools | coming | Child #75 PR, not open yet |

Every MCP tool here calls the API route beside it with the caller's own credential, so the API and MCP behave the same: same checks, same refusals, same audit. Nothing new is needed to connect: the existing Clean sign in (OAuth) or workspace API key reaches these routes.

## How refusals look

Over the API a refusal is an HTTP error with this body:

```json
{ "error": { "class": "conflict", "domain": "outreach", "reason": "SMART_REPLY_STALE" } }
```

`class` sets the HTTP status: `invalid` 400, `forbidden` 403, `not_found` 404, `conflict` 409, `rate_limited` 429. A request that fails its input schema is 400 with reason `REQUEST_INVALID`.

Over MCP the same refusal comes back as a tool error whose text starts with `Clean refused:` followed by the reason (for example `Clean refused: SMART_REPLY_STALE`). Switch on the reason, not the wording.

## get_notifications

Depends on [Mono-icarus #1071](https://github.com/cleanmcp/Mono-icarus/pull/1071).

Reads the workspace event log: every event Clean sends to the workspace's Slack channel or to a linked member in Slack (reply alerts, digests, campaign reports, account alerts, research questions, chat answers and cards), in order, after a cursor. An agent that polls this gets the same events Slack gets, so it can ping the user the same way the Slack bot does.

### Input

| MCP field | API query | Type | Notes |
| --- | --- | --- | --- |
| `since_cursor` | `since` | string of digits | The `nextCursor` from the last call. Omit (or `0`) to read from the first event. |
| `kinds` | `kinds` | MCP: array; API: comma list | Only these kinds. Omit for all. Unknown kind is a 400. |
| `limit` | `limit` | integer 1 to 500 | Default 100. |

Role: any workspace member, viewer and up. Read-only credentials work.

### Output

```json
{
  "schemaVersion": 1,
  "events": [
    {
      "eventId": "uuid",
      "cursor": "42",
      "kind": "channel_reply",
      "audience": "workspace",
      "occurredAt": "2026-11-10T07:00:00.000Z",
      "payload": { "version": 1, "conversationId": "uuid", "replySource": "linkedin" }
    }
  ],
  "hasMore": false,
  "nextCursor": "42"
}
```

The API route sends `Cache-Control: private, no-store`.

### Cursor rules

- Keep `nextCursor` and pass it back as `since_cursor` on the next call. Nothing is missed and nothing is repeated.
- Each event's number is set when its write commits, with no gaps, per workspace, in commit order. So once you have read past number N, no event with a lower number can appear later.
- When nothing new has committed, `nextCursor` comes back unchanged and `events` is empty.
- When `hasMore` is true, call again right away with the new `nextCursor`.
- Each event also carries its own `cursor`. If you stop partway through a page, save the `cursor` of the last event you finished.

### Dedupe

Dedupe on `eventId`. The server never writes a second event for the same underlying Slack delivery (a replayed Slack event writes nothing). If your agent crashes after acting on an event but before saving the cursor, it reads that event again on restart; a set of seen `eventId` values stops a second ping.

### Kinds

`agent_reply`, `link_prompt`, `notice`, `channel_reply`, `connect_channel`, `digest`, `proposal_card`, `proposal_result`, `account_alert`, `campaign_report`, `signal_questions`, `smart_reply_draft`, `reminder`.

The first eleven are the Slack delivery kinds and are written for every Slack delivery. `smart_reply_draft` and `reminder` are reserved: the PR adds them to the list, but no code in that PR writes them yet.

Channel-wide kinds (`channel_reply`, `connect_channel`, `digest`, `account_alert`, `campaign_report`, `signal_questions`) have `audience: "workspace"` and every member sees them. The other kinds have `audience: "member"` and only the linked Clean member they were for sees them.

### Payload: references only

The payload never holds message text, the lead's words, names or Slack ids. It holds only these optional fields, plus `version: 1`:

`runId`, `conversationId`, `subjectId`, `replySource` (`linkedin`, `instantly`, `heyreach`, `dripify`, `gmail`), `sourceOccurredAt`, `noticeCode`, `outcome`, `stepIndex`.

To show who replied and what they said, follow up with `inbox_read` (or `conversations_read`), which applies the workspace's sharing rules.

### Refusals

| Reason | When |
| --- | --- |
| `REQUEST_INVALID` (400) | Bad cursor, unknown kind, limit out of range |
| 404 | The caller is not a member of the workspace |

### Polling loop (example)

```
cursor = load() or "0"
seen = load_seen_event_ids()
loop every few minutes:
  page = get_notifications(since_cursor=cursor, limit=100)
  for event in page.events:
    if event.eventId not in seen:
      handle(event)        # for a reply alert, read details with inbox_read
      seen.add(event.eventId)
    cursor = event.cursor
    save(cursor, seen)
  if page.hasMore: continue right away
```

## Smart replies

Depends on [Mono-icarus #1069](https://github.com/cleanmcp/Mono-icarus/pull/1069).

Smart replies answer a lead's newest message in a shared LinkedIn thread. Clean writes the draft; a person reads the exact text and approves it; only then does it send, through the same send the Team reply box and the chat Send card use. Nothing sends on its own.

Role: workspace member (viewer is refused). Drafting and sending need a full-scope credential; a read-only credential reaches only `smart_reply_list`.

### smart_reply_list

Shared LinkedIn threads where the lead wrote last, newest first, each with your open draft or `null`.

| MCP field | API query | Type | Notes |
| --- | --- | --- | --- |
| `cursor` | `cursor` | string | `nextCursor` from the last page. Omit for the first page. A cursor this list did not return is a 400. |
| `limit` | `limit` | integer 1 to 50 | Default 20. |

```json
{
  "nextCursor": "opaque string or null",
  "replies": [
    {
      "conversationId": "uuid",
      "draft": { "draftId": "uuid", "body": "...", "bodySha256": "hex", "conversationId": "uuid", "expiresAt": "iso", "inboundMessageId": "uuid" },
      "lastMessage": { "deliveredAt": "iso", "direction": "inbound", "isTruncated": false, "preview": "..." },
      "lead": { "name": "...", "headline": "...", "leadId": "uuid" },
      "sender": { "isMe": true, "name": "...", "userId": "..." }
    }
  ]
}
```

`draft` shows only your own draft that has not expired. It never includes the approval token. `nextCursor` is `null` on the last page.

### smart_reply_draft

Input: `conversationId` (uuid, from `smart_reply_list`). Writes a draft in the sender's voice and never sends. Each call is a model call; a caller may draft at most 20 a minute.

Response (HTTP 201):

```json
{
  "draft": {
    "draftId": "uuid",
    "body": "the exact text",
    "bodySha256": "64 hex chars",
    "approvalToken": "one-time token",
    "conversationId": "uuid",
    "expiresAt": "iso, 60 minutes after drafting",
    "inboundMessageId": "uuid"
  },
  "sender": { "isMe": false, "name": "...", "userId": "..." },
  "lead": { "name": "..." }
}
```

- **The approval token is returned once.** Clean keeps only its hash. No list or read returns it again. If you lose it, draft again.
- Drafting again on the same thread replaces the open draft there; the old one can no longer be sent (`SMART_REPLY_DRAFT_CHANGED`).
- The text has already passed Clean's outbound dash rule (no em dashes) before it is stored, hashed and returned, so what you show, what you hash and what sends are the same.
- A draft expires 60 minutes after it is written.

### smart_reply_approve_send

Sends one draft the person approved, unchanged.

| Field | Type | Notes |
| --- | --- | --- |
| `conversationId` | uuid | Same thread as the draft (API: in the path). |
| `draftId` | uuid | From `smart_reply_draft`. |
| `approvalToken` | string, 20 to 200 chars | From `smart_reply_draft`. |
| `textSha256` | 64 hex chars | SHA-256 of the exact text the person approved, UTF-8 bytes, hex. Upper or lower case both work. |
| `idempotencyKey` | string, 8 to 200 chars | Pick one per approval and reuse it on every retry of that approval. |

Response (HTTP 202):

```json
{ "draftId": "uuid", "requestId": "uuid", "state": "sent | failed | queued", "isReplay": false }
```

How to compute `textSha256`: hash the `body` string exactly as `smart_reply_draft` returned it, encoded as UTF-8, with no trimming or edits. For example in Python `hashlib.sha256(body.encode("utf-8")).hexdigest()`. It must equal `bodySha256`; if the person edited the text, the hashes differ and the send is refused. To send different words, draft again.

Retries: calling again with the same `idempotencyKey` returns the same send (`isReplay: true`) and sends nothing twice. A different key for a draft already sent is refused with `SMART_REPLY_ALREADY_SENT`. At most one smart reply is ever sent per lead message.

Who may send: only the member who drafted it, with their own credential. Clean staff signed in as a client cannot send (`STAFF_CANNOT_CONFIRM_SENDS`).

Approval rule for agents: always show the person the exact `body` and wait for a clear yes before calling `smart_reply_approve_send`. Never call it on your own judgment.

### Smart reply refusals

| Reason | Status | When | What to do |
| --- | --- | --- | --- |
| `LINKEDIN_CONVERSATION_NOT_FOUND` | 404 | Thread does not exist or you cannot see it | Pick another thread |
| `LINKEDIN_REPLY_NOT_ALLOWED` | 403 | You cannot reply from this thread's LinkedIn seat | Tell the person |
| `SMART_REPLY_LEAD_OPTED_OUT` | 409 | The lead said no or asked to be left alone anywhere in the thread | Do not reply |
| `SMART_REPLY_NOTHING_TO_ANSWER` | 409 | The newest message is not from the lead | Nothing to do |
| `SMART_REPLY_THREAD_CHANGED` | 409 | A newer message arrived, or a reply was already sent, since the draft | Draft again |
| `SMART_REPLY_THREAD_TOO_LONG` | 409 | Thread longer than 2,000 messages | Reply in Clean |
| `SMART_REPLY_RATE_LIMITED` | 429 | More than 20 drafts a minute (retryable) | Wait, then retry |
| `SMART_REPLY_DRAFT_EMPTY` | 400 | The written draft was empty or too long | Draft again |
| `SMART_REPLY_DRAFT_NOT_FOUND` | 404 | Wrong draft id, wrong thread, or someone else's draft | Check the ids |
| `SMART_REPLY_TOKEN_INVALID` | 403 | Wrong approval token | Draft again |
| `SMART_REPLY_TEXT_CHANGED` | 409 | `textSha256` does not match the draft text | Hash the exact body, or draft again |
| `SMART_REPLY_DRAFT_CHANGED` | 409 | The draft was replaced by a newer draft | Use the newest draft |
| `SMART_REPLY_STALE` | 409 | The draft is over 60 minutes old | Draft again and re-confirm |
| `SMART_REPLY_ALREADY_SENT` | 409 | Already sent with another idempotency key, or this lead message already got a smart reply | Do not resend |
| `STAFF_CANNOT_CONFIRM_SENDS` | 403 | Staff signed in as the client | The client member sends |
| `REQUEST_INVALID` | 400 | Input failed the schema | Fix the input |

## Coming

These are not documented yet because their pull requests are not open. This section is filled in from each PR's diff once its number is known.

- **Webhooks** (Child #74): push delivery of the same events as `get_notifications`. PR to come.
- **Remaining parity tools** (Child #75): the rest of the Slack bot's functions over the API and MCP. PR to come.
