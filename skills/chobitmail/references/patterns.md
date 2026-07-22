# Integration patterns

Assumes the helpers from `SKILL.md` (`createInbox`, `waitForMessage`, `deleteInbox`) and env:

```bash
export CHOBITMAIL_API_KEY=cbm_live_...
# optional override for self-hosted / local:
# export CHOBITMAIL_BASE_URL=http://localhost:8787
```

## Playwright — preferred package

```bash
pnpm add -D @chobitmail/playwright
export CHOBITMAIL_API_KEY=cbm_live_...
```

```ts
import { test, expect } from "@chobitmail/playwright";

test("email OTP signup", async ({ page, inbox }) => {
  await page.goto("https://app.example/signup");
  await page.getByLabel("Email").fill(inbox.address);
  await page.getByRole("button", { name: /sign up|register/i }).click();

  const code = await inbox.waitForCode(
    { subject: "verification", timeout: 90_000 },
  );
  await page.getByLabel(/code|otp/i).fill(code);
  await page.getByRole("button", { name: /verify|continue/i }).click();
  await expect(page.getByText(/welcome|dashboard/i)).toBeVisible();
});
```

```ts
test("verify email link", async ({ page, inbox }) => {
  // ... trigger app to send verification email ...
  const link = await inbox.waitForLink({
    subject: "Verify",
    includes: "/verify",
  });
  await page.goto(link);
  await expect(page.getByText(/verified|success/i)).toBeVisible();
});
```

- Fixture auto-deletes the inbox after each test (frees free-tier concurrent slot).
- `waitForCode` / `waitForLink` are **fail-fast** on the first matching message (they do not skip to a later email).
- Free tier: concurrent **1**, daily create/message **5** — use `workers: 1`, avoid sharing one key across CI shards.
- Cypress is not supported.

## Playwright — copy-paste helpers (fallback)

If you cannot add the package yet, use the helpers from `SKILL.md` (`createInbox`, `waitForMessage`).

When multiple links appear (unsubscribe, tracking, CTA), filter by path/host — do not always take `[0]`.

## Vitest / Node (no browser)

Useful when the SUT is an API that sends mail:

```js
import { describe, it, expect } from "vitest";
import { createInbox, waitForMessage, deleteInbox } from "./chobitmail";

describe("password reset mail", () => {
  it("contains a reset link", async () => {
    const inbox = await createInbox();
    try {
      await fetch("https://api.example/password-reset", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ email: inbox.address }),
      });

      const message = await waitForMessage(inbox.id, {
        from: "noreply@example.com",
        subject: "reset",
      });
      expect(message.links.some((l) => l.includes("token="))).toBe(true);
    } finally {
      await deleteInbox(inbox.id);
    }
  });
});
```

## AI / browser agent loop

Agents should follow the same contract as tests:

1. Create inbox once per task.
2. Fill the app's email field with `address`.
3. Call wait with a **total** deadline (e.g. 2 minutes), reconnecting on 408.
4. Prefer `codes` / `links` over vision or HTML scraping.
5. On 429 `concurrent`, delete stale inboxes or wait for TTL before retrying create.
6. On 404 mid-flow, inbox expired — recreate and restart the mail step (user action may need re-trigger).

Pseudo-flow:

```text
inbox = createInbox()
fill_email(inbox.address); submit()
loop until deadline:
  res = wait(inbox.id, timeout=25, subject=hint)
  if res.status == 200: break
  if res.status == 408: continue
  else: fail
use res.message.codes[0] or open res.message.links[...]
```

## CI configuration

```yaml
# GitHub Actions example
env:
  CHOBITMAIL_API_KEY: ${{ secrets.CHOBITMAIL_API_KEY }}
```

- Do not print the key in logs.
- Free tier: **1 concurrent inbox** (2 with verified sender domain), **5 creates/day**, **5 messages/day** (higher when verified). Keep mail-dependent jobs serial and few unless domain-verified.
- Prefer deleting inboxes in `afterEach` / `finally` when running many sequential tests so concurrent quota frees immediately.
- Key rotation: create new key in dashboard → update secret → delete old key.

## Filtering tips

| Situation | Query |
|-----------|--------|
| Only care about verification mail | `subject` substring unique to that template |
| App sends multiple mails quickly | Narrow `subject` or `from` so wait ignores noise |
| Envelope From differs from header | `from` filter uses **envelope** sender |

Unmatched messages still appear in `GET .../messages` for debugging.

## Local development against this monorepo

```bash
pnpm dev   # localhost:8787
export CHOBITMAIL_BASE_URL=http://localhost:8787
export CHOBITMAIL_API_KEY=cbm_live_devkey0123456789  # after pnpm run seed:dev
node scripts/demo.mjs
```

Production domain for real mail delivery is `https://chobitmail.com` (not local).

## Failure diagnosis

| Symptom | Likely cause |
|---------|----------------|
| Create returns 429 concurrent | Another active inbox; DELETE it or wait TTL |
| Create returns 429 daily | Hit daily create cap (UTC day) |
| Wait always 408 then deadline | App never sent; wrong filter; spam/delay; receive quota exhausted |
| 404 after long test | TTL 10m max — shorten test or re-create earlier |
| Empty `codes` | Code not 4–8 digits, or only inside a URL |
| Empty `links` | Link only in image / unusual format |
| OTP wrong | Multiple codes — pick by length or message context, not always `[0]` if ambiguous |
