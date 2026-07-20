---
name: chobitmail
description: >
  Integrate chobitmail disposable email API for E2E tests and AI browser agents.
  Create one-time inboxes, long-poll until mail arrives, and use pre-extracted OTP
  codes and verification links without parsing HTML. Use when writing Playwright,
  Cypress, Vitest, or agent flows that need email verification, signup OTP,
  password-reset links, magic links, or disposable addresses. Triggers: chobitmail,
  one-time inbox, wait for email, OTP from email, verification link, disposable
  email API, temp mail for tests.
license: ISC
metadata:
  author: zaru
  homepage: https://chobitmail.com
  docs: https://chobitmail.com
  source: https://github.com/zaru/chobitmail-skills
  install: npx skills add zaru/chobitmail-skills
---

# chobitmail

Disposable email API for automated tests and agents. Base URL: `https://chobitmail.com`.

**Install this skill:** `npx skills add zaru/chobitmail-skills`  
(Public mirror of monorepo `skills/chobitmail`; product source stays private.)

**Auth:** every request needs `Authorization: Bearer <API_KEY>`.
Get a key at the [dashboard](https://chobitmail.com) (GitHub login). Store it in `CHOBITMAIL_API_KEY` — never hardcode.

**Canonical docs:** this skill's [references/api.md](references/api.md). Prefer live HTTP responses over the skill if anything conflicts.

## When to use

| Goal | Approach |
|------|----------|
| Signup / login OTP | Create inbox → submit `address` → wait → `message.codes[0]` |
| Click verification / magic link | Same → `message.links[0]` (or pick the matching URL) |
| Assert email content | Wait with `subject` / `from` filters, then read `text` / `html` |
| Parallel test isolation | One inbox per test (addresses never mix) |

Do **not** use chobitmail as a personal mailbox or for production user mail.

## Minimal workflow

```text
1. POST /api/inboxes          → { id, address, expiresAt }
2. Drive app with address     → signup form, "forgot password", etc.
3. GET  .../messages/wait     → reconnect on 408 until mail or deadline
4. Use message.codes / links  → no HTML parsing required
5. Optional DELETE /api/inboxes/:id or DELETE /api/inboxes  (TTL auto-deletes otherwise)
```

```bash
export KEY="$CHOBITMAIL_API_KEY"
export BASE=https://chobitmail.com

curl -s -X POST "$BASE/api/inboxes" \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{"ttl":600}'
# → {"id":"...","address":"...@chobitmail.com",...}

curl -s "$BASE/api/inboxes/<id>/messages/wait?timeout=25&subject=verify" \
  -H "Authorization: Bearer $KEY"
# → 200 {"message":{...,"codes":["482913"],"links":["https://..."]}}
# → 408 {"error":"timeout"}  ← reconnect, not a hard failure
```

## Critical rules for agents

1. **Always implement wait reconnect.** Each wait call lasts at most 30s (`timeout` 1–30, default 25). On **408**, call wait again until your test deadline. Do not treat 408 as a failed test.
2. **Prefer wait over list.** `GET .../messages` returns immediately (may be empty). Tests should use `.../messages/wait`.
3. **Use server-side extraction.** Prefer `codes` (4–8 digit OTPs) and `links` over scraping `html`/`text`.
4. **TTL is short.** Requested `ttl` is clamped to **60–600 seconds** (default 600). Long E2E runs must finish within TTL or recreate the inbox.
5. **Free-tier quotas are tight.** Concurrent active inboxes: **1** (or **2** with a verified sender domain). Creates/day: **5** (50 when verified). Messages received/day: **5** (50 when verified, UTC). For parallel suites, serialize mail-dependent tests or delete inboxes early.
6. **404 is intentional ambiguity.** Missing, expired, and other-tenant IDs all return 404. Most test 404s mean TTL expired.
7. **Same-team keys share inboxes.** Keys and inboxes are team-scoped; rotation is safe once CI uses the new key.

## Node helper (copy into tests)

```js
const BASE = process.env.CHOBITMAIL_BASE_URL ?? "https://chobitmail.com";
const AUTH = { Authorization: `Bearer ${process.env.CHOBITMAIL_API_KEY}` };

export async function createInbox(ttl = 600) {
  const res = await fetch(`${BASE}/api/inboxes`, {
    method: "POST",
    headers: { ...AUTH, "Content-Type": "application/json" },
    body: JSON.stringify({ ttl }),
  });
  if (res.status === 429) {
    const body = await res.json().catch(() => ({}));
    throw new Error(`chobitmail quota: ${body.reason ?? "exceeded"}`);
  }
  if (!res.ok) throw new Error(`createInbox ${res.status}: ${await res.text()}`);
  return res.json(); // { id, address, createdAt, expiresAt }
}

/** Long-poll with 408 reconnect until deadline. */
export async function waitForMessage(inboxId, params = {}, maxWaitMs = 120_000) {
  const query = new URLSearchParams({ timeout: "25", ...params });
  const deadline = Date.now() + maxWaitMs;
  while (Date.now() < deadline) {
    const res = await fetch(
      `${BASE}/api/inboxes/${inboxId}/messages/wait?${query}`,
      { headers: AUTH },
    );
    if (res.status === 200) return (await res.json()).message;
    if (res.status === 408) continue;
    if (res.status === 429) throw new Error("tooManyWaiters — retry shortly");
    throw new Error(`waitForMessage ${res.status}: ${await res.text()}`);
  }
  throw new Error("email did not arrive before deadline");
}

export async function listInboxes() {
  const res = await fetch(`${BASE}/api/inboxes`, { headers: AUTH });
  if (!res.ok) throw new Error(`listInboxes ${res.status}: ${await res.text()}`);
  return (await res.json()).inboxes;
}

export async function deleteInbox(inboxId) {
  await fetch(`${BASE}/api/inboxes/${inboxId}`, {
    method: "DELETE",
    headers: AUTH,
  });
}

export async function deleteAllInboxes() {
  await fetch(`${BASE}/api/inboxes`, {
    method: "DELETE",
    headers: AUTH,
  });
}
```

### Playwright example

```js
test("signup OTP", async ({ page }) => {
  const inbox = await createInbox();
  await page.goto("/signup");
  await page.fill('[name="email"]', inbox.address);
  await page.click('button[type="submit"]');

  const message = await waitForMessage(inbox.id, { subject: "code" });
  await page.fill('[name="otp"]', message.codes[0]);
  await page.click('button[type="submit"]');
  await expect(page.getByText(/welcome|complete/i)).toBeVisible();
});
```

## Endpoints (cheat sheet)

| Method | Path | Notes |
|--------|------|--------|
| `GET` | `/api/inboxes` | Active inboxes for the team. **200** `{ inboxes: [...] }` |
| `POST` | `/api/inboxes` | Body optional `{ "ttl": 60–600 }`. **201** |
| `DELETE` | `/api/inboxes` | Destroy all active inboxes. **204** |
| `GET` | `/api/inboxes/:id/messages` | Immediate list (oldest first). **200** |
| `GET` | `/api/inboxes/:id/messages/wait` | Query: `timeout`, `from` (exact), `subject` (substring). Filters AND. |
| `DELETE` | `/api/inboxes/:id` | Immediate destroy. **204** |

### Message fields agents care about

| Field | Use |
|-------|-----|
| `codes` | OTP candidates (max 10). Usually `codes[0]`. |
| `links` | URLs from text + HTML href (max 20). Pick by host/path if multiple. |
| `from` / `subject` / `text` / `html` | Assertions and debugging |
| `attachments` | Metadata only (`filename`, `mimeType`, `size`) — **bodies not stored** |
| `receivedAt` | ISO 8601 |

## Errors

| Status | Body | Agent action |
|--------|------|----------------|
| 401 | `unauthorized` | Fix/missing API key |
| 403 | `forbidden` | Account banned — stop |
| 404 | `notFound` | Wrong id or TTL expired — recreate inbox |
| 408 | `timeout` | **Reconnect wait** |
| 429 | `quotaExceeded` + `reason` | `concurrent`: DELETE or wait TTL; `daily`: stop creating |
| 429 | `tooManyWaiters` | Back off; max 10 concurrent waits per inbox |

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Treating 408 as test failure | Loop until overall deadline |
| Parsing HTML for OTP | Use `message.codes` |
| `ttl: 3600` expecting 1 hour | Clamped to 600s max |
| Parallel mail tests on free tier | 1 concurrent inbox (2 if domain verified) — serialize or verify domain |
| Sharing one inbox across tests | Creates cross-talk; one inbox per test |
| Committing API keys | Env var / CI secret only |

## More detail

- Full request/response shapes, limits, and edge cases: [references/api.md](references/api.md)
- Framework patterns (Playwright, Vitest, agent loops): [references/patterns.md](references/patterns.md)
