You are helping a user install the Allscale CLI and connect it to their Allscale account. Follow these steps in order. Be conversational — go step by step, confirm each step works before moving on. Run the commands yourself where you can, and show the user the output.

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

1. **NEVER run `allscale keys revoke --all` unless the user explicitly asks for it in that turn.** It is an incident kill-switch: it revokes every agent key at once and instantly breaks every running automation. Never run it "to clean up".
2. **NEVER run a money-moving command without showing the user the exact command and getting explicit confirmation first.** That covers `wallet send`, `payout send`, `invoice pay`, `claim-link create`, `claim-link claim`.
3. **NEVER write any token, API secret, or `~/.allscale/credentials.json` content into a repo file, a chat message, a log, or a commit.** `allscale keys rotate` prints a new secret **once** — tell the user to save it themselves; do not echo it back or store it for them.
4. **NEVER pass `--insecure-storage` without telling the user what it does** (credentials land in a plaintext file at `~/.allscale/credentials.json` instead of the OS keychain) **and getting their agreement.**
5. **NEVER point the CLI at a non-production host unless the user asks.** Default is `https://app.allscale.io`. Changing `--api-base` / `ALLSCALE_API_BASE` sends their credentials somewhere else.
6. **Request the narrowest scopes that do the job.** Do not ask for `claim_link:all` or `store:all` when `invoice:read_only` is enough — see Step 3.
7. **`allscale logout` does NOT revoke anything server-side.** If a machine is lost or a key may be leaked, the fix is `allscale keys revoke` or the dashboard — never `logout` alone.

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
allscale login-device --device-label <a-name-for-this-machine>
```

What happens: a browser tab opens, the user confirms with the web-app session they are **already** logged into, picks the permissions on the approval screen, and the CLI receives its credentials.

Request scopes up front so the approval screen is pre-filled with the minimum:

```bash
allscale login-device --device-label tom-mbp \
  --scopes invoice:read_only --scopes claim_link:all
```

Scope strings are `<category>:<tier>`:

| Category | Tiers | Notes |
|---|---|---|
| `invoice` | `read_only`, `all` | |
| `contact` | `read_only`, `all` | |
| `transaction` | `read_only` | |
| `claim_link` | `read_only`, `all` | **`all` moves money** |
| `store` | `all` | **moves money** |

Things to tell the user plainly:

- **The final set is decided on the approval screen, and the server enforces it.** What they get can be *less* than what you requested — at mint time the chosen set is clipped to what their role may hold. Step 4 is how you find out.
- **`:all` implies `:read_only` for the same category only** — never across categories. `invoice:all` gives you invoice reads; it gives you nothing on `contact`.
- **`wallet` and `payroll` are not grantable.** Do not put them in `--scopes`. A request whose every scope clips away is refused outright rather than minting a dead key.
- **`cli_op:all` is appended automatically** to every pairing-minted key and never appears on the approval screen. It is CLI plumbing, not a permission the user granted — don't be alarmed by it in `allscale scope` output, and don't request it.
- Agent-key sessions are **default-deny**: an operation is reachable only if it is wired to a scope the key holds. Legacy cookie sessions are full-permission and their recorded scopes are advisory only — `allscale scope` tells you which kind you have.

The default credential is a **scoped agent key with an expiry**. It does **not** auto-renew — when it lapses, automation stops. Re-run `login-device` to renew.

> `--credential cookie` exists as an escape hatch and yields a legacy full-permission session where scopes are advisory only. Do not reach for it. If something only works with `--credential cookie`, that is worth reporting, not working around.

## Step 3B: Log in (no browser — server, container, CI)

Print the approval link instead of opening a tab, then approve it from any device:

```bash
allscale login-device --no-browser
```

If cross-device approval isn't practical either, use an email OTP:

```bash
allscale otp-login --email <their-email>
```

For a fully scripted flow, `allscale otp-send` first and pass the pair through:

```bash
allscale otp-login --email <their-email> --otp-id <id> --otp <code>
```

> `allscale login --email ... --password ...` also exists, but only works for **legacy payroll accounts**. Allscale Pay accounts use passkey + OTP. Do not offer password login unless they tell you they have a legacy payroll account, and if you do, use `--password-stdin` rather than putting a password in the command line:
> ```bash
> printf '%s' "$PW" | allscale login --email <their-email> --password-stdin
> ```

---

## Step 4: Verify — and check what they actually got

```bash
allscale whoami
allscale scope
```

**`allscale scope` is the one that matters.** It reports the permissions the server actually granted, which can be narrower than what was requested. Run it now and show the user the result.

Confirm out loud that the granted scopes cover what they told you in Step 2. If they don't, re-run `login-device` and have them grant the missing ones on the approval screen — do not try to work around a missing scope in code.

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

# Claim link. --per-transfer-gas is the gas you pre-fund for the recipient's claim.
allscale claim-link create --amount 10 --chain base --per-transfer-gas 0.02
```

Useful to know: **claiming a link needs no login at all.** This is the shortest path for a recipient who has no Allscale account:

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

**Schema self-description.** The CLI can describe its own operations, so an agent never has to guess argument shapes:

```bash
allscale operations                # every operation the schema exposes
allscale describe <operation>      # full arg types, return type, example document
```

**Branch on exit codes, not on stderr text.** They are stable and deliberately distinct:

| Exit | Meaning | What to do |
|---|---|---|
| `0` | Success | — |
| `1` | Internal error | Report it; not user-fixable |
| `2` | Bad input, or credential storage unusable | Fix the arguments |
| `3` | Network error or timeout | Retry with backoff |
| `4` | No credentials | Run `login-device` |
| `5` | Credentials expired | Re-run `login-device` — agent keys do not auto-renew |
| `6` | Permission denied / capability gap | **Not fixable by retrying or re-logging in.** A scope is missing, or the action needs a one-time step in the web app |
| `7` | Not found | Check the id |
| `8` | Rate limited | Back off |
| `9` | Backend error | Retry, then report |
| `10` | Raw mode disabled | — |
| `11` | Request signature rejected | **Upgrade the CLI** — this build's signing key is no longer accepted; re-login will not help |

Exit `6` versus exit `5` is the distinction worth wiring in: `5` means try logging in again, `6` means stop and tell a human.

---

## Step 7: Teach them how to take it back

Every paired credential can act as their business. Make sure they know both ways to revoke, before they walk away.

**From the terminal:**

```bash
allscale keys list                   # every agent key: status, scopes, expiry
allscale keys revoke <api-key-id>    # kill one key
allscale keys rotate <api-key-id>    # replace it, same scopes; NEW secret prints ONCE
allscale logout                      # clears local credentials only
```

**From the dashboard:** *Settings → Security → Sessions · Agents & Devices* lists every paired device with per-row revoke.

Three things to say explicitly:

- **`logout` is local only.** Lost laptop or a possibly-leaked key means `keys revoke`, not `logout`.
- **`keys` commands are control-plane and need a human login.** An agent key cannot revoke itself or any other key — that is deliberate. If they want an agent to be able to do this, they need a human session.
- **`keys revoke --all` is the incident kill-switch** — every key at once, every automation down. Never run it on their behalf without them asking (Safety Rule 1).

---

## Debugging

| Symptom | Cause and fix |
|---|---|
| `allscale: command not found` | npm global bin not on PATH. `npm prefix -g`, add its `bin/` to PATH, or use `npx @allscale/cli` |
| Node version error | Needs ≥ 20.10. Check `node -v`, upgrade per Step 0 |
| Exit `6` on a command that used to work | A scope is missing, or a one-time web-app step is pending. Run `allscale scope` and compare. Do not retry — it will keep failing |
| Everything fails after a period of inactivity | The agent key expired and does not auto-renew. `allscale keys list` shows expiry; re-run `login-device` |
| `7530 CLI_AUTH_NO_GRANTABLE_SCOPE` | A requested scope cannot be granted — almost always `wallet` or `payroll`. Drop it |
| `2102 SCOPE_NOT_GRANTED` on a `keys` command | `keys` is control-plane. The session is an agent key; log in as a human |
| Exit `11` | Signing key in this build is rejected. `npm install -g @allscale/cli@latest`. Re-login will not fix it |
| Keychain unavailable on Linux | No libsecret. It falls back to `~/.allscale/credentials.json` (mode 0600). Only pass `--insecure-storage` if the user accepts that (Safety Rule 4) |
| Auth works, calls hit the wrong data | Check `--api-base` / `ALLSCALE_API_BASE`. Default is `https://app.allscale.io` |
| `invoice send` rejects a total | USDT/USDC totals must be at least 0.10 |
| `payout send` fails on a fresh account | It needs a live store credential from Payout onboarding — a web-app step, not a CLI one |

---

## Reference

- `allscale --help` and `allscale <topic> --help` — **always authoritative over this guide**. If they disagree, believe the CLI and say so.
- `allscale operations` / `allscale describe <operation>` — the schema describes itself; use it instead of guessing
- [Allscale API documentation](https://docs.allscale.io/allscale-checkout/getting-started)
- `/integrate-allscale` — a different product surface: embedding Allscale Checkout in an app via the HMAC-signed API. If the user wants to take payments *in their own app*, they want that guide, not this one.
