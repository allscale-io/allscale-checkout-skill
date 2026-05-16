# Allscale Checkout Integration Guide for AI Coding Agents

A portable, agent-agnostic prompt that guides an AI coding agent through integrating [Allscale Checkout](https://www.allscale.io/products/checkout) into any app or website.

Works with **Claude Code**, **OpenAI Codex**, **Cursor**, **Aider**, **Continue**, **Cline**, **Hermes**, **OpenClaw**, and any other agent that can fetch a URL or read a local Markdown file.

---

## Before You Start

You need an **Allscale account** with **Commerce enabled**:

1. Go to [allscale.io](https://allscale.io) and create an account with passkey or email (recommended)
2. Go to _Avatar_ > _Commerce_ > _Enable now_ to enable checkout in your profile (need to verify your email if you haven't already)
3. Moving on to **Create new store** and follow the steps to generate your **API Key** and **API Secret** (remember to save both especially your API Secret as it'll only be shown once)

> If you need help getting set up, contact the Allscale BD team.

Once you have your API Key and API Secret, you're ready to load the guide into your agent.

---

## How It Works

The whole guide is a single Markdown file: [`integrate-allscale.md`](./integrate-allscale.md). Hand it to your AI coding agent and the agent walks you through the integration step by step — verifying credentials, writing code for your stack, testing connectivity, building the checkout flow, and debugging.

You don't install anything globally. Each agent has its own way of consuming a prompt; pick whichever section below matches your tool.

---

## Install

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

## What It Covers

The guide walks any agent through:

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
5. **Debugging** if anything goes wrong (signature mismatches, enum issues, etc.)

---

## Security

The guide instructs the agent to follow these rules to keep your credentials safe:

- API Secret is **never written into source code** — only into `.env` files
- `.env` is always added to `.gitignore` before credentials are stored
- All signing happens **server-side** — the secret never touches frontend/client code
- The agent will warn you if it detects credentials in a file that could be committed
- Endpoints calling the Allscale API get rate limiting and server-side amount validation

---

## Reference

- [Allscale API Documentation](https://docs.allscale.io/allscale-checkout/getting-started)
- [Buy Me a Bagel](https://github.com/allscale-io/buy_me_a_bagel) — working example built with this integration
- [AGENTS.md](https://agents.md/) — open standard this guide is designed to coexist with

## License

MIT
