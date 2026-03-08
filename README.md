# Allscale Checkout Skill for Claude Code

A [Claude Code](https://claude.com/claude-code) skill that guides developers through integrating [Allscale Checkout](https://allscale.io) into any app or website.

## Install

Clone or add this repo as a git submodule in your project, then copy (or symlink) the skill into your project's `.claude/commands/` directory:

```bash
# Option 1: Copy the skill file directly
mkdir -p .claude/commands
curl -o .claude/commands/integrate-allscale.md \
  https://raw.githubusercontent.com/allscale-io/allscale-checkout-skill/main/.claude/commands/integrate-allscale.md
```

```bash
# Option 2: Clone the whole repo and symlink
git clone https://github.com/allscale-io/allscale-checkout-skill.git
mkdir -p .claude/commands
ln -s ../../allscale-checkout-skill/.claude/commands/integrate-allscale.md .claude/commands/integrate-allscale.md
```

## Usage

In Claude Code, run:

```
/integrate-allscale
```

The skill will:

1. Ask about your tech stack (Next.js, Flask, Rails, Express, etc.)
2. Walk you through each integration step with code written for your stack:
   - API credential setup
   - HMAC-SHA256 request signing
   - Test route verification
   - Checkout intent creation
   - Payment status polling
   - Webhook signature verification
3. Help you debug common issues (signature mismatches, enum gotchas, etc.)

## What it covers

- Full Allscale Checkout API authentication (HMAC-SHA256 signing)
- Creating checkout intents (`POST /v1/checkout_intents/`)
- Polling payment status (`GET /v1/checkout_intents/{id}/status`)
- Webhook callback verification
- Currency enum mappings
- Error code reference and debugging guide
- Security best practices

## Reference

- [Allscale API Documentation](https://github.com/allscale-io/AllScale_Third-Party_API_Doc)
- [Buy Me a Bagel](https://github.com/allscale-io/buy_me_a_bagel) — working example built with this integration

## License

MIT
