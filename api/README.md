# Open API

The StableMesh Open API lets you issue and manage virtual Visa and Mastercard cards from your own servers. It is a server-to-server REST API, authenticated with a secret API key that you create yourself in the dashboard.

## One account, two ways in

The API works on **the same account you use in the StableMesh dashboard**. A card you create through the API appears in the dashboard, and a card you create in the dashboard is visible to the API. Balances, card statuses and transactions are identical on both sides — there is no separate "API account" to fund or reconcile.

***

## Base URLs

| Environment    | API base URL                                 | Dashboard                                         |
| -------------- | -------------------------------------------- | ------------------------------------------------- |
| **Production** | `https://api.stablemesh.io`                  | `https://app.stablemesh.io`                       |
| **Sandbox**    | `https://api-server-dev-1440.up.railway.app` | `https://stablemesh-dashboard-dev.up.railway.app` |

Sandbox and production are fully separate: separate accounts, separate API keys, separate balances. Sandbox cards cannot make real purchases — use the [sandbox helpers](wallet.md#sandbox-helpers) to simulate deposits and payments.

***

## How money moves

1. **Funding account.** Your account holds one balance, in **USD**. You top it up by sending **USDT** (or **USDC**, where the network accepts it) on-chain to your deposit address; it is credited 1:1 as USD.
2. **Cards draw from it.** Creating a card, or topping one up, debits your funding account and credits the card — minus fees.
3. **And back.** You can move money from a card back to your funding account with a withdrawal.

```
  USDT / USDC on-chain ──► Funding account (USD) ──► Card (USD) ──► merchants
                          ▲                       │
                          └────── withdraw ───────┘
```

***

## Conventions

* Every request is `POST` with a JSON body, under `/v1` — for example `POST https://api.stablemesh.io/v1/card/list`.
* Send `X-API-KEY` and `Content-Type: application/json` on every request. See [Authentication](authentication.md).
* Every response has the same envelope:

  ```json
  { "code": 0, "message": "ok", "data": { } }
  ```

  `code` is `0` on success and non-zero on error. See [Errors](errors.md).
* Amounts in responses are **strings** (`"49.5"`), so no precision is lost. Requests accept either a number or a string. Card and account amounts are always in **USD**.
* Timestamps are ISO 8601 in UTC, e.g. `2026-09-29T08:30:00Z`.
* **Card operations are asynchronous at the card issuer.** Issuing, freezing, unfreezing, withdrawing and cancelling are accepted first and completed shortly after. [Cards](cards.md) explains how to follow each one.
* Rate limit: **20 requests per second** per API key.

***

## Endpoints

| Area         | Endpoint                                                   | Purpose                                                |
| ------------ | ---------------------------------------------------------- | ------------------------------------------------------ |
| Account      | `POST /v1/account/balance`                                 | Funding-account balance                                |
| Account      | `POST /v1/account/limits`                                  | How many more cards you can open                       |
| Account      | `POST /v1/account/transactions`                            | Account history (the dashboard's Transactions page)    |
| Deposits     | `POST /v1/deposit/networks`                                | Networks and stablecoins you can deposit               |
| Deposits     | `POST /v1/deposit/address`                                 | Your deposit address on a network                      |
| Cards        | `POST /v1/card/products`                                   | Card products you can issue, with fees                 |
| Cards        | `POST /v1/card/create`                                     | Issue a card                                           |
| Cards        | `POST /v1/card/list`                                       | All your cards                                         |
| Cards        | `POST /v1/card/detail`                                     | One card                                               |
| Cards        | `POST /v1/card/balance`                                    | A card's balance and how much can be withdrawn         |
| Cards        | `POST /v1/card/deposit`                                    | Top up a card                                          |
| Cards        | `POST /v1/card/withdraw`                                   | Move money from a card back to your funding account    |
| Cards        | `POST /v1/card/withdraw/status`                            | Follow a withdrawal                                    |
| Cards        | `POST /v1/card/freeze` · `unfreeze`                        | Block / resume spending                                |
| Cards        | `POST /v1/card/cancel`                                     | Close a card                                           |
| Cards        | `POST /v1/card/update`                                     | Rename a card                                          |
| Transactions | `POST /v1/card/transactions`                               | Card payments, refunds and fees                        |
| Webhooks     | `POST /v1/webhook/get` · `set` · `rotateSecret` · `delete` | Push events to your server                             |
| Sandbox      | `POST /v1/sandbox/deposit` · `spend`                       | Simulate a deposit or a card payment (sandbox only)    |

***

## What the API does not do

* **Card number, CVV and expiry are never returned by the API.** You view them in the dashboard (Cards → the card → Sensitive Info), protected by your card passcode. The same applies to the cardholder name and billing address.
* **On-chain withdrawals** from your funding account to an external address are made in the dashboard only.

***

## Get started

Create an account in the dashboard, then open **Settings → API keys → Create key**. The [Integration Guide](integration-guide.md) takes you from there to a funded, working card.
