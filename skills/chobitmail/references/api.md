# chobitmail API reference (agent detail)

Base URL: `https://chobitmail.com`  
Auth: `Authorization: Bearer <API_KEY>` on every call  
Format: JSON request/response  

API keys are issued in the dashboard (GitHub login). Plaintext is shown once at creation. Up to **2 keys per team** (disabled keys count). Keys and inboxes are **team-scoped**: any active key on the same team can create/read the same inboxes.

## POST /api/inboxes

Create a disposable address.

**Body (optional):**

```json
{ "ttl": 300 }
```

| Field | Type | Rules |
|-------|------|--------|
| `ttl` | number | Seconds. Clamped to **60–300**. Default **300**. |

**201:**

```json
{
  "id": "v56m2aq5ly17piq3",
  "address": "v56m2aq5ly17piq3@chobitmail.com",
  "createdAt": "2026-07-17T07:30:00.000Z",
  "expiresAt": "2026-07-17T07:35:00.000Z"
}
```

Use `address` in the app under test. After `expiresAt`, the inbox and messages are gone (404).

**429:**

```json
{ "error": "quotaExceeded", "reason": "concurrent" }
```
or `"reason": "daily"`.

Quota applies to **create** only. Listing/waiting existing inboxes still works.

## GET /api/inboxes/:id/messages

Return all messages oldest-first. Does not wait.

**200:**

```json
{ "messages": [ /* Message */ ] }
```

Empty array if nothing received yet.

## GET /api/inboxes/:id/messages/wait

Block until a matching message exists or timeout.

| Query | Meaning |
|-------|---------|
| `timeout` | Seconds to hold connection. **1–30**, default **25** |
| `from` | Exact match on envelope sender |
| `subject` | Substring match on subject |

Multiple filters are AND. Non-matching mail is stored but does not resolve the wait.

**200:** `{ "message": { /* Message */ } }` — first match  
**408:** `{ "error": "timeout" }` — client should reconnect  
**429:** `{ "error": "tooManyWaiters" }` — >10 concurrent waiters on this inbox  

Overall test wait = loop of wait calls until wall-clock deadline.

## DELETE /api/inboxes/:id

Destroy inbox + messages immediately. **204** empty body.  
Optional: TTL already cleans up; call early to free concurrent quota.

## Message object

```json
{
  "id": "k2x9m4p7q1w8e5r3",
  "from": "noreply@myapp.example",
  "subject": "Your code",
  "text": "Your code is 482913",
  "html": "<p>Your code is <b>482913</b></p>",
  "links": ["https://myapp.example/verify?token=abc123"],
  "codes": ["482913"],
  "attachments": [{ "filename": "a.pdf", "mimeType": "application/pdf", "size": 1024 }],
  "receivedAt": "2026-07-17T07:30:00.000Z"
}
```

| Field | Notes |
|-------|--------|
| `subject` | MIME-decoded (UTF-8 Japanese etc.) |
| `links` | From HTML `href` + plaintext URLs, max 20 |
| `codes` | 4–8 digit sequences that look like OTPs, max 10; digits inside URLs excluded |
| `attachments` | Metadata only — raw files not retained |
| `receivedAt` | ISO 8601 |

## Error catalog

| Status | Body | Meaning |
|--------|------|---------|
| 401 | `{"error":"unauthorized"}` | Missing/invalid/disabled/deleted key |
| 403 | `{"error":"forbidden"}` | Account suspended |
| 404 | `{"error":"notFound"}` | Unknown, expired, or other team's inbox (intentionally indistinct) |
| 408 | `{"error":"timeout"}` | Wait window elapsed — reconnect |
| 429 | `{"error":"quotaExceeded","reason":"concurrent"\|"daily"}` | Free-tier create limits |
| 429 | `{"error":"tooManyWaiters"}` | Concurrent wait cap on one inbox |

## Limits (free beta)

| Limit | Value |
|-------|--------|
| Concurrent active inboxes | 1 |
| Inbox creates / day (UTC) | 5 |
| Messages received / day (UTC) | 5 |
| API keys per team (incl. disabled) | 2 |
| Inbox TTL | 60s–5m (default 5m) |
| Messages stored per inbox | 50 (further mail dropped) |
| Max message size | 1 MB (larger discarded) |
| Max single wait | 30s |
| Concurrent waits per inbox | 10 |
| Attachment bodies | Not stored |

Mail to unknown/expired addresses, or past daily receive quota, is dropped silently (no bounce to sender).

## Beta notes

- Quotas may change.
- No sender-domain allowlist yet (any From is accepted).
- Service may stop or wipe data without notice.
- Intended for **tests and agents**, not real-user mailboxes.
