# Notifications and smart replies (pending tools)

Status: **not live yet.** Every tool on this page depends on an open pull request in Clean's server repo (cleanmcp/Mono-icarus). Each section names the PR it depends on. A tool is usable only after its PR merges and deploys; until then the hosted server does not list it. The hosted server stays the source of truth for names and schemas.

| Tool | API route | Depends on |
| --- | --- | --- |
| `get_notifications` | `GET /v1/w/:workspaceId/notifications/events` | [Mono-icarus #1071](https://github.com/cleanmcp/Mono-icarus/pull/1071) |
| `smart_reply_list` | `GET /v1/w/:workspaceId/smart-replies` | [Mono-icarus #1069](https://github.com/cleanmcp/Mono-icarus/pull/1069) |
| `smart_reply_draft` | `POST /v1/w/:workspaceId/smart-replies/:conversationId/draft` | [Mono-icarus #1069](https://github.com/cleanmcp/Mono-icarus/pull/1069) |
| `smart_reply_approve_send` | `POST /v1/w/:workspaceId/smart-replies/:conversationId/send` | [Mono-icarus #1069](https://github.com/cleanmcp/Mono-icarus/pull/1069) |
| `channel_posts_read` | `GET /v1/w/:workspaceId/channel-posts/{digest,campaign-report,account-alerts,signal-questions}` | [Mono-icarus #1072](https://github.com/cleanmcp/Mono-icarus/pull/1072) |
| Webhook subscriptions | `POST`, `GET /v1/w/:workspaceId/webhooks`, `DELETE .../webhooks/:subscriptionId`, `GET .../deliveries`, `POST .../replay` | [Mono-icarus #1073](https://github.com/cleanmcp/Mono-icarus/pull/1073) (built on #1071) |
| `ack_notifications` | `POST /v1/w/:workspaceId/webhooks/:subscriptionId/ack` | [Mono-icarus #1073](https://github.com/cleanmcp/Mono-icarus/pull/1073) |

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

## channel_posts_read

Depends on [Mono-icarus #1072](https://github.com/cleanmcp/Mono-icarus/pull/1072).

Reads what Clean's Slack channel posts, built by the same code that builds the Slack post, for a workspace with or without Slack installed. Use it after a `get_notifications` event of kind `digest`, `campaign_report`, `account_alert` or `signal_questions` to show the person the full post.

Role: any workspace member, viewer and up. Read-only credentials work. Responses send `Cache-Control: private, no-store`.

| MCP `kind` | API route | Inputs | Response |
| --- | --- | --- | --- |
| `digest` | `GET .../channel-posts/digest` | `timeZone` (IANA name, default `UTC`) | `{ "schemaVersion": 1, "digest": {...} or null }` |
| `campaign_report` | `GET .../channel-posts/campaign-report` | `campaignId` (uuid, required), `provider` (`clean` or `gmail`, default `clean`), `timeZone` | `{ "schemaVersion": 1, "report": {...} or null }` |
| `account_alerts` | `GET .../channel-posts/account-alerts` | none | `{ "schemaVersion": 1, "alerts": [...] }` |
| `signal_questions` | `GET .../channel-posts/signal-questions` | `signalPlanId` (uuid, optional; omit for all open research) | `{ "schemaVersion": 1, "research": [...] }` |

What each one returns. These are the exact types the #1072 readers return (`record/slack/channel-posts.ts`); the object types themselves live on main in `source/slack/blocks.ts`, the same ones the Slack post is drawn from.

`digest` (the weekday Today post; `null` on a day with nothing to say), type `SlackDigest`:

```json
{
  "dayLabel": "string",
  "openUrl": "string",
  "stats": [ { "label": "string", "value": "string" } ],
  "rows": [ { "text": "string", "url": "string" } ]
}
```

`stats` are the week's numbers; `rows` are the things that need someone, each with a link.

`report` (one campaign's report once all its leads are done), type `SlackCampaignReport`:

```json
{
  "campaignName": "string",
  "finish": "completed | through_everyone",
  "openUrl": "string",
  "ran": "string or null",
  "stats": [ { "label": "string", "value": "string" } ]
}
```

`null` while leads still run, when the campaign is not active or completed, and for a campaign id Clean cannot find.

`alerts` (every LinkedIn account that needs reconnecting, and every one disconnected in the last 7 days; at most 50, most recently changed first), each of type `SlackAccountAlertMessage`:

```json
{
  "accountName": "string or null",
  "alert": "linkedin_disconnected | linkedin_reconnect_required",
  "memberName": "string",
  "settingsUrl": "string"
}
```

`accountName` is the LinkedIn profile's name, `null` when Clean did not capture it; `memberName` is the Clean member who connected the account. The type also allows `alert: "linkedin_held"` with a `heldUntil` time, but this read never returns it: a hold is not stored, so it shows only in the Slack post.

`research` (open questions from current research, newest first; empty when none stand), each of type `SlackSignalQuestions` plus `signalPlanId`:

```json
{
  "signalPlanId": "uuid",
  "icpName": "string",
  "openUrl": "string",
  "questions": [ { "question": "string", "assumption": "string" } ]
}
```

`assumption` is what Clean assumes until the question is answered. Answer in the chat `openUrl` names.

A bad `timeZone`, a missing `campaignId` for `campaign_report`, or a non uuid id is a 400 `REQUEST_INVALID`.

## Webhooks

Depends on [Mono-icarus #1073](https://github.com/cleanmcp/Mono-icarus/pull/1073), which is built on #1071. Webhooks push the same events `get_notifications` reads, so an agent does not have to poll.

Role: workspace admin for every webhook route, by session, API key or OAuth token.

### Manage subscriptions

| Route | Body or query | Response |
| --- | --- | --- |
| `POST /v1/w/:workspaceId/webhooks` | `{ "url": "https://...", "kinds": [...] }` (`kinds` optional, at least one kind if given; omit for all) | 201, see below. The only response that ever holds the secret. |
| `GET /v1/w/:workspaceId/webhooks` | none | `{ "schemaVersion": 1, "subscriptions": [...] }` |
| `DELETE /v1/w/:workspaceId/webhooks/:subscriptionId` | none | `{ "isDeleted": true }` |
| `GET .../webhooks/:subscriptionId/deliveries` | `limit` 1 to 200, default 50 | `{ "schemaVersion": 1, "attempts": [...], "deadLetters": [...] }` |
| `POST .../webhooks/:subscriptionId/replay` | `{ "eventIds": [...] }` (optional, 1 to 500 uuids; omit for every dead letter) | `{ "requeued": 3 }` |
| `POST .../webhooks/:subscriptionId/ack` | `{ "throughCursor": "42" }` | `{ "ackedCursor": "42", "deliveredCursor": "40" }` |

Create response:

```json
{
  "schemaVersion": 1,
  "subscriptionId": "uuid",
  "url": "https://example.com/clean",
  "kinds": null,
  "deliveredCursor": "42",
  "createdAt": "iso",
  "secret": "whsec_...",
  "signing": {
    "algorithm": "HMAC-SHA256",
    "signedContent": "{timestamp}.{raw body}",
    "headers": { "eventId": "clean-webhook-id", "signature": "clean-webhook-signature", "timestamp": "clean-webhook-timestamp" },
    "toleranceSeconds": 300,
    "dedupeOn": "eventId"
  }
}
```

- **The secret is shown once.** Clean keeps only a sealed copy and never returns it again. Lost it: delete the subscription and create a new one.
- A new subscription starts at the newest event when it is created (`deliveredCursor`). It does not push older events; read those with `get_notifications`.
- It receives every workspace audience event, plus member events addressed to the admin who created it, filtered by `kinds`.
- The URL must be https with a public host: no user or password in it, no `localhost`, no name without a dot, no `.local`, `.internal`, `.localhost` or `.home.arpa`, no private, loopback or link-local IP. Otherwise 400 `WEBHOOK_URL_UNSAFE`.
- At most 10 active subscriptions per workspace (409 `WEBHOOK_SUBSCRIPTION_LIMIT`).
- An unknown or deleted subscription id is 404 `WEBHOOK_SUBSCRIPTION_NOT_FOUND`.

Each list entry has `subscriptionId`, `url`, `kinds`, `deliveredCursor`, `ackedCursor`, `breakerState` (`closed`, `half_open`, `open`), `consecutiveFailures`, `nextAttemptAt`, `deadLetterCount`, `replayPendingCount`, `createdAt`. It never includes the secret.

### What Clean sends

One `POST` per event, `content-type: application/json`, `user-agent: Clean-Webhooks/1`, with these headers:

| Header | Value |
| --- | --- |
| `clean-webhook-id` | The event's `eventId`. Dedupe on it. |
| `clean-webhook-timestamp` | Unix seconds when this attempt was signed. |
| `clean-webhook-signature` | `v1=` then lowercase hex of HMAC-SHA256, keyed with the secret, over `{timestamp}.{raw body}`. |

Body:

```json
{
  "schemaVersion": 1,
  "subscriptionId": "uuid",
  "workspaceId": "uuid",
  "attempt": 1,
  "isReplay": false,
  "event": {
    "audience": "workspace",
    "cursor": "43",
    "eventId": "uuid",
    "kind": "channel_reply",
    "occurredAt": "iso",
    "payload": { "version": 1, "conversationId": "uuid", "replySource": "linkedin" }
  }
}
```

`event` is the same event `get_notifications` returns, with the same refs only payload.

### Delivery rules

- **At least once.** The same event can arrive more than once (a retry, a replay). Dedupe on `clean-webhook-id`.
- In cursor order per subscription. A failed event is retried before the ones behind it are sent.
- Any 2xx is delivered. Anything else, a timeout (10 seconds) or a network error is a failure. Redirects are not followed.
- Retries: up to 8 attempts per event, with a random wait up to 60 s, 4 min, 16 min and so on, capped at 6 hours, about 18 hours in all. A 429 or 503 with `Retry-After` waits that long instead (capped at 24 hours).
- After the 8th failure the event becomes a dead letter and delivery moves on to the next event. `POST .../replay` re-queues dead letters on the same schedule.
- Circuit breaker per subscription: 5 failures in a row open it for 5 minutes (or the `Retry-After`, if longer); then one test attempt; one success closes it.
- Request bodies are at most 16 KB. Clean reads at most 4 KB of your response and ignores it.
- `deliveries` shows each attempt's `outcome` (`delivered`, `retry`, `dead_lettered`), `statusCode`, `latencyMs`, `errorCode` and `nextRetryAt`. Error codes: `WEBHOOK_HTTP_STATUS` (non 2xx), `WEBHOOK_TIMEOUT`, `WEBHOOK_DNS_FAILED`, `WEBHOOK_NETWORK_FAILED`, `WEBHOOK_TARGET_REFUSED` (the host resolved to a non public address), `WEBHOOK_URL_UNSAFE`.

### ack_notifications

MCP input: `subscription_id` (uuid) and `through_cursor` (the cursor of the last event you processed, pushed or read with `get_notifications`). Calls `POST .../webhooks/:subscriptionId/ack`.

- Clean never pushes an event at or below the acked cursor, so an agent that polled ahead stops duplicate pushes.
- The ack never moves back and never moves past the newest event.
- Still dedupe on `eventId`: delivery is at least once.

### Receiver example

Verify against the **raw bytes** you received, before parsing JSON. The key is the whole secret string, `whsec_` included, as UTF-8. Reply 2xx quickly (within 10 seconds) and do slow work afterwards.

Python:

```python
import hashlib, hmac, os, re, time

SECRET = os.environ["CLEAN_WEBHOOK_SECRET"]  # the "whsec_..." value from create

def is_authentic(raw_body: bytes, headers) -> bool:
    timestamp = headers.get("clean-webhook-timestamp") or ""
    signature = headers.get("clean-webhook-signature") or ""
    if not re.fullmatch(r"0|[1-9][0-9]{0,15}", timestamp):
        return False
    if abs(int(time.time()) - int(timestamp)) > 300:
        return False
    if not re.fullmatch(r"v1=[0-9a-f]{64}", signature):
        return False
    digest = hmac.new(SECRET.encode("utf-8"), timestamp.encode("ascii") + b"." + raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest("v1=" + digest, signature)
```

Node:

```js
import { createHmac, timingSafeEqual } from 'node:crypto'

const secret = process.env.CLEAN_WEBHOOK_SECRET // the "whsec_..." value from create

export function isAuthentic(rawBody /* Buffer */, headers) {
  const timestamp = headers['clean-webhook-timestamp']
  const signature = headers['clean-webhook-signature']
  if (typeof timestamp !== 'string' || !/^(0|[1-9][0-9]{0,15})$/.test(timestamp)) return false
  if (Math.abs(Math.floor(Date.now() / 1000) - Number(timestamp)) > 300) return false
  if (typeof signature !== 'string' || !/^v1=[0-9a-f]{64}$/.test(signature)) return false
  const digest = createHmac('sha256', secret).update(`${timestamp}.`).update(rawBody).digest('hex')
  const expected = Buffer.from(`v1=${digest}`)
  const received = Buffer.from(signature)
  return expected.length === received.length && timingSafeEqual(expected, received)
}
```

Then:

```
if not is_authentic(raw_body, headers): return 401
event = json.loads(raw_body)["event"]
if event["eventId"] in seen: return 200
seen.add(event["eventId"]); queue(event); return 200
every so often: ack_notifications(subscription_id, through_cursor=last_cursor_handled)
```

### Not documented yet

#1073 will also pause a subscription whose creator is no longer an admin. That change is not in the PR yet, so it is not described here.
