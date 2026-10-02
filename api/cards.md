# Cards

All card endpoints are `POST` under `/v1`, authenticated with `X-API-KEY`. Cards are virtual Visa or Mastercard cards denominated in **USD**, funded from your [funding account](wallet.md).

> **Make money-moving calls retry-safe.** Send an `Idempotency-Key` header on `card/create`, `card/deposit`, `card/withdraw`, `card/freeze`, `card/unfreeze` and `card/cancel`. See [Idempotency](idempotency.md).

***

## The card object

```json
{
  "cardId": "2026092917803181873402",
  "binId": "MC-WSB-512631",
  "productName": "Mastercard 512631",
  "network": "Mastercard",
  "last4": "6654",
  "alias": "Ads – EU",
  "status": "ACTIVE",
  "pendingOperation": null,
  "balance": { "available": "147", "frozen": "0", "currency": "USD" },
  "createdAt": "2026-09-29T08:30:15Z",
  "operations": ["TOP_UP", "WITHDRAW", "FREEZE", "UNFREEZE", "CANCEL"]
}
```

| Field              | Description                                                                                         |
| ------------------ | --------------------------------------------------------------------------------------------------- |
| `cardId`           | Our identifier for the card. Use it in every card call.                                             |
| `binId`            | The product the card was issued from — see [List products](#list-products).                        |
| `last4`            | Last four digits of the card number. `null` until the card is `ACTIVE`.                             |
| `alias`            | Your label for the card (max 50 characters).                                                        |
| `status`           | `PENDING`, `ACTIVE`, `FROZEN` or `CANCELLED` — see below.                                           |
| `pendingOperation` | `FREEZE`, `UNFREEZE` or `CANCEL` while the issuer is still processing one; otherwise `null`.        |
| `balance`          | `available` can be spent; `frozen` is held for payments not yet settled or a withdrawal in progress. |
| `operations`       | The changes the API accepts on this card: `TOP_UP`, `WITHDRAW`, `FREEZE`, `UNFREEZE`, `CANCEL`. See [What each card accepts](#what-each-card-accepts). |

The same cards, with the same values, appear in the dashboard.

***

## Statuses and what you can do

```
card/create ──► PENDING ──(usually < 1 min)──► ACTIVE ◄───── unfreeze ─────┐
                                                 │                          │
                                                 ├────── freeze ──────► FROZEN
                                                 │
                                                 └────── cancel ──────► pendingOperation: CANCEL ──(~30 days)──► CANCELLED
```

| Status      | Meaning                                                              |
| ----------- | -------------------------------------------------------------------- |
| `PENDING`   | The issuer is still provisioning the card. Nothing can be done yet.  |
| `ACTIVE`    | Ready to use.                                                        |
| `FROZEN`    | New payments are declined. Unfreeze to use it again.                 |
| `CANCELLED` | Closed for good. Its remaining balance has been returned to you.     |

| When the card is…                          | Top up | Withdraw | Freeze | Unfreeze | Cancel | Rename |
| ------------------------------------------ | :----: | :------: | :----: | :------: | :----: | :----: |
| `PENDING`                                  |        |          |        |          |        |   ✓    |
| `ACTIVE`                                   |   ✓    |    ✓     |   ✓    |          |   ✓    |   ✓    |
| `FROZEN`                                   |        |          |        |    ✓     |        |   ✓    |
| `CANCELLED`                                |        |          |        |          |        |   ✓    |
| any status, with a `pendingOperation` set  |        |          |        |          |        |   ✓    |

A call outside this table is refused with code `4014` (wrong status), `4015` (a freeze/unfreeze is still processing) or `4008` (a cancellation is pending) — and nothing changes.

### What each card accepts

Not every product supports every change through the API. Each card says which ones it does in `operations`, and each product in `card/products` says the same for a new card of it:

| `operations` holds | Through the API you can                                   | Everything else                          |
| ------------------ | --------------------------------------------------------- | ---------------------------------------- |
| all five           | top up, withdraw, freeze, unfreeze, cancel                | —                                        |
| `FREEZE`, `UNFREEZE` | freeze and unfreeze                                     | top up, withdraw and cancel in the dashboard |
| nothing (`[]`)     | view it and rename it                                     | in the dashboard                         |

A change the card doesn't list is refused with `4011`, and nothing changes. The status rules above still apply on top. Renaming works on every card.

{% hint style="info" %}
**A frozen card can't be topped up or withdrawn from.** Unfreeze it first. This is a rule of the card issuer, not a StableMesh setting.
{% endhint %}

***

## Fees

| When           | You pay from the funding account | Fees taken out of it                     | Lands on the card                           |
| -------------- | -------------------------------- | ---------------------------------------- | ------------------------------------------- |
| Issuing a card | `amount` (some products: `amount + openFee`) | `openFee` + `amount × depositFeeRate` | `amount − openFee − depositFee` (those products: `amount − depositFee`) |
| Topping up     | `amount`                         | `amount × depositFeeRate`                | `amount − depositFee`, rounded down to cents |
| Withdrawing    | —                                | the issuer may charge a withdrawal fee   | `receivedAmount` reaches your funding account |

* Most products take `openFee` out of `amount`. Some charge it **on top** — you pay `amount + openFee` and the opening fee doesn't reduce what lands on the card. Either way `charges.totalCharged` in the `card/create` response is exactly what left your funding account.
* `openFee` and `depositFeeRate` are per product and returned by [List products](#list-products). The rate shown is the one that applies to **your** account, including any rate you negotiated with us.
* Example at `depositFeeRate = 0.02`, `openFee = 0`: issuing with `amount = 50` puts **49** on the card; topping up **100** adds **98**.
* Spending-side fees (FX, cross-border) are charged by the network per payment and appear on each [transaction](transactions.md).

***

## List products

`POST /v1/card/products`

The card products your account can issue.

**Request** `{}`

**Response**

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "products": [
      {
        "binId": "MC-WSB-512631",
        "name": "Mastercard 512631",
        "network": "Mastercard",
        "bin": "512631",
        "description": "Virtual Mastercard for global online payments",
        "fees": { "openFee": "0", "depositFeeRate": "0.02" },
        "minimumFirstDeposit": "20",
        "worksWith": {
          "ads": ["meta", "google", "tiktok"],
          "aiTools": ["chatgpt"],
          "wallets": [],
          "physicalCard": false
        },
        "operations": ["TOP_UP", "WITHDRAW", "FREEZE", "UNFREEZE", "CANCEL"]
      }
    ]
  }
}
```

| Field                 | Description                                                                                                      |
| --------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `minimumFirstDeposit` | Smallest `amount` accepted by `card/create` for this product.                                                    |
| `operations`          | What the API accepts on a card of this product — see [What each card accepts](#what-each-card-accepts).         |
| `worksWith`           | Platforms the issuer has **confirmed** for this product. A platform that isn't listed is *not confirmed* — it may still work. |

***

## Issue a card

`POST /v1/card/create`

Issues a card from a product and funds it from your funding account.

**Request**

```json
{ "binId": "MC-WSB-512631", "amount": "50", "alias": "Ads – EU" }
```

| Field    | Required | Description                                                                         |
| -------- | -------- | ----------------------------------------------------------------------------------- |
| `binId`  | yes      | A product from `card/products`                                                      |
| `amount` | yes      | USD taken from your funding account. Fees come out of it — see [Fees](#fees).       |
| `alias`  | no       | A label for the card, max 50 characters                                             |

**Response**

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "card": {
      "cardId": "2026092917803181873402",
      "binId": "MC-WSB-512631",
      "productName": "Mastercard 512631",
      "network": "Mastercard",
      "last4": null,
      "alias": "Ads – EU",
      "status": "PENDING",
      "pendingOperation": null,
      "balance": { "available": "49", "frozen": "0", "currency": "USD" },
      "createdAt": "2026-09-29T08:30:15Z",
      "operations": ["TOP_UP", "WITHDRAW", "FREEZE", "UNFREEZE", "CANCEL"]
    },
    "charges": { "amount": "50", "openFee": "0", "depositFee": "1", "cardCredit": "49", "totalCharged": "50" }
  }
}
```

`charges.totalCharged` is what left your funding account; `charges.cardCredit` is what landed on the card.

**Most cards start `PENDING`** while the issuer provisions it — usually **under a minute**. (Some products are `ACTIVE` as soon as `card/create` returns; you then receive `card.created` and `card.activated` together.) Its `last4` is not known yet. Poll `card/detail` every few seconds until `status` is `ACTIVE`, or wait for the [`card.activated`](webhooks.md#card-events) webhook. If a card is still `PENDING` after 10 minutes, contact support with its `cardId`.

Your funding account is debited when the call succeeds, not when the card activates.

| Error   | Meaning                                                                                                   |
| ------- | --------------------------------------------------------------------------------------------------------- |
| `3001`  | Funding account balance is below `amount`                                                                 |
| `4003`  | You've reached your card limit — see [`account/limits`](wallet.md#card-limit) |
| `4004`  | Unknown `binId`                                                                                           |
| `4005`  | This product isn't available to your account                                                              |
| `4012`  | Cardholder verification is still in progress. **Nothing was charged.** Retry in a few minutes.            |
| `4013`  | This product requires a verified cardholder on your account — contact support                             |
| `8001`  | `amount` is below `minimumFirstDeposit`, or too small to leave enough on the card after fees (**10 USD** for some products) |

***

## List cards

`POST /v1/card/list`

**Request**

```json
{ "status": "ACTIVE" }
```

`status` is optional — omit it to get every card. Cancelled cards stay in the list with status `CANCELLED`.

The list shows **every** card on your account, including any of a product not offered through the API (for example one issued earlier in the dashboard). Those can be viewed and renamed, but any other change returns `4011` — their `operations` is empty.

**Response**

```json
{ "code": 0, "message": "ok", "data": { "cards": [ { "cardId": "2026092917803181873402", "status": "ACTIVE", "…": "…" } ] } }
```

Each item is a full [card object](#the-card-object).

***

## Card detail

`POST /v1/card/detail`

**Request** `{ "cardId": "2026092917803181873402" }`

**Response** — the [card object](#the-card-object), plus:

| Field             | Description                                                                                                       |
| ----------------- | ----------------------------------------------------------------------------------------------------------------- |
| `retainedMinimum` | A minimum balance some products keep on the card, which cannot be withdrawn. It is returned when the card is cancelled. `"0"` if none. |

***

## Card balance

`POST /v1/card/balance`

**Request** `{ "cardId": "2026092917803181873402" }`

**Response**

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "cardId": "2026092917803181873402",
    "available": "147",
    "frozen": "0",
    "retainedMinimum": "1",
    "maxWithdrawable": "146",
    "currency": "USD"
  }
}
```

`maxWithdrawable` is `available − retainedMinimum` — the most `card/withdraw` will accept right now.

***

## Top up a card

`POST /v1/card/deposit`

Moves USD from your funding account onto the card. **Synchronous** — the card is credited when the call returns.

**Request**

```json
{ "cardId": "2026092917803181873402", "amount": "100" }
```

**Response**

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "cardId": "2026092917803181873402",
    "amount": "100",
    "depositFee": "2",
    "cardCredit": "98",
    "balance": { "available": "147", "frozen": "0", "currency": "USD" }
  }
}
```

The card must be `ACTIVE`. Errors: `3001` (funding balance too low), `4014` / `4015` / `4008` (card state), `8001` (amount too small to cover the fee).

***

## Withdraw from a card

`POST /v1/card/withdraw`

Moves USD from the card **back to your funding account**. This is not a blockchain withdrawal.

**Request**

```json
{ "cardId": "2026092917803181873402", "amount": "20" }
```

| Field    | Required | Description                                              |
| -------- | -------- | -------------------------------------------------------- |
| `cardId` | yes      | An `ACTIVE` card                                         |
| `amount` | yes      | At least `0.01`, at most `maxWithdrawable` from `card/balance` |

**Response**

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "withdrawalId": "3f2b8c1e-5d4a-4f9b-9c1e-7a6d5b4c3e21",
    "cardId": "2026092917803181873402",
    "amount": "20",
    "status": "PENDING",
    "receivedAmount": null,
    "fee": null,
    "failureReason": null,
    "createdAt": "2026-09-29T09:12:40Z",
    "completedAt": null
  }
}
```

**Withdrawals are asynchronous.** The moment the call returns, `amount` is held on the card (it moves from `available` to `frozen`). When the issuer completes it — usually within a few minutes, occasionally longer — `receivedAmount` is credited to your funding account and the status becomes `COMPLETED`. If the issuer declines it, the hold is released back to the card and the status becomes `FAILED`. Either way nothing is lost or doubled.

Follow it with [`card/withdraw/status`](#withdrawal-status) or the `card.withdrawal.completed` / `card.withdrawal.failed` [webhooks](webhooks.md#withdrawal-events).

| Error   | Meaning                                                                                                             |
| ------- | ------------------------------------------------------------------------------------------------------------------- |
| `3001`  | `amount` is above `maxWithdrawable` — the message names the maximum                                                 |
| `4014` / `4015` / `4008` | The card is not in a state that allows withdrawals                                                  |
| `9999`  | **The outcome could not be confirmed with the issuer.** The amount is on hold and will complete or be returned automatically. The error includes `data.withdrawalId` — follow that withdrawal instead of sending a new one. |

## Withdrawal status

`POST /v1/card/withdraw/status`

**Request** `{ "withdrawalId": "3f2b8c1e-5d4a-4f9b-9c1e-7a6d5b4c3e21" }`

**Response** — the withdrawal object shown above.

| `status`    | Meaning                                                                          |
| ----------- | -------------------------------------------------------------------------------- |
| `PENDING`   | Held on the card, waiting for the issuer                                         |
| `COMPLETED` | `receivedAmount` credited to your funding account; `fee` is what the issuer kept |
| `FAILED`    | Declined; the hold was released back to the card. `failureReason` says why.      |

Error `8102` if the `withdrawalId` doesn't exist on your account.

***

## Freeze / Unfreeze

`POST /v1/card/freeze` · `POST /v1/card/unfreeze`

**Request** `{ "cardId": "2026092917803181873402" }`

**Response** — the [card object](#the-card-object).

Both are **asynchronous**. The response usually shows the change still in progress — though the issuer sometimes completes it inside the call, and then the response already shows the final `status` with `pendingOperation: null`:

```json
{ "code": 0, "message": "ok", "data": { "cardId": "2026092917803181873402", "status": "ACTIVE", "pendingOperation": "FREEZE", "…": "…" } }
```

It normally completes within seconds: `status` becomes `FROZEN` (or `ACTIVE`) and `pendingOperation` returns to `null`. Poll `card/detail`, or wait for `card.frozen` / `card.unfrozen`. While `pendingOperation` is set, other changes to the card are refused with `4015`.

A frozen card declines new payments. Freeze only works on an `ACTIVE` card, unfreeze only on a `FROZEN` one; otherwise `4014`.

***

## Cancel a card

`POST /v1/card/cancel`

Closes the card **permanently** and returns its balance to your funding account.

**Request** `{ "cardId": "2026092917803181873402" }`

**Response** — the [card object](#the-card-object), with `pendingOperation: "CANCEL"`, plus `heldAmount` — the balance that will be returned.

**Before you cancel:**

* The card must be `ACTIVE` — a frozen card must be unfrozen first. Otherwise `4014`.
* No payment may still be pending and no withdrawal in progress (`balance.frozen` is `0`). Otherwise `4007` — wait for the activity to settle and try again.

**After you cancel:**

1. The card can no longer be used for new payments. **This cannot be undone.**
2. Its balance is held, and returned to your funding account when the issuer settles the cancellation — **about 30 days later.**
3. Until then the card shows `pendingOperation: "CANCEL"` and accepts no other changes.
4. On settlement the status becomes `CANCELLED` and you receive `card.cancelled`.

{% hint style="info" %}
**Want the money back sooner?** [Withdraw](#withdraw-from-a-card) the balance first — withdrawals usually complete in minutes — then cancel. Only the product's `retainedMinimum`, if any, waits the 30 days.
{% endhint %}

In **sandbox**, a cancellation settles within seconds, so you see `card.cancelled` right away.

***

## Rename a card

`POST /v1/card/update`

**Request** `{ "cardId": "2026092917803181873402", "alias": "Ads – US" }`

**Response** — the updated [card object](#the-card-object). Allowed in any status.

***

## Card numbers

The card number, CVV, expiry, cardholder name and billing address are **not available through the API**. To use a card at a merchant, open it in the dashboard — **Cards → the card → Sensitive Info** — and confirm with your card passcode.
