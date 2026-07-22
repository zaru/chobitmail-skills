# chobitmail Agent Skills

[Agent Skills](https://agentskills.io/) for the [chobitmail](https://chobitmail.com) disposable email API — one-time inboxes, long-poll wait, pre-extracted OTP codes and verification links for E2E tests and browser agents.

This repository is the **public install mirror**. Skill content is authored in the private chobitmail monorepo (`skills/`) and published here so anyone can install without access to the product source.

## Install

```bash
npx skills add chobitapp/chobitmail-skills
# or pin the skill name
npx skills add chobitapp/chobitmail-skills --skill chobitmail
# global (all projects)
npx skills add chobitapp/chobitmail-skills --skill chobitmail -g -y
```

After install, ask your coding agent to write Playwright/Vitest flows that need signup OTP, magic links, or password-reset email.

## Prerequisites

1. Create an API key at [chobitmail.com](https://chobitmail.com) (GitHub login)
2. Set `CHOBITMAIL_API_KEY` in your environment or CI secrets

```bash
export CHOBITMAIL_API_KEY=cbm_live_...
```

## Skills

| Skill | Path | Purpose |
|-------|------|---------|
| `chobitmail` | [skills/chobitmail/SKILL.md](./skills/chobitmail/SKILL.md) | Create disposable inboxes, wait with 408 reconnect, use `codes` / `links` |

## License

ISC — see [LICENSE](./LICENSE).
