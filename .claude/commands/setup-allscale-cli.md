You are helping a user install the AllScale CLI and connect it to their AllScale account. Follow these steps in order. Be conversational — go step by step, confirm each step works before moving on. Run the commands yourself where you can, and show the user the output.

---

## Who this guide is for

Before you start, figure out which case this is:

- **Human at a workstation** — a developer or operator who wants `allscale` on their own machine. Steps 0–5 cover everything. Use the browser-approval login.
- **Agent / CI / container** — no browser available, commands run unattended. Same install, different login path (Step 3B), and read **Step 6** on how to consume CLI output programmatically.

Ask the user which one applies before proceeding.

> **What this CLI is.** `@allscale/cli` drives the same actions a user can take in [app.allscale.io](https://app.allscale.io) — invoices, claim links, wallets, transactions, payouts, stores. It is built for AI agents and humans alike. It is **not** the Checkout HMAC API; if the user wants to embed payments in their own app, they want `/integrate-allscale` instead, not this guide.

---

## CRITICAL SAFETY RULES

Every credential this CLI holds can move real money as the user's business. You MUST follow these rules at all times:

1. **The public CLI has no key-management commands at all** — no `keys list`, `keys revoke`, `keys rotate`, no kill-switch. Revocation lives in the Dashboard, on purpose: whoever hands out a key is the one who takes it back, and an agent key is refused by the server if it tries. **Never tell the user a CLI command will revoke something.** Point them at *Settings → Security → Sessions · Agents & Devices*.
2. **NEVER run a money-moving command without showing the user the exact command and getting explicit confirmation first.** That covers `wallet send`, `payout send`, `invoice pay`, `claim-link create`, `claim-link claim`.
3. **NEVER write any token or `~/.allscale/credentials.json` content into a repo file, a chat message, a log, or a commit.** If a secret is ever shown once, it is the user's to save — do not echo it back or keep a copy for them.
4. **NEVER pass `--insecure-storage` without telling the user what it does** (credentials land in a plaintext file at `~/.allscale/credentials.json` instead of the OS keychain) **and getting their agreement.**
5. **The published CLI talks to one place and cannot be redirected.** It is bound to `https://app.allscale.io`; `--api-base` does not exist and `ALLSCALE_API_BASE` is rejected. If a user asks how to point it at a test environment, the answer is that there is none — use a sandbox store instead (`allscale store create` makes one by default).
6. **Request the narrowest scopes that do the job.** Do not ask for `claim_link:all` or `store:all` when `invoice:read_only` is enough — see Step 3.
7. **`allscale logout` does NOT revoke anything server-side.** If a machine is lost or a key may be leaked, the fix is Dashboard revocation (*Settings → Security → Sessions · Agents & Devices*) — never `logout` alone, and there is no CLI command that does it.

---

## Step 0: Prerequisites

The CLI is a Node package. There is no standalone installer today, so Node is required.

Check what's there:

```bash
node -v
```

**Node must be ≥ 20.10.** If it's missing or older, install it for their platform:

**Windows (PowerShell):**
```powershell
winget install OpenJS.NodeJS.LTS
```

**macOS:**
```bash
brew install node
```

**Linux (Debian / Ubuntu)** — distro packages are usually too old, use NodeSource:
```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs
```

Re-run `node -v` and confirm ≥ 20.10 before continuing.

---

## Step 1: Install

```bash
npm install -g @allscale/cli
allscale --version
```

If they'd rather not install globally, `npx` works and always fetches the latest:

```bash
npx @allscale/cli --help
```

Global install is much faster for repeated use; recommend it for anything beyond a one-off.

**If `allscale: command not found` after installing**, npm's global bin directory is not on their PATH:

```bash
npm prefix -g        # the global prefix; its bin/ subdirectory must be on PATH
```

Add that `bin` directory to PATH, or fall back to `npx @allscale/cli`.

---

## Step 2: Ask what they need it for

Before logging in, ask:

1. **What do you want to do with it?** (read invoices, send invoices, issue claim links, run payouts, wire up an agent…)
2. **Is this your own machine, or a server / CI runner?**

Their answer decides the scopes you request in Step 3. Do not skip this — logging in with more permission than the task needs is the single easiest mistake to make here, and the key that gets minted is what an attacker would inherit.

---

## Step 3A: Log in (browser available)

This is the recommended path. It needs no password and works for passkey-only accounts.

```bash
allscale device-login --device-label <a-name-for-this-machine>
```

What happens: a browser tab opens, the user confirms with the web-app session they are **already** logged into, picks the permissions on the approval screen, and the CLI receives its credentials.

Request scopes up front so the approval screen is pre-filled with the minimum:

```bash
allscale device-login --device-label tom-mbp \
  --scopes invoice:read_only --scopes claim_link:all
```

Scope strings are `<category>:<tier>`:

| Category | Tiers | Notes |
|---|---|---|
| `invoice` | `read_only`, `all` | |
| `contact` | `read_only`, `all` | |
| `transaction` | `read_only` | Transaction history lives here, not under `wallet` |
| `wallet` | `read_only` | Wallet ids, names, networks, addresses, assets and balances. **No transactions, no transfers, no signing.** Grantable only where the narrow wallet endpoint is enabled — `allscale scope` is the authority |
| `claim_link` | `read_only`, `all` | `all` covers claim-link creation, which is a browser hand-off (Step 5) |
| `store` | `all` | Issues Store API credentials |

Things to tell the user plainly:

- **The final set is decided on the approval screen, and the server enforces it.** What they get can be *less* than what you requested — at mint time the chosen set is clipped to what their role may hold. Step 4 is how you find out.
- **`:all` implies `:read_only` for the same category only** — never across categories. `invoice:all` gives you invoice reads; it gives you nothing on `contact`.
- **`wallet` and `payroll` are not grantable.** Do not put them in `--scopes`. A request whose every scope clips away is refused outright rather than minting a dead key.
- **`cli_op:all` is appended automatically** to every pairing-minted key and never appears on the approval screen. It is CLI plumbing, not a permission the user granted — don't be alarmed by it in `allscale scope` output, and don't request it.
- Agent-key sessions are **default-deny**: an operation is reachable only if it is wired to a scope the key holds. Legacy cookie sessions are full-permission and their recorded scopes are advisory only — `allscale scope` tells you which kind you have.

**The approval screen also sets how long the key lives.** Four choices — **1 hour / 1 day / 30 days / 90 days**, defaulting to **30 days**. Picking 90 days shows a risk warning, and it should: the key is what an attacker inherits, and nothing renews it automatically.

Help the user pick by how long the work actually runs:

| Their situation | Suggest |
|---|---|
| Trying something out, one-off script | **1 hour** |
| A day's work, a debugging session | **1 day** |
| An automation they will keep running | **30 days** (default) |
| Long-lived unattended job | **90 days** — say out loud that it is the riskiest option |

When it lapses, automation stops with exit 5 and the fix is a fresh `device-login`. There is no renewal.

> **Every login path mints an agent key.** There is no cookie mode in the public CLI — no `--credential cookie`, no full-permission session. If you find a flow that only works with one, that is worth reporting, not working around.

## Step 3B: Log in (no browser — server, container, CI)

Print the approval link instead of opening a tab, then approve it from any device:

```bash
allscale device-login --no-browser
```

If cross-device approval isn't practical either, use an email OTP:

```bash
allscale otp-login --email <their-email>
```

For a fully scripted flow, `allscale otp-send` first and pass the pair through:

```bash
allscale otp-login --email <their-email> --otp-id <id> --otp <code>
```

Both OTP and the password path below mint the **same kind of scoped, expiring agent key** as browser pairing — they are different ways to prove who you are, not different kinds of credential. Where the flow cannot show an approval screen, the request still carries scopes and a validity period, defaulting to 30 days.

> **There is no password login in the public CLI.** `device-login` and `otp-login` are the only two ways in, and both mint the same kind of scoped, expiring credential.

---

## Step 4: Verify — and check what they actually got

```bash
allscale whoami
allscale scope
```

**`allscale scope` is the one that matters.** It reports the permissions the server actually granted, which can be narrower than what was requested. Run it now and show the user the result.

Confirm out loud that the granted scopes cover what they told you in Step 2. If they don't, re-run `device-login` and have them grant the missing ones on the approval screen — do not try to work around a missing scope in code.

Then prove a real call works, using a read-only command so nothing can go wrong:

```bash
allscale invoice list
```

Do not move on until this returns without an auth error.

---

## Step 5: Show them the commands they asked for

Only cover what's relevant to their Step 2 answer. Read-only first.

```bash
# Invoices
allscale invoice list                                  # first 50 rows
allscale invoice list --input '{"limit":50,"skip":50}'  # next page
allscale invoice sent
allscale invoice received
allscale invoice get <invoice-id>

# Wallets and transactions
allscale wallet list
allscale transaction list
```

Money-moving commands — **confirm with the user before running any of these** (Safety Rule 2):

```bash
# Send an invoice. Agent-key sessions must pass --wallet-id explicitly.
allscale invoice send --to-email client@example.com --amount 1.00 --wallet-id <wallet-id>

# Itemized: a three-field --line carries "description|quantity|amount".
# When EVERY line uses the three-field form the CLI sums them and --amount is optional.
allscale invoice send --to-email client@example.com --wallet-id <wallet-id> \
  --line "Discovery (4h)|4|25.00" --line "Implementation (10h)|10|25.00"

# Claim link. The CLI fixes the intent and the browser authorizes it: you pass
# the amount and chain here, they are locked, and the funding ceremony happens
# in the browser — the page cannot change what you specified.
# --idempotency-key is required; reuse it after an ambiguous result.
allscale claim-link create --idempotency-key order-1042 --amount 10 --chain base

# Read-only claim-link commands
allscale claim-link list
allscale claim-link get <claim-link-id>
allscale claim-link status --claim-token <token>
```

> Do not build the claim URL yourself, and do not treat this as a silent API call — funding needs the browser ceremony. For fully unattended payouts, that is the Store API (`/integrate-allscale`), not this CLI.

Useful to know: **claiming a link needs no login at all.** This is the shortest path for a recipient who has no AllScale account:

```bash
allscale claim-link claim --claim-token <token> --to 0x<recipient-address>
```

---

## Step 6: If the caller is an agent or a script

Two things make this CLI usable without hardcoding anything.

**Machine-readable output.** Most commands accept `--json`, which emits a structured envelope on stdout instead of human text. Prefer it in any script:

```bash
allscale invoice list --json
```

**Narrow the payload.** Read commands accept `--select` to override the default selection set, which keeps responses small and stable for parsing:

```bash
allscale invoice get <invoice-id> --select 'id status amount currency'
```

**Schema self-description — off by default.** The CLI can describe its own operations, but the full API map is deliberately hidden unless someone opts in:

```bash
export ALLSCALE_ALLOW_RAW=1        # required, or these exit 10 (raw.disabled)
allscale operations                # every operation the schema exposes
allscale describe <operation>      # full arg types, return type, example document
```

Without the variable they fail with `raw.disabled` and **exit 10** — that is the gate working, not a bug. Turning it on does not widen access: the boundary is still the key's scopes, enforced server-side. Mention the variable to the user rather than exporting it silently.

**Branch on exit codes, not on stderr text.** They are stable and deliberately distinct:

| Exit | Meaning | What to do |
|---|---|---|
| `0` | Success | — |
| `1` | Internal error | Report it; not user-fixable |
| `2` | Bad input, or credential storage unusable | Fix the arguments |
| `3` | Network error or timeout | Retry with backoff |
| `4` | No credentials | Run `device-login` |
| `5` | Credentials expired | Re-run `device-login` — agent keys do not auto-renew |
| `6` | Permission denied / capability gap | **Not fixable by retrying or re-logging in.** A scope is missing, or the action needs a one-time step in the web app |
| `7` | Not found | Check the id |
| `8` | Rate limited | Back off |
| `9` | Backend error | Retry, then report |
| `10` | Discovery / raw passthrough disabled | Set `ALLSCALE_ALLOW_RAW=1` only if the user wants `operations` / `describe` |
| `11` | Request signature rejected | **Upgrade the CLI** — this build's signing key is no longer accepted; re-login will not help |

Exit `6` versus exit `5` is the distinction worth wiring in: `5` means try logging in again, `6` means stop and tell a human.

---

## Step 7: Teach them how to take it back

Every paired credential can act as their business, so finish by making sure they know where revocation lives — **and it is not the CLI**.

**The public CLI ships no key-management commands.** There is no `keys list`, no `keys revoke`, no `keys rotate`, no kill-switch — those commands are not in the published package at all. That is deliberate: whoever issues a credential is the one who withdraws it, and the server refuses key-management calls made with an agent key.

**Revocation is in the Dashboard:** *Settings → Security → Sessions · Agents & Devices*. Every paired device is listed with what it can do, how long it has left, and a per-row revoke, plus an emergency revoke-all behind an explicit confirmation. Revoking takes effect immediately and **cannot be undone** — getting access back means authorizing again.

**What the CLI does have is local:**

```bash
allscale logout      # deletes the credential saved on THIS machine
```

Say this part plainly, because it is the one people get wrong: **`logout` changes nothing server-side.** If a laptop is lost or a credential may have leaked, the answer is Dashboard revocation. Logging out only removes the local copy — the credential keeps working for anyone who has it.

---

## Debugging

| Symptom | Cause and fix |
|---|---|
| `allscale: command not found` | npm global bin not on PATH. `npm prefix -g`, add its `bin/` to PATH, or use `npx @allscale/cli` |
| Node version error | Needs ≥ 20.10. Check `node -v`, upgrade per Step 0 |
| Exit `6` on a command that used to work | A scope is missing, or a one-time web-app step is pending. Run `allscale scope` and compare. Do not retry — it will keep failing |
| Everything fails after a period of inactivity | The key hit the validity period chosen at authorization, and nothing renews it. `allscale whoami` shows the expiry; re-run `device-login` and pick a longer one if the work justifies it |
| `7530 CLI_AUTH_NO_GRANTABLE_SCOPE` | A requested scope cannot be granted — almost always `wallet` or `payroll`. Drop it |
| A key-management call is refused | Expected — key management is not in the public CLI, and the server rejects it from an agent key. Use the Dashboard |
| Exit `11` | Signing key in this build is rejected. `npm install -g @allscale/cli@latest`. Re-login will not fix it |
| Keychain unavailable on Linux | No libsecret. It falls back to `~/.allscale/credentials.json` (mode 0600). Only pass `--insecure-storage` if the user accepts that (Safety Rule 4) |
| Auth works, calls hit the wrong data | Check which profile is active (`--profile`). The endpoint cannot be changed — the published build only talks to `https://app.allscale.io` |
| `invoice send` rejects a total | USDT/USDC totals must be at least 0.10 |
| `payout send` fails on a fresh account | It needs a live store credential from Payout onboarding — a web-app step, not a CLI one |

---

## Reference

- `allscale --help` and `allscale <topic> --help` — **always authoritative over this guide**. If they disagree, believe the CLI and say so.
- `allscale operations` / `allscale describe <operation>` — the schema describes itself; use it instead of guessing
- [AllScale API documentation](https://docs.allscale.io/allscale-checkout/getting-started)
- `/integrate-allscale` — a different product surface: embedding AllScale Checkout in an app via the HMAC-signed API. If the user wants to take payments *in their own app*, they want that guide, not this one.
