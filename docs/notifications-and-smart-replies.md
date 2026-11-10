# Notifications and smart replies (pending tools)

Status: **not live yet.** Every tool on this page depends on an open pull request in Clean's server repo (cleanmcp/Mono-icarus). Each section names the PR it depends on. A tool is usable only after its PR merges and deploys; until then the hosted server does not list it. The hosted server stays the source of truth for names and schemas.

| Tool | API route | Depends on |
| --- | --- | --- |
| `get_notifications` | `GET /v1/w/:workspaceId/notifications/events` | [Mono-icarus #1071](https://github.com/cleanmcp/Mono-icarus/pull/1071) |
| `smart_reply_list` | `GET /v1/w/:workspaceId/smart-replies` | [Mono-icarus #1069](https://github.com/cleanmcp/Mono-icarus/pull/1069) |
| `smart_reply_draft` | `POST /v1/w/:workspaceId/smart-replies/:conversationId/draft` | [Mono-icarus #1069](https://github.com/cleanmcp/Mono-icarus/pull/1069) |
| `smart_reply_approve_send` | `POST /v1/w/:workspaceId/smart-replies/:conversationId/send` | [Mono-icarus #1069](https://github.com/cleanmcp/Mono-icarus/pull/1069) |
| `smart_reply_auto_send` | `POST /v1/w/:workspaceId/smart-replies/:conversationId/auto-send` | [Mono-icarus #1075](https://github.com/cleanmcp/Mono-icarus/pull/1075) (built on #1069) |
| `proposals_read` | `GET /v1/w/:workspaceId/agent/conversations/:conversationId/proposals` | [Mono-icarus #1074](https://github.com/cleanmcp/Mono-icarus/pull/1074) |
| `proposal_decide` | `POST /v1/w/:workspaceId/agent/runs/:runId/steps/:stepIndex/proposal/{confirm,decline}` | [Mono-icarus #1074](https://github.com/cleanmcp/Mono-icarus/pull/1074) |
| `channel_posts_read` | `GET /v1/w/:workspaceId/channel-posts/{digest,campaign-report,account-alerts,signal-questions}` | [Mono-icarus #1072](https://github.com/cleanmcp/Mono-icarus/pull/1072) |
| Webhook subscriptions | `POST`, `GET /v1/w/:workspaceId/webhooks`, `DELETE .../webhooks/:subscriptionId`, `GET .../deliveries`, `POST .../replay` | [Mono-icarus #1073](https://github.com/cleanmcp/Mono-icarus/pull/1073) (built on #1071) |
| `ack_notifications` | `POST /v1/w/:workspaceId/webhooks/:subscriptionId/ack` | [Mono-icarus #1073](https://github.com/cleanmcp/Mono-icarus/pull/1073) |

Every MCP tool here calls the API route beside it with the caller's own credential, so the API and MCP behave the same: same checks, same refusals, same audit. Nothing new is needed to connect: the existing Clean sign in (OAuth) or workspace API key reaches these routes. A bot acting for many members can instead use one shared agent key (#1076, below).

## How refusals look

Over the API a refusal is an HTTP error with this body:

```json
{ "error": { "class": "conflict", "domain": "outreach", "reason": "SMART_REPLY_STALE" } }
```

`class` sets the HTTP status: `invalid` 400, `forbidden` 403, `not_found` 404, `conflict` 409, `rate_limited` 429, `internal` 500. A request that fails its input schema is 400 with reason `REQUEST_INVALID`.

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
- Without `kinds`: a call returns up to `limit` events. `nextCursor` is the last event returned. `hasMore` is true when more events are already there.
- With `kinds`: a call scans up to 1,000 events and returns the matching ones, up to `limit`. `nextCursor` moves past every event it scanned, matching or not, so events that did not match are never scanned again. If more than `limit` matched, `nextCursor` is the last event returned instead.
- So **`hasMore` can be true while `events` is empty** (a filtered call scanned 1,000 events and none matched). Keep calling while `hasMore` is true; do not stop on an empty page.
- When nothing new has committed, `nextCursor` comes back unchanged, `events` is empty and `hasMore` is false.
- Always save `nextCursor` after a page, not the last event's `cursor`: with `kinds`, `nextCursor` can be ahead of the last returned event. Each event's own `cursor` is only for stopping partway through a page; resuming from it is safe but rescans.

### Dedupe

Dedupe on `eventId`. The server never writes a second event for the same underlying Slack delivery (a replayed Slack event writes nothing). If your agent crashes after acting on an event but before saving the cursor, it reads that event again on restart; a set of seen `eventId` values stops a second ping.

### Kinds

`agent_reply`, `link_prompt`, `notice`, `channel_reply`, `connect_channel`, `digest`, `proposal_card`, `proposal_result`, `account_alert`, `campaign_report`, `signal_questions`, `smart_reply_draft`, `reminder`.

The first eleven are the Slack delivery kinds and are written for every Slack delivery. #1071 itself writes neither `smart_reply_draft` nor `reminder`. #1069 writes `smart_reply_draft` (member audience, for the drafter) with `outcome` `drafted`, `approved` or `send_failed`, and #1075 adds `autosent` and `autosend_refused`. No PR writes `reminder` yet.

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
every few minutes:
  repeat:
    page = get_notifications(since_cursor=cursor, kinds=[...], limit=100)   # kinds optional
    for event in page.events:
      if event.eventId not in seen:
        handle(event)        # for a reply alert, read details with inbox_read
        seen.add(event.eventId)
    cursor = page.nextCursor
    save(cursor, seen)
  until not page.hasMore     # an empty page with hasMore true still means call again
```

## Smart replies

Depends on [Mono-icarus #1069](https://github.com/cleanmcp/Mono-icarus/pull/1069).

Smart replies answer a lead's newest message in a shared LinkedIn thread. Clean writes the draft; a person reads the exact text and approves it; only then does it send, through the same send the Team reply box and the chat Send card use. The one exception is `smart_reply_auto_send` (#1075, below), which is off for everyone until staff turn it on for a single LinkedIn seat.

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

A send that LinkedIn settles as failed (not one that may have reached LinkedIn) moves the draft to state `send_failed`, with an audit row and a `smart_reply_draft` event (`outcome: "send_failed"`). That frees the lead message: a member may draft again and approve the new draft. Draft states are `ready`, `superseded`, `approved` and `send_failed`.

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
| `SMART_REPLY_BODY_NOT_SANITIZED` | 409 | The stored text would still be changed by the outbound dash rule (the rule grew after drafting), or no longer matches its hash. Clean refuses rather than rewrite text on the way out | Draft again |
| `SMART_REPLY_REFUSED` | 500 | The draft failed Clean's sender identity check, the workspace's member names could not be read, or the thread has no seat or its seat belongs to someone other than the thread's owner (retryable) | Draft again later, or reply in Clean |
| `SMART_REPLY_GENERATION_FAILED` | 500 | The model call that writes the draft failed (retryable) | Draft again |
| `STAFF_CANNOT_CONFIRM_SENDS` | 403 | Staff signed in as the client | The client member sends |
| `REQUEST_INVALID` | 400 | Input failed the schema | Fix the input |

### smart_reply_auto_send

Depends on [Mono-icarus #1075](https://github.com/cleanmcp/Mono-icarus/pull/1075), which is built on #1069.

Drafts the reply to the lead's newest message in one LinkedIn thread and sends it with **no one approving the text**. Input: `conversationId` (uuid). API: `POST /v1/w/:workspaceId/smart-replies/:conversationId/auto-send` with body `{}`. Role: member; works with an API key or OAuth token.

**Off by default.** It sends only when both of these are on:

1. Clean's global kill switch, the API setting `SMART_REPLY_AUTOSEND_ENABLED=true`. While it is not `true`, every call is refused with `SMART_REPLY_AUTOSEND_OFF`.
2. A staff grant for that one LinkedIn seat: capability `smart_reply_autosend` with the seat's `connectionId`, given by Clean staff only after Pavan approves that seat. It is never a customer setting. Revoking the grant turns it off.

When it may send, it still sends only:

- If the caller owns the thread's seat.
- Between 08:00 and 20:00 in the seat's own time zone. That is the seat's sending zone, or else the workspace's Slack digest zone.
- At most 10 automatic sends per seat in any 24 hours.
- If every detail the draft cites (a time, date, weekday, meeting length, link, email address, amount of money or capitalised name) already appears in the thread, the offer, the lead's or sender's name, or the seat's saved context.

Response when it sent (HTTP 202):

```json
{ "decision": "sent", "draftId": "uuid", "requestId": "uuid", "state": "sent | failed | queued" }
```

Response when it refused (HTTP 200; nothing was sent):

```json
{ "decision": "refused", "draftId": "uuid or null", "reason": "SMART_REPLY_AUTOSEND_QUIET_HOURS" }
```

`draftId` is `null` when it refused before writing a draft. A refused draft is not sent, and this response holds no approval token. To send by hand, run `smart_reply_draft` and `smart_reply_approve_send` as usual. Each decision, sent or refused, writes one audit row and one `smart_reply_draft` event for the caller (`outcome` `autosent` or `autosend_refused`). Neither holds the text.

Refusal reasons (always HTTP 200, `decision: "refused"`):

| Reason | When |
| --- | --- |
| `SMART_REPLY_AUTOSEND_NOT_SEAT_OWNER` | The caller does not own this thread's LinkedIn seat |
| `SMART_REPLY_AUTOSEND_OFF` | The global kill switch is off |
| `SMART_REPLY_AUTOSEND_SEAT_NOT_ALLOWED` | Staff have not allowed this seat |
| `SMART_REPLY_AUTOSEND_NO_TIME_ZONE` | Neither the seat nor the workspace has a known time zone |
| `SMART_REPLY_AUTOSEND_QUIET_HOURS` | Outside 08:00 to 20:00 local time |
| `SMART_REPLY_AUTOSEND_RATE_CAPPED` | The seat already sent 10 automatic replies in the last 24 hours |
| `SMART_REPLY_PREVIOUS_SEND_FAILED` | An earlier send to this lead message failed. Automatic sends never retry it; a member may approve one by hand |
| `SMART_REPLY_UNGROUNDED_DETAIL` | The draft cites a detail that none of its sources hold |
| `LINKEDIN_REPLY_NOT_ALLOWED` | The caller cannot reply from this seat |
| `SMART_REPLY_LEAD_OPTED_OUT` | The lead said no or asked to be left alone |
| `SMART_REPLY_NOTHING_TO_ANSWER` | The newest message is not from the lead |
| `SMART_REPLY_THREAD_CHANGED` | A newer message or a reply arrived, including one that arrived while the draft was being written |
| `SMART_REPLY_RATE_LIMITED` | More than 20 drafts a minute by this caller |
| `SMART_REPLY_DRAFT_CHANGED` | The draft was replaced before it could be approved |
| `SMART_REPLY_STALE` | The draft expired before it could be approved |
| `SMART_REPLY_BODY_NOT_SANITIZED` | The stored text failed the outbound dash rule or its hash |
| `SMART_REPLY_ALREADY_SENT` | This lead message already got a smart reply |

These are still HTTP errors, not refusals: `STAFF_CANNOT_CONFIRM_SENDS` (403, Clean staff signed in as the client), a thread that does not exist or that you cannot see (404), `SMART_REPLY_THREAD_TOO_LONG`, drafting failures (`SMART_REPLY_DRAFT_EMPTY`, `SMART_REPLY_REFUSED`, `SMART_REPLY_GENERATION_FAILED`) and `REQUEST_INVALID`.

## Chat cards: proposals_read and proposal_decide

Depends on [Mono-icarus #1074](https://github.com/cleanmcp/Mono-icarus/pull/1074).

A chat card is an action the Clean agent proposed in a conversation and that waits for a person: add leads to a campaign, launch, pause, add to a CRM. These two tools list and decide cards through the same routes the app and the Slack buttons use, so the same rules and audit apply. Only the person who asked for the card can decide it.

### proposals_read

Input: `conversationId` (uuid, from research or reply). API: `GET /v1/w/:workspaceId/agent/conversations/:conversationId/proposals`. Role: viewer; works with an API key or OAuth token.

Returns the conversation's cards from the last day, newest first:

```json
{ "schemaVersion": 1, "proposals": [ { "...": "one card view, below" } ] }
```

Each card view: `schemaVersion` (1), `action`, `runId`, `stepIndex`, `state` (`pending`, `confirmed`, `declined`, `expired` or `unavailable`), `canDecide` (true only for the requester while the card is pending), `card` (what the card shows, including its warnings), `proposedAt`, `expiresAt`, `requesterName` (or null) and `result` (`{ facts, isSettled }`, or null before a decision runs). A deleted, closed or other workspace's conversation returns `CONVERSATION_NOT_FOUND`.

One card on its own: `GET /v1/w/:workspaceId/agent/runs/:runId/steps/:stepIndex/proposal` returns `{ "proposal": <card view> }` (viewer).

### proposal_decide

Input:

| Field | Type | Notes |
| --- | --- | --- |
| `runId` | uuid | From `proposals_read` |
| `stepIndex` | integer | 1 to 100000, from `proposals_read` |
| `decision` | `"confirm"` or `"decline"` | |
| `acknowledged` | string array | Up to 4 codes of 1 to 40 characters, default `[]`. The code of each warning that needs acknowledging, sent only after the person agreed to it. Used on confirm only |

API (role member; works with an API key or OAuth token):

- Confirm: `POST /v1/w/:workspaceId/agent/runs/:runId/steps/:stepIndex/proposal/confirm` with body `{ "acknowledged": [...] }`.
- Decline: `POST /v1/w/:workspaceId/agent/runs/:runId/steps/:stepIndex/proposal/decline` with no body.

Both return `{ "proposal": <card view>, "replayed": false }`. A second confirm or decline of the same card, or two at once, returns the first result with `replayed: true` and does nothing again. Confirm runs exactly the arguments stored on the card; the call sends only acknowledgements, so it cannot change what the card does. If the action itself refuses, nothing is written and the card stays pending until it expires.

**Sign-in rule for sends (`SIGN_IN_AGAIN`).** Confirming a card that sends outreach needs the member's own recent sign-in to Clean: their latest live session, with no staff member acting in it, must be no older than the Clean setting `chat.recent_sign_in_hours` (12 hours as first set). With an API key or OAuth token, Clean reads that latest sign-in, exactly as it does for a Slack button. Otherwise it returns 403 `SIGN_IN_AGAIN`: ask the person to sign in to Clean, then confirm again. Staff acting as the member can never confirm a send (`STAFF_CANNOT_CONFIRM_SENDS`).

Errors:

| Code | HTTP | Meaning |
| --- | --- | --- |
| `INTEGRATION_PROPOSAL_NOT_FOUND` | 404 | No such card |
| `INTEGRATION_NOT_REQUESTER` | 403 | Only the person who asked for the card can decide it |
| `SIGN_IN_AGAIN` | 403 | A send card needs a recent sign-in, above |
| `STAFF_CANNOT_CONFIRM_SENDS` | 403 | Staff acting as the member cannot confirm a send |
| `INTEGRATION_ROLE_INSUFFICIENT` | 403 | The member's role is below what the action needs |
| `INTEGRATION_PROPOSAL_CONFIRMED` | 409 | Already confirmed |
| `INTEGRATION_PROPOSAL_DECLINED` | 409 | Already declined |
| `INTEGRATION_PROPOSAL_EXPIRED` | 409 | Expired |
| `INTEGRATION_ACTION_UNAVAILABLE` | 409 | The action can no longer run |
| `INTEGRATION_ACKNOWLEDGEMENT_REQUIRED` | 409 | A warning that needs acknowledging is missing from `acknowledged` |
| `INTEGRATION_DECISION_RATE_LIMITED` | 429 | More than 20 decisions a minute by this member (retryable) |
| `INTEGRATION_PROPOSAL_CARD_INVALID` | 500 | The stored card could not be read |

Decide a card only when the person asked you to.

## Shared agent key and X-Clean-Acting-Member

Depends on [Mono-icarus #1076](https://github.com/cleanmcp/Mono-icarus/pull/1076).

One bot (the Grok bot) can act for many members of a workspace with one API key, the way the Slack bot does. Without it, each member needs their own key.

### Making the key

An owner or admin makes it from a signed-in browser session, never with a key or token:

- `POST /v1/w/:workspaceId/api-keys` with `{ "name": "...", "scope": "full" | "read", "actingMemberUserIds": ["userId", ...], "expiresInDays": n }` (`expiresInDays` optional). `actingMemberUserIds` holds 1 to 50 user ids, each a member of this workspace; duplicates are dropped. Returns 201 `{ apiKey, key, workspaceId }`. The full `key` is shown only this once.
- `PATCH /v1/w/:workspaceId/api-keys/:keyId` with `{ "actingMemberUserIds": [...] }` replaces the list (role admin). Returns `{ actingMemberUserIds, keyId }`.

Each create and each list change writes an audit row (`api_key.created`, `api_key.members_changed`) holding the list. If that audit row cannot be written, the change is undone.

### Calling with it

Send the key as usual and name the member in the `X-Clean-Acting-Member` header, by user id:

```http
GET /v1/w/:workspaceId/notifications/events
Authorization: Bearer <shared agent key>
X-Clean-Acting-Member: <member user id>
```

The request then runs as that member. Their rights are the lowest of three:

1. the member's own workspace role, checked on every request as usual;
2. the key's scope (`read` keys can only make GET calls);
3. the current role of the member who made the key.

Every audit row the request writes names the acting member as actor and carries the key's id (`agentKeyId`). A send still needs that member's own recent sign-in to Clean (see `SIGN_IN_AGAIN` above).

### Refusals

| Code | HTTP | When |
| --- | --- | --- |
| `ACTING_MEMBER_REQUIRED` | 403 | A shared agent key was sent without `X-Clean-Acting-Member` (or with it empty) |
| `ACTING_MEMBER_NOT_ALLOWED` | 403 | The named member is not on the key's list |
| `ACTING_MEMBER_FORBIDDEN` | 403 | The header was sent with a member's own key or an OAuth token. They always act as their owner, so the header is refused, not ignored |
| `WORKSPACE_NOT_FOUND` | 404 | The acting member, or the key's maker, is no longer in the workspace. A maker who left stops the key at once |
| `ROLE_INSUFFICIENT` | 403 | The lowest of the three rights is below what the route needs. Also returned when someone other than an owner or admin tries to make or change a shared agent key |
| `TOKEN_SCOPE_FORBIDDEN` | 403 | A `read` key on a call that is not a GET |
| `ACTING_MEMBER_NOT_IN_WORKSPACE` | 400 | Making or changing the key: a listed user is not a member of this workspace |
| `API_KEY_NOT_FOUND` | 404 | Changing the list of a key that does not exist, is in another workspace, or is not a shared agent key |

## Per-route rate limits

Depends on [Mono-icarus #1077](https://github.com/cleanmcp/Mono-icarus/pull/1077).

Every route an API key, OAuth token or MCP tool reaches is limited, per route, to:

- **120 calls a minute per credential.** An API key counts as one credential, so a shared agent key shares its 120 across every member it acts for. An OAuth token counts per connector, member and workspace.
- **600 calls a minute per workspace**, across all its keys and tokens.

A route is its method plus its path pattern, so `GET /v1/w/:workspaceId/notifications/events` is one route whatever the ids, and each MCP tool call counts at each route it reaches. The window is a rolling 60 seconds, counted in each API process. A refused call is not counted. Browser sessions are not counted. The key's own limit of 240 calls a minute across all routes still applies.

The call over the limit gets HTTP 429 with reason `TOKEN_ROUTE_RATE_LIMITED` (`class: "rate_limited"`, `retryable: true`), at that route only: other routes and other credentials keep working. Over MCP the tool returns an error result with the same reason. The response has no `Retry-After` header, so back off yourself:

```python
import random, time

def call_with_backoff(call, max_tries=6):
    delay = 1.0
    for attempt in range(max_tries):
        response = call()
        if response.status_code != 429:
            return response
        reason = response.json().get("error", {}).get("reason")
        if reason != "TOKEN_ROUTE_RATE_LIMITED":
            return response              # some other limit; handle it on its own terms
        time.sleep(delay + random.uniform(0, delay / 2))
        delay = min(delay * 2, 60.0)     # the window is 60 seconds, so never wait longer
    return response
```

For polling, one `get_notifications` call every few minutes per workspace stays far under the limit. When `hasMore` is true, page on at once, and back off only after a 429.

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
- At most 10 subscriptions per workspace, active or paused (409 `WEBHOOK_SUBSCRIPTION_LIMIT`).
- An unknown or deleted subscription id is 404 `WEBHOOK_SUBSCRIPTION_NOT_FOUND`.

Each list entry has `subscriptionId`, `url`, `kinds`, `deliveredCursor`, `ackedCursor`, `breakerState` (`closed`, `half_open`, `open`), `consecutiveFailures`, `nextAttemptAt`, `deadLetterCount`, `replayPendingCount`, `createdAt`, `state` (`active` or `paused`) and `pausedReason` (null unless paused). It never includes the secret.

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

### Paused subscriptions

Clean delivers only while the admin who created the subscription is still an owner or admin of the workspace. When that is no longer true, the next delivery pauses the subscription instead: `state` becomes `paused`, `pausedReason` is `creator_not_admin`, nothing more is sent, and an audit row (`webhook.subscription_paused`) is written. A paused subscription still shows in the list, still counts toward the 10, and can still be acked or deleted. #1073 has no route that resumes it: an admin deletes it and creates a new one.
