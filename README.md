# Allscale Checkout Skill for Claude Code

A [Claude Code](https://claude.com/claude-code) skill that guides developers through integrating [Allscale Checkout](https://www.allscale.io/products/checkout) into any app or website.

---

## Before You Start

You need an **Allscale account** with **Commerce enabled**:

1. Go to [allscale.io](https://allscale.io) and create an account with passkey or email (recommended)
2. Go to _Avartar_ > _Commerce_ > _Enable now_ to enable checkout in your profile (need to verify your email if you haven't already)
4. Moving on to **Create new store** and follow the steps to generate your **API Key** and **API Secret** (remember to save both especially your API Secret as it'll only be shown once)

> If you need help getting set up, contact the Allscale BD team.

Once you have your API Key and API Secret, you're ready to install the skill and start integrating.

---

## Install

In Claude Code, paste this:

> Read and install allscale.io/skill

Claude will fetch the skill and save it to `.claude/commands/integrate-allscale.md`. That's it.

<details>
<summary>Prefer shell? Install manually with curl</summary>

```bash
mkdir -p .claude/commands
curl -o .claude/commands/integrate-allscale.md \
  https://raw.githubusercontent.com/allscale-io/allscale-checkout-skill/main/.claude/commands/integrate-allscale.md
```

</details>

---

## Usage

In Claude Code, run:

```
/integrate-allscale
```

The skill will:

1. **Check you have Allscale credentials** — if not, it tells you exactly how to get them
2. **Help you store credentials safely** — in a `.env` file that is gitignored, never in source code
3. **Ask about your tech stack** — then writes integration code for your specific framework (Next.js, Flask, Rails, Express, etc.)
4. **Walk you through step by step:**
   - HMAC-SHA256 request signing
   - Test route verification (confirm your setup works)
   - Checkout intent creation (the core payment flow)
   - Payment status polling
   - Webhook signature verification
5. **Help you debug** if anything goes wrong (signature mismatches, enum issues, etc.)

---

## Security

This skill follows these rules to keep your credentials safe:

- API Secret is **never written into source code** — only into `.env` files
- `.env` is always added to `.gitignore` before credentials are stored
- All signing happens **server-side** — the secret never touches frontend/client code
- The skill will warn you if it detects credentials in a file that could be committed

---

## What It Covers

- Full Allscale Checkout API authentication (HMAC-SHA256 signing)
- Creating checkout intents (`POST /v1/checkout_intents/`)
- Polling payment status (`GET /v1/checkout_intents/{id}/status`)
- Webhook callback verification
- Currency enum mappings
- Error code reference and debugging guide

---

## Reference

- [Allscale API Documentation](https://docs.allscale.io/allscale-checkout/getting-started)
- [Buy Me a Bagel](https://github.com/allscale-io/buy_me_a_bagel) — working example built with this integration

## License

MIT
