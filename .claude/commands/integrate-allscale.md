You are helping a developer integrate Allscale Checkout into their app or website. Follow these steps in order. Be conversational — go step by step, confirm each step works before moving on.

---

## Who this guide is for

Before you start, figure out which kind of integration this is:

- **Single-tenant** — you're integrating Allscale for your OWN use. Your Allscale credentials live in your app's `.env` and never change at runtime. Steps 1–8 cover everything.
- **Multi-tenant / platform** — you're building an app where END USERS paste their own Allscale credentials into your UI (e.g. a dashboard where each store owner pastes their own API key + secret). You have one additional obligation: validate every set of pasted credentials before storing them, and re-validate on every rotation. See **Step 4.5**.

Ask the developer which one applies before proceeding.

---

## CRITICAL SAFETY RULES

You MUST follow these rules at all times:

1. **NEVER write API Secret values into any source code file, config file, or any file that is not `.env`.**
2. **NEVER log, print, or echo the API Secret in any code you generate.**
3. **ALWAYS ensure `.env` is in `.gitignore` BEFORE writing credentials to `.env`.** Check first, add it if missing.
4. **NEVER commit `.env` files.** If you see one staged in git, warn the developer immediately.
5. **All API signing MUST happen server-side.** Never generate code that signs requests in frontend/browser code. The API Secret must never be in any client-side bundle.
6. **ALWAYS add rate limiting** to any endpoint you create that calls the Allscale API.
7. **ALWAYS validate amounts server-side** — never trust client-sent values without bounds checking.

---

## Step 0: Allscale Account and Credentials

Before writing any code, ask the developer:

> Do you have an Allscale account with Commerce enabled? You'll need:
>
> 1. An **Allscale account** — sign up at [allscale.io](https://allscale.io)
> 2. **Allscale Commerce** enabled on your account
> 3. A **store** created in the Allscale Commerce dashboard
> 4. Your **API Key** and **API Secret** (generated in the dashboard — the secret is shown only once)
>
> If you don't have these yet, go to [allscale.io](https://allscale.io) to get set up, or contact the Allscale BD team for help.

**Do NOT proceed until they confirm they have their API Key and API Secret.**

Once they confirm, ask them to provide:
- Their **API Key** (starts with `st_`)
- Their **API Secret** (starts with `st_`)

Then immediately proceed to set up their `.env` file safely (Step 1).

---

## Step 1: Secure Credential Setup

**Before writing the `.env` file, you MUST:**

1. Check if `.gitignore` exists. If not, create it.
2. Check if `.env` is listed in `.gitignore`. If not, add it.
3. Only THEN create or update the `.env` file with their credentials.

Write a `.env` file:

```
ALLSCALE_API_KEY=<their api key>
ALLSCALE_API_SECRET=<their api secret>
ALLSCALE_BASE_URL=https://openapi.allscale.io
ALLSCALE_CURRENCY=USD
```

Also create a `.env.example` file (safe to commit) with placeholder values:

```
ALLSCALE_API_KEY=your_api_key_here
ALLSCALE_API_SECRET=your_api_secret_here
ALLSCALE_BASE_URL=https://openapi.allscale.io
ALLSCALE_CURRENCY=USD
```

Tell them:
- There is one base URL: `https://openapi.allscale.io`. Sandbox has been retired — to build and test without real payments, create a **test store** in the Allscale dashboard and use its API credentials
- Available currency codes: `USD`, `EUR`, `GBP`, `CAD`, `AUD`, `JPY`, `CNY`, `SGD`, `HKD`

---

## Step 2: Understand Their Project

Now ask:

1. **What are you building?** (donation page, e-commerce checkout, tipping feature, subscription flow, etc.)
2. **What's your tech stack?** (framework, language, platform — e.g., Next.js, Flask, Rails, vanilla JS, mobile, etc.)
3. **Where will you deploy?** (Vercel, Netlify, AWS, self-hosted, etc.)

Adapt all code in the following steps to their specific stack.

---

## Step 3: Implement API Authentication (HMAC-SHA256 Request Signing)

Every Allscale API request must be signed. This is the most critical part.

### Required headers on every request:

| Header | Description |
|---|---|
| `X-API-Key` | Their API key |
| `X-Timestamp` | Unix timestamp in seconds |
| `X-Nonce` | Random unique string (UUID recommended) |
| `X-Signature` | `v1=<signature>` |

### Signing algorithm:

1. Build a **canonical string** by joining these with newline (`\n`):
   ```
   METHOD          (e.g., "POST", "GET" — uppercase)
   PATH            (e.g., "/v1/checkout_intents/")
   QUERY_STRING    (e.g., "" for no query, or "key=value" without the ?)
   TIMESTAMP       (same value as X-Timestamp header)
   NONCE           (same value as X-Nonce header)
   BODY_SHA256     (SHA-256 hex digest of the raw request body; for GET with no body, hash the empty string "")
   ```

2. Compute signature:
   ```
   signature = Base64( HMAC-SHA256( api_secret, canonical_string ) )
   ```

3. Set header:
   ```
   X-Signature: v1=<signature>
   ```

### Reference implementations:

**Node.js:**
```javascript
import crypto from "crypto";

function signRequest(method, path, query, body, apiSecret) {
  const timestamp = Math.floor(Date.now() / 1000).toString();
  const nonce = crypto.randomUUID();
  const bodyHash = crypto.createHash("sha256").update(body).digest("hex");

  const canonical = [method, path, query, timestamp, nonce, bodyHash].join("\n");
  const signature = crypto.createHmac("sha256", apiSecret).update(canonical).digest("base64");

  return {
    "X-Timestamp": timestamp,
    "X-Nonce": nonce,
    "X-Signature": `v1=${signature}`,
  };
}
```

**Python:**
```python
import hmac, hashlib, base64, time, uuid

def sign_request(method, path, query, body, api_secret):
    timestamp = str(int(time.time()))
    nonce = str(uuid.uuid4())
    body_hash = hashlib.sha256(body.encode()).hexdigest()

    canonical = "\n".join([method, path, query, timestamp, nonce, body_hash])
    signature = base64.b64encode(
        hmac.new(api_secret.encode(), canonical.encode(), hashlib.sha256).digest()
    ).decode()

    return {
        "X-Timestamp": timestamp,
        "X-Nonce": nonce,
        "X-Signature": f"v1={signature}",
    }
```

**Important gotchas:**
- `BODY_SHA256` must be computed from the **raw body string**, not a parsed/re-serialized object
- For GET requests (no body), hash the empty string `""`
- Timestamp must be within ±10 minutes of server time or request is rejected (default; the merchant may configure a smaller window per-store via `replay_window_seconds`)
- Each nonce can only be used once — always generate a fresh UUID

Write the signing utility in whatever language/framework they're using. Read the API secret from the environment variable `ALLSCALE_API_SECRET` — NEVER hardcode it.

---

## Step 4: Test Connectivity

Before building the checkout flow, verify auth works. Write a small test script or route that calls all three test endpoints:

1. `GET /v1/test/ping` — should return `{"code":0,"payload":{"pong":"ok"}}`
2. `GET /v1/test/fail` — should return an error response (this is expected, it tests error handling)
3. `POST /v1/test/post` — send a JSON body, the server echoes it back in `payload`

Run the tests and confirm all three pass before proceeding. If they get error code `20002` (bad signature), help them debug — see the debugging section at the bottom.

If you're building a multi-tenant app where end users paste their own credentials, ALSO read Step 4.5 — you need to wire the `/v1/test/ping` call into your user-onboarding form, not just your one-time dev smoke test.

---

## Step 4.5: Validating credentials from end users (multi-tenant apps only)

Skip this step if you're a single-tenant integrator. If your app accepts an `apiKey` + `apiSecret` from a user — account onboarding, credential rotation, switching environments — you must validate at paste time.

Anywhere your app accepts pasted Allscale credentials, call `GET /v1/test/ping` with those credentials BEFORE writing them to your database. Same signed HTTP call you wrote in Step 4; just invoke it from your form handler.

If the probe fails:
- Do NOT store the credentials.
- Do NOT redirect the user to a "setup complete" page.
- Re-render the form with a specific error message keyed to the failure mode (table below).

**Why this matters:** without upfront validation, bad credentials silently make it into your database. The user thinks setup worked. The error surfaces minutes to days later — at first checkout or first webhook — by which time the user has lost context and may blame your app instead of their own bad paste. Validate at paste time.

### Map probe results to user-facing messages

| Probe result | What to tell the user |
|---|---|
| code `20002` (invalid signature) | "The API secret is incorrect — re-copy it from your Allscale dashboard." |
| code `20001` / HTTP 401 | "The API key isn't recognized." |
| code `30001` (IP forbidden) | "Allscale's IP allowlist rejected our server. Add your server's outbound IP to your Allscale API settings." |
| Network timeout (~5s) | "Couldn't reach Allscale — try again in a moment." |
| Any other failure | "Allscale returned an error. Please try again." |

### Wire the same probe into your credential-rotation handler

A user who pastes a wrong new secret while updating credentials will otherwise silently break payments until the next checkout — and rotation is when this bug bites hardest, because the old (working) credentials get overwritten.

### Anti-patterns to avoid

- Relying on the first checkout attempt as your validation surface — the error arrives too late and the user no longer associates it with their paste.
- Relying on webhook signature mismatches as your validation surface — same problem, more delayed.
- Treating the test endpoint as "optional once your signing works" — it's required at every user-facing credential-accept moment.

---

## Step 5: Create Checkout Intent

This is the core payment flow. The server creates a checkout intent, gets back a hosted checkout URL, and opens that URL for the user to pay.

### Endpoint: `POST /v1/checkout_intents/`

**Important:** The trailing slash is required.

### Request body:

```json
{
  "currency": 1,
  "amount_cents": 500,
  "order_id": "order_123",
  "order_description": "Monthly subscription",
  "user_id": "user_456",
  "user_name": "Tom",
  "redirect_url": "https://example.com/checkout/allscale",
  "extra": {
    "source": "web"
  }
}
```

For native stable-coin pricing (priced in USDT directly, no FX), use `stable_coin` instead of `currency`:

```json
{
  "stable_coin": 1,
  "amount_cents": 1000,
  "order_id": "order_124",
  "redirect_url": "https://example.com/checkout/allscale"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `currency` | int or null | one of* | Fiat currency as integer enum (see table below) |
| `stable_coin` | int or null | one of* | Stable-coin enum for native stable-coin pricing (see table below). Currently only USDT (`1`) is supported |
| `amount_cents` | int | YES | Amount in cents. With `currency`: fiat cents ($5.00 = 500). With `stable_coin`: cents of the stable-coin (1000 = 10.00 USDT) |
| `order_id` | string or null | no | Your internal order ID |
| `order_description` | string or null | no | Description shown to payer |
| `user_id` | string or null | no | Your internal user ID |
| `user_name` | string or null | no | Payer display name |
| `redirect_url` | string or null | no | Where to send the user after payment completes |
| `extra` | object or null | no | Arbitrary metadata |

*Exactly one of `currency` or `stable_coin` must be set — not both, not neither. Minimum payment is **0.1 USDT**.

**CRITICAL: `currency` and `stable_coin` must be integers, NOT strings. Do NOT send `"USD"` — send `1`. Same for `stable_coin`: send `1`, not `"USDT"`.**

### Currency enum (common values):

| Value | Code |
|---|---|
| 1 | USD |
| 9 | AUD |
| 27 | CAD |
| 31 | CNY |
| 44 | EUR |
| 48 | GBP |
| 57 | HKD |
| 72 | JPY |
| 126 | SGD |

### Stable-coin enum:

| Value | Code | Status |
|---|---|---|
| 1 | USDT | Supported |
| 2 | USDC | Disabled |
| 3 | BUSD | Disabled |

### Successful response:

```json
{
  "code": 0,
  "payload": {
    "checkout_url": "https://checkout.allscale.io/abc123",
    "allscale_checkout_intent_id": "65b2f3d0d2d9c0a1b2c3d4e5",
    "amount_coins": "5.0000",
    "stable_coin_type": 1,
    "rate": "1.0000"
  },
  "error": null,
  "request_id": "req_xxxxx"
}
```

- `checkout_url` — redirect or open this URL for the user to pay
- `allscale_checkout_intent_id` — save this to poll status later
- `rate` — fiat→stable-coin exchange rate as a decimal string. `null` when the intent was created with `stable_coin` (native pricing, no FX)
- Settlement is currently **USDT only** (`stable_coin_type: 1`)

### What to do with the response:

- **Web app:** Open `checkout_url` in a new tab (`window.open`) or redirect (`window.location.href`)
- **Mobile app:** Open `checkout_url` in an in-app browser or system browser
- **API/backend:** Return the `checkout_url` to your frontend

### Security requirements for the checkout endpoint:

When writing the server-side route that calls this API, you MUST:
1. **Rate limit** the endpoint (e.g., 5 requests per minute per IP)
2. **Validate the amount** server-side (positive number, within reasonable bounds)
3. **Read credentials from environment variables only**

---

## Step 6: Poll Payment Status

After opening the checkout URL for the user, poll to know when they've paid.

### Endpoint: `GET /v1/checkout_intents/{intent_id}/status`

No request body. Returns:

```json
{
  "code": 0,
  "payload": 20,
  "error": null,
  "request_id": "req_xxxxx"
}
```

`payload` is the status integer:

| Value | Name | Meaning | Terminal? |
|---|---|---|---|
| -5 | TIMEOUT | Intent stayed pending too long and expired | Yes |
| -4 | CANCELED | User canceled | Yes |
| -3 | UNDERPAID | Paid less than required | Yes |
| -2 | REJECTED | Failed KYT checks | Yes |
| -1 | FAILED | Processing error | Yes |
| 1 | CREATED | Intent created, not yet opened | No |
| 2 | PAYING | User is on the checkout page (committed to paying, no funds received yet) | No |
| 3 | TEMP_WALLET_RECEIVED | Deposit wallet assigned | No |
| 4 | PENDING_MANUAL_OPERATION | Pending manual review | No |
| 5 | SEND_BACK | Refund in progress | No |
| 10 | ON_CHAIN | Transaction detected, awaiting confirmation | No |
| 20 | CONFIRMED | Payment confirmed on-chain | Yes |

**Recommended polling strategy:**
- Poll every 5 seconds
- Stop when status is terminal (negative values or 20)
- Timeout after 10 minutes
- Show user-friendly messages for each state transition

### Full intent details (optional):

`GET /v1/checkout_intents/{intent_id}` returns the complete object. Key fields:

| Field | Type | Notes |
|---|---|---|
| `currency` | int or null | `null` when created with `stable_coin` (native pricing) |
| `currency_symbol` | string or null | e.g., `"USD"`. `null` for native pricing |
| `currency_rate` | string or null | Decimal string. `null` when no FX conversion |
| `amount_cents` | int | Original amount in cents |
| `amount_coins` | string | Stable-coin amount (decimal string) |
| `coin_symbol` | string | e.g., `"USDT"` |
| `coin_contract` | string | ERC-20 contract address |
| `chain_id` | int | EIP-155 chain ID |
| `status` | int | Status enum (see table above) |
| `tx_hash` | string or null | On-chain transaction hash |
| `tx_from` | string or null | Sender wallet address |
| `tx_to` | string or null | Receiver wallet address |
| `payment_method_type` | int | `0`=UNKNOWN, `1`=WALLET_SCAN, `2`=WALLET_CONNECT, `3`=ALL_SCALE_PAY |
| `actual_paid_amount` | string or null | Amount actually received on-chain |
| `service_fee_amount` | string or null | Platform fee deducted |
| `net_income_amount` | string or null | After-fee amount to merchant |

---

## Step 7: Webhook Verification (Optional but Recommended)

If they configure a webhook URL in the Allscale dashboard, Allscale sends a POST to their server when payment is confirmed.

### Webhook headers:

| Header | Description |
|---|---|
| `X-API-Key` | API key |
| `X-Webhook-Id` | Unique webhook ID |
| `X-Webhook-Timestamp` | Unix timestamp |
| `X-Webhook-Nonce` | Unique nonce |
| `X-Webhook-Signature` | `v1=<signature>` |

### Webhook signature verification:

The canonical string format for webhooks is **different** from regular API requests:

```
allscale:webhook:v1
METHOD
PATH
QUERY_STRING
WEBHOOK_ID
TIMESTAMP
NONCE
BODY_SHA256
```

Note the prefix line `allscale:webhook:v1` — this is NOT present in regular API signing.

Then: `expected = Base64( HMAC-SHA256( api_secret, canonical ) )`

Compare with timing-safe equality against the signature in the header.

### Webhook payload fields:

| Field | Type | Description |
|---|---|---|
| `all_scale_transaction_id` | string | Allscale transaction ID |
| `all_scale_checkout_intent_id` | string | Checkout intent ID |
| `webhook_id` | string | Must match X-Webhook-Id header |
| `amount_cents` | int or null | Fiat amount in cents. `null` for native stable-coin pricing |
| `currency` | int or null | Currency enum. `null` for native stable-coin pricing |
| `currency_symbol` | string or null | e.g., `"USD"`. `null` for native stable-coin pricing |
| `amount_coins` | string | Stable-coin amount (decimal string) |
| `coin_symbol` | string | e.g., `"USDT"` |
| `coin_contract_address` | string | ERC-20 contract address |
| `chain_id` | int | EIP-155 chain ID |
| `tx_hash` | string | On-chain transaction hash |
| `tx_from` | string | Sender wallet address |
| `payment_method_type` | int | `0`=UNKNOWN, `1`=WALLET_SCAN, `2`=WALLET_CONNECT, `3`=ALL_SCALE_PAY |
| `order_id` | string or null | Your order ID |
| `user_id` | string or null | Your user ID |
| `user_name` | string or null | Payer display name |
| `extra_obj` | object or null | Arbitrary metadata passed at intent creation |

### Verification checklist:
1. Validate timestamp is within ±5 minutes
2. Check nonce hasn't been used before (store in Redis/memory with TTL)
3. Compute body SHA-256 from **raw bytes before JSON parsing**
4. Build canonical string and verify signature
5. Only process payload after verification passes
6. Respond with 200 OK

---

## Step 8: Response Signing (Optional)

Response signing lets the client verify each API response came from Allscale and hasn't been tampered with. It is **off by default** — only implement this if the merchant has enabled it on their store (`signed_response = true` in the store config). If not enabled, skip this step.

### Response headers (when enabled):

| Header | Description |
|---|---|
| `X-Response-Timestamp` | Server-generated Unix timestamp |
| `X-Response-Nonce` | Unique per-response nonce |
| `X-Response-Signature` | `v1=<signature>` |
| `X-Request-Nonce` | Echo of the request's `X-Nonce` |
| `X-Request-Id` | Unique request identifier |

### Canonical string:

```
STATUS_CODE
PATH
REQUEST_NONCE
REQUEST_BODY_SHA256
RESPONSE_TIMESTAMP
RESPONSE_NONCE
RESPONSE_BODY_SHA256
```

Fields joined with `\n`.

- `REQUEST_BODY_SHA256` — SHA-256 hex of the body you sent (hash of `""` for GET requests).
- `RESPONSE_BODY_SHA256` — SHA-256 hex of the raw response body bytes, **before** JSON parsing.

### Algorithm:

Same as request signing: `Base64( HMAC-SHA256( api_secret, canonical_string ) )`. Compare timing-safe against `X-Response-Signature` (strip the `v1=` prefix).

Reject the response if verification fails.

---

## Step 9: Claim Link Auto-Payout (Optional — sending money out)

Everything above is about **taking** money (checkout). This step is the reverse: **paying money out** to a recipient with a single API call — a payout, disbursement, refund, reward, etc.

The **Claim Link Auto-Payout** endpoint creates a claim link AND funds it automatically from your own custodied wallet in one call, then returns a shareable **claim URL** you deliver to the recipient. There is no manual on-chain deposit step — AllScale funds the link for you.

> If instead you want the sender-funded flow (you deposit into the link's pool wallet yourself), that's a different payout path — not covered here.

### ⚠️ Prerequisite: turn on Payout in Store Settings FIRST

**This endpoint is off by default. You must enable the Payout capability on your store before it will work.**

1. Open your store in the Allscale dashboard → **Store Settings**.
2. Find the **Payout** (auto-payout) section and **enable it**.
3. Complete the one-time onboarding it walks you through (it sets up the signing session and a spending-limit policy for your wallet).

Enabling Payout is what grants your API key the `claim_link:auto_payout` permission. **Until you do this, every call to this endpoint fails with a permission error (see the error table below).** If you are getting `403` / scope errors, this is almost always the reason — go back and enable Payout in Store Settings.

Also note:
- This is a **production capability** — it moves real funds. A sandbox/test-only key cannot use it.
- Funding draws from **your** wallet, so it must hold enough of the stablecoin to cover the payout **amount plus fees**. (Gas is sponsored — you don't need to hold native gas.)

### Endpoint: `POST /v1/claim_link_auto_payouts`

Signed exactly like every other request (Step 3) — same headers, same HMAC canonical string.

### Request body:

```json
{
  "amount": "10.50",
  "stable_coin": 1,
  "chain": 5,
  "receiver_email": "recipient@example.com",
  "sender_display_name": "Acme Payouts",
  "sender_message": "Your payout is ready",
  "reference_id": "payout_0001"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `amount` | string | YES | Recipient's claimable amount in **token units** as a **string** (e.g. `"10.50"`), not a JSON number. Must be positive and no finer than the token's decimals. |
| `stable_coin` | int | YES | Stablecoin enum (same integer enums as checkout). |
| `chain` | int | YES | Chain enum (same integer enums as the rest of the API). |
| `receiver_email` | string \| null | no | If set, AllScale emails the claim invite to the recipient. |
| `sender_display_name` | string \| null | no | Shown to the recipient as the sender (max 100 chars). |
| `sender_message` | string \| null | no | Short note shown to the recipient (max 500 chars). |
| `reference_id` | string | YES* | **Your idempotency key.** Re-sending the same `reference_id` returns the original link instead of creating a second one. Required by default. |

**About `reference_id` — do not skip it.** This call is **synchronous**: it blocks (up to ~90s) while the funding transaction settles. The most likely failure you'll hit is a client-side timeout followed by a retry — and without a stable `reference_id`, that retry creates a **second** link and a **second** debit. With it, retries are safe: the second call just returns the first link (`idempotent_hit: true`), no double-charge.

> Same rules as checkout: `amount` is a **string**; `stable_coin` / `chain` are **integers**, not `"USDT"` / `"BASE"`.

### Successful response:

```json
{
  "code": 0,
  "payload": {
    "claim_link_id": "665b2f3d0d2d9c0a1b2c3d4e",
    "reference_id": "payout_0001",
    "amount": "10.50",
    "token_symbol": "USDT",
    "status": "pending_deposit",
    "token": "clk_live_9f8a...c1",
    "claim_url": "https://claim.allscale.io/clk_live_9f8a...c1",
    "funding_tx_hash": "0xabc...def",
    "funded_amount": "10.50",
    "idempotent_hit": false
  },
  "error": null,
  "request_id": "req_xxxxx"
}
```

- `claim_url` — **deliver this to your recipient over a secure channel.** Anyone with it can claim the funds.
- `token` — the raw bearer claim token, returned **once**; treat it as a secret.
- `funding_tx_hash` / `funded_amount` — proof this call funded the link.
- `idempotent_hit` — `true` when the call matched a prior `reference_id` and returned the existing link (no new debit). On an idempotent replay, `token` / `funding_tx_hash` / `funded_amount` may be `null`.

### If it errors, here's why (read this before filing a bug)

Treat **any** non-zero `code` as a failure, and always surface `error` + `request_id` when asking for support.

| Code | HTTP | What actually went wrong | What to do |
|---|---|---|---|
| Scope forbidden — "not authorized for this capability" (`30002`) | 403 | **Payout is not enabled on your store**, so your key doesn't carry the payout permission. | Go to **Store Settings → enable Payout** and finish onboarding. This is the #1 cause. |
| Auto-payout disabled (`50104`) | 403 | The payout capability is off for this key/environment (e.g. a sandbox/test key, or the feature isn't live yet). | Use a production key on a store where Payout is enabled. |
| Create / auto-fund failed (`50103`) | 400 | The link couldn't be funded — most commonly **your wallet doesn't hold enough** of the stablecoin to cover amount + fees; also signing-session / policy not ready. | Top up your wallet, or re-check that Payout onboarding completed. The `reason` field says which. |
| Validation error (`10001`) | 400 / 422 | Bad input — missing/blank `reference_id`, `amount` not positive or too precise, wrong field types (sending `"USDT"` instead of the integer), unsupported coin. | Fix the field named in the error. |
| Duplicate `reference_id` in flight (`50105`) | 409 | A concurrent create for the **same** `reference_id` is still running. | Wait and re-poll with the same `reference_id`; don't spin up a parallel call. |
| Missing/invalid auth (`20001` / `20002`) | 401 | Auth headers missing or signature wrong. | See the signing debug section below. |
| Rate limited (`40001`) | 429 | Too many requests. | Back off and retry. |

**The single most common mistake** is calling this endpoint before enabling Payout in Store Settings and expecting it to work. If you see a `403`, don't debug your signing code — go enable Payout first.


---

## Debugging Signature Errors (Error Code 20002)

If they get `20002` (bad signature), check these in order:

1. Is the body hash computed from the **exact raw body string** (not re-serialized)?
2. Is the canonical string joined with `\n` (not `\r\n` or other separators)?
3. Is the path exactly right? `POST /v1/checkout_intents/` needs the **trailing slash**.
4. Is the query string an empty string `""` (not `undefined`, `null`, or missing)?
5. Is the timestamp fresh (within ±10 minutes of current UTC time by default; the merchant store may have configured a smaller `replay_window_seconds`)?
6. Is the API secret correct with no extra whitespace?
7. Is the HMAC using SHA-256 and the output encoded as Base64 (not hex)?

---

## Error Codes Reference

| Code | Meaning |
|---|---|
| 10001 | Validation error |
| 20001 | Missing authentication headers |
| 20002 | Invalid signature |
| 30001 | Forbidden / IP not allowed |
| 40001 | Rate limit exceeded |
| 50001 | Checkout intent not found |
| 50002 | Failed to create checkout intent |
| 50103 | Claim link auto-payout create/fund failed (e.g. wallet balance too low) |
| 50104 | Auto-payout disabled for this key/environment |
| 50105 | Duplicate reference_id create in progress |
| 30002 | Scope forbidden — store lacks the payout permission (enable Payout in Store Settings) |
| 90000 | Internal server error |
| 99999 | Unknown error |

---

## Working Example

Point them to the Buy Me a Bagel repo (`allscale-io/buy_me_a_bagel`) as a complete working reference:
- `api/checkout.js` — checkout intent creation with HMAC signing and rate limiting
- `api/status.js` — status polling with signing
- `app.js` — frontend checkout flow with status polling UI

API documentation: https://github.com/allscale-io/AllScale_Third-Party_API_Doc
