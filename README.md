# Allscale Guides for AI Coding Agents

Portable, agent-agnostic prompts that guide an AI coding agent through working with Allscale. Two guides live here, for two different jobs:

| Guide | Use it when you want to… | Load it from |
|---|---|---|
| **Checkout integration** | Take payments **inside your own app or website** — [Allscale Checkout](https://www.allscale.io/products/checkout) via the HMAC-signed API | [allscale.io/skill](https://allscale.io/skill) |
| **CLI setup** | Drive Allscale **from a terminal or an agent** — invoices, claim links, wallets, payouts — no app to build | [allscale.io/cliskill](https://allscale.io/cliskill) |

Not sure which? If you are writing code that charges your users, you want **Checkout integration**. If you want to run Allscale operations yourself or hand that ability to an agent, you want **CLI setup**.

Both work with **Claude Code**, **OpenAI Codex**, **Cursor**, **Aider**, **Continue**, **Cline**, **Hermes**, **OpenClaw**, and any other agent that can fetch a URL or read a local Markdown file.

---

## Checkout integration — Before You Start

You need an **Allscale account** with **Commerce enabled**:

1. Go to [allscale.io](https://allscale.io) and create an account with passkey or email (recommended)
2. Go to _Avatar_ > _Commerce_ > _Enable now_ to enable checkout in your profile (need to verify your email if you haven't already)
3. Moving on to **Create new store** and follow the steps to generate your **API Key** and **API Secret** (remember to save both especially your API Secret as it'll only be shown once)

> If you need help getting set up, contact the Allscale BD team.

Once you have your API Key and API Secret, you're ready to load the guide into your agent.

---

## How It Works

Each guide is a single Markdown file. Hand one to your AI coding agent and it walks you through the job step by step — for Checkout: verifying credentials, writing code for your stack, testing connectivity, building the flow, debugging; for the CLI: installing it, connecting it with the right permissions, and showing you how to take that access back.

Nothing is installed globally to use a guide. Each agent has its own way of consuming a prompt; pick whichever section below matches your tool.

---

## Checkout integration — Install

### Any AI coding agent (simplest, works everywhere)

Paste this into your agent's chat:

> Read and follow the instructions at allscale.io/skill

The agent fetches the file and starts the integration. This is the recommended path for Hermes, OpenClaw, Aider, Continue, Cline, or anything else that can read a URL.

### Claude Code (as a slash command)

Paste this in Claude Code:

> Read allscale.io/skill and save it to `.claude/commands/integrate-allscale.md`

Then invoke it with `/integrate-allscale`.

### OpenAI Codex

Download the guide into your repo so Codex picks it up as project context:

```bash
curl -L -o integrate-allscale.md https://allscale.io/skill
```

Then in your `AGENTS.md` (or in a Codex session), reference it:

```markdown
When the user wants to integrate Allscale Checkout, read and follow `./integrate-allscale.md`.
```

Or just paste the URL line from the "any agent" section above directly into Codex.

### Cursor

Save the guide as a Cursor rule:

```bash
mkdir -p .cursor/rules
curl -L -o .cursor/rules/integrate-allscale.mdc https://allscale.io/skill
```

Reference `@integrate-allscale` in chat to invoke.

### Manual / offline

```bash
curl -L -O https://allscale.io/skill
```

Open the file in your agent of choice and tell it to follow the instructions.

---

## CLI setup — Install

Same idea, different guide. Paste this into your agent's chat:

> Read and follow the instructions at https://allscale.io/cliskill

**Claude Code (as a slash command):**

> Read https://allscale.io/cliskill and save it to `.claude/commands/setup-allscale-cli.md`

Then invoke it with `/setup-allscale-cli`.

**Or fetch it directly:**

```bash
curl -L -o setup-allscale-cli.md https://allscale.io/cliskill
```

### What the CLI guide covers

1. **Prerequisites** — Node ≥ 20.10, with the install line for Windows, macOS and Linux
2. **Installing** `@allscale/cli`, globally or via `npx`
3. **Asking what you need it for first** — that decides which permissions get requested, and over-granting is the easiest mistake to make here
4. **Connecting** — browser-approved device pairing by default; `--no-browser` and email-OTP paths for servers, containers and CI
5. **Verifying** — including `allscale scope`, which shows the permissions the server *actually* granted (they can be narrower than what was asked for)
6. **The commands for your case** — invoices, claim links, wallets, transactions
7. **Using it from an agent or a script** — `--json`, `--select`, the self-describing schema (`operations` / `describe`), and the exit-code table. Exit 5 means log in again; exit 6 means a permission is missing and retrying will never help
8. **Revoking access** — `keys list` / `revoke` / `rotate`, the dashboard, and why `logout` alone is not enough if a machine goes missing

---

## What It Covers

The **Checkout integration** guide walks any agent through:

1. **Verifying you have Allscale credentials** — if not, it tells you exactly how to get them
2. **Storing credentials safely** — in a `.env` file that is gitignored, never in source code
3. **Asking about your tech stack** — then writing integration code for your specific framework (Next.js, Flask, Rails, Express, etc.)
4. **Step by step:**
   - HMAC-SHA256 request signing
   - Test route verification (confirm your setup works)
   - Checkout intent creation (the core payment flow)
   - Payment status polling
   - Webhook signature verification
   - Response signing (optional)
   - Claim Link Auto-Payout — sending money **out** (payouts / disbursements), including the required **Store Settings → Payout → enable API-Auto-Payout** step and the **14-day auto-refund** your ledger has to expect
   - Claim-link webhooks (`claimed` / `expired` / `cancelled`) — there is no status endpoint to poll
5. **Debugging** if anything goes wrong (signature mismatches, enum issues, permission/scope errors, etc.)

---

## Security

The **Checkout integration** guide instructs the agent to follow these rules to keep your credentials safe:

- API Secret is **never written into source code** — only into `.env` files
- `.env` is always added to `.gitignore` before credentials are stored
- All signing happens **server-side** — the secret never touches frontend/client code
- The agent will warn you if it detects credentials in a file that could be committed
- Endpoints calling the Allscale API get rate limiting and server-side amount validation

The **CLI setup** guide adds rules for the fact that a paired credential can move real money as you:

- `keys revoke --all` is an incident kill-switch that takes down every automation at once — the agent must never run it unless you ask for it
- Money-moving commands are shown to you and confirmed before they run
- `keys rotate` prints a new secret exactly once — the agent will not echo it back or store it for you
- Permissions requested are the narrowest that do the job, because the key that gets minted is what an attacker would inherit
- `logout` clears local credentials only; the guide is explicit that a lost machine needs `keys revoke` or the dashboard

---

## Reference

- [Allscale API Documentation](https://docs.allscale.io/allscale-checkout/getting-started)
- [Buy Me a Bagel](https://github.com/allscale-io/buy_me_a_bagel) — working example built with this integration
- [AGENTS.md](https://agents.md/) — open standard this guide is designed to coexist with

## License

MIT
