# Integration Guide

From nothing to a funded card that has made a payment — every step is a copy-paste `curl`. Do it in **sandbox** first. Going live is a change of base URL and key; nothing else.

For field-level detail, see the reference pages: [Authentication](authentication.md), [Funding & Deposits](wallet.md), [Cards](cards.md), [Transactions](transactions.md), [Webhooks](webhooks.md), [Idempotency](idempotency.md), [Errors](errors.md).

***

## 1. Create an API key

1. Sign in to the **sandbox dashboard**: `https://stablemesh-dashboard-dev.up.railway.app`
2. Open **Settings → API keys → Create key**.
3. Name it. Optionally add your server's public IP to the allowlist.
4. Confirm with your card passcode or the email code.
5. **Copy the key.** It starts with `smk_test_` and is shown only once.

***

## 2. Set up your shell

```bash
export SM_BASE="https://api-server-dev-1440.up.railway.app"
export SM_KEY="smk_test_your_key_here"

sm() {
  local body="${2:-}"; [ -z "$body" ] && body='{}'
  curl -s -X POST "$SM_BASE$1" \
    -H "Content-Type: application/json" \
    -H "X-API-KEY: $SM_KEY" \
    -d "$body"
  echo
}
```

Every step below is now `sm /v1/…`.

***

## 3. Check your key

```bash
sm /v1/account/balance
```

```json
{ "code": 0, "message": "ok", "data": { "currency": "USD", "available": "0", "frozen": "0", "onCards": { "available": "0", "frozen": "0" }, "total": "0" } }
```

A `401` with code `11002` means the key is wrong or revoked; a `403` with `11003` means your IP isn't on the key's allowlist.

***

## 4. Fund your account

**Sandbox** — simulate a deposit:

```bash
sm /v1/sandbox/deposit '{"amount":"500"}'
```

**Production** — deposit real stablecoins:

```bash
sm /v1/deposit/networks
sm /v1/deposit/address '{"network":"TRX","currency":"USDT"}'
```

Send USDT on that network to the returned `address` (each network in `deposit/networks` lists the currencies it accepts). It is credited 1:1 as USD, usually within 5 minutes of on-chain confirmation. Poll `account/balance` or wait for `deposit.credited`.

***

## 5. Choose a product

```bash
sm /v1/card/products
```

```json
{
  "code": 0, "message": "ok",
  "data": { "products": [
    { "binId": "MC-WSB-512631", "name": "Mastercard 512631", "network": "Mastercard",
      "fees": { "openFee": "0", "depositFeeRate": "0.02" }, "minimumFirstDeposit": "20", "…": "…" }
  ] }
}
```

Keep the `binId` of the product you want. `fees` tells you exactly what a card costs.

***

## 6. Issue a card

```bash
curl -s -X POST "$SM_BASE/v1/card/create" \
  -H "Content-Type: application/json" -H "X-API-KEY: $SM_KEY" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{"binId":"MC-WSB-512631","amount":"50","alias":"First card"}'
```

```json
{
  "code": 0, "message": "ok",
  "data": {
    "card": { "cardId": "2026092917803181873402", "status": "PENDING", "last4": null,
              "balance": { "available": "49", "frozen": "0", "currency": "USD" }, "…": "…" },
    "charges": { "amount": "50", "openFee": "0", "depositFee": "1", "cardCredit": "49" }
  }
}
```

50 USD left your funding account; 49 is on the card after the 2% deposit fee.

**Wait for it to activate** — usually under a minute:

```bash
export CARD=2026092917803181873402
sm /v1/card/detail "{\"cardId\":\"$CARD\"}"      # repeat until "status": "ACTIVE"
```

In production, prefer the `card.activated` webhook (step 12) to polling.

***

## 7. Top up the card

```bash
sm /v1/card/deposit "{\"cardId\":\"$CARD\",\"amount\":\"100\"}"
```

```json
{ "code": 0, "message": "ok", "data": { "cardId": "2026092917803181873402", "amount": "100", "depositFee": "2", "cardCredit": "98",
  "balance": { "available": "147", "frozen": "0", "currency": "USD" } } }
```

Top-ups are immediate.

***

## 8. Make a payment

**Sandbox** — simulate one:

```bash
sm /v1/sandbox/spend "{\"cardId\":\"$CARD\",\"amount\":\"12.5\",\"merchantName\":\"GOOGLE ADS\"}"
```

**Production** — use the card at any merchant. To see its number, CVV and expiry, open it in the dashboard (**Cards → the card → Sensitive Info**).

Then read it back:

```bash
sm /v1/card/transactions "{\"cardId\":\"$CARD\"}"
```

***

## 9. Move money back to your funding account

```bash
sm /v1/card/balance "{\"cardId\":\"$CARD\"}"                       # note maxWithdrawable
sm /v1/card/withdraw "{\"cardId\":\"$CARD\",\"amount\":\"20\"}"      # → "status": "PENDING", withdrawalId
```

Withdrawals are asynchronous — the amount is held on the card at once and credited to your funding account when the issuer completes it, usually within minutes:

```bash
sm /v1/card/withdraw/status '{"withdrawalId":"3f2b8c1e-5d4a-4f9b-9c1e-7a6d5b4c3e21"}'   # until COMPLETED
```

***

## 10. Freeze and unfreeze

```bash
sm /v1/card/freeze "{\"cardId\":\"$CARD\"}"      # → "pendingOperation": "FREEZE"
sm /v1/card/detail "{\"cardId\":\"$CARD\"}"      # a few seconds later → "status": "FROZEN"
sm /v1/card/unfreeze "{\"cardId\":\"$CARD\"}"
```

A frozen card declines payments and can't be topped up or withdrawn from until you unfreeze it.

***

## 11. Cancel the card

```bash
sm /v1/card/cancel "{\"cardId\":\"$CARD\"}"      # → "pendingOperation": "CANCEL", "heldAmount": "…"
```

Cancelling is permanent. The card stops working at once; its remaining balance returns to your funding account when the issuer settles the cancellation, **about 30 days later**. To get most of the money back sooner, withdraw it before you cancel.

***

## 12. Receive webhooks

Register an HTTPS endpoint instead of polling:

```bash
sm /v1/webhook/set '{"url":"https://yourapp.example/stablemesh/webhook","enabled":true}'
```

Store the `secret` from the response — it is shown once. Verify the `StableMesh-Signature` header on every delivery, reply `2xx` quickly, and deduplicate on the event `id`. Full details and code samples: [Webhooks](webhooks.md).

***

## 13. Go live

* [ ] Create a **production** account at `https://app.stablemesh.io` and a `smk_live_` key, **with an IP allowlist**.
* [ ] Store the key in a secret manager — never in source control, apps or logs.
* [ ] Switch `SM_BASE` to `https://api.stablemesh.io`.
* [ ] Send an `Idempotency-Key` on every `create`, `deposit`, `withdraw`, `freeze`, `unfreeze` and `cancel`.
* [ ] Wait for `ACTIVE` before using a new card; treat `PENDING` as "not yet".
* [ ] Treat withdrawals and cancellations as asynchronous — follow them to a final status.
* [ ] Verify webhook signatures over the raw body; reply `2xx` fast; deduplicate on event `id`.
* [ ] Handle errors by `code`, and on a `9999` from a money-moving call, check the state before retrying.
* [ ] Fund with real USDT / USDC through `deposit/address` — sandbox helpers don't exist in production.
