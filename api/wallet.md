# Funding & Deposits

Your account has one **funding account**, held in **USD**. It pays for issuing and topping up cards, and receives money you withdraw from cards. You fund it by depositing stablecoins on-chain — **USDT**, and **USDC** on the networks that accept it.

***

## Funding flow

1. `POST /v1/deposit/networks` — see which networks you can deposit on.
2. `POST /v1/deposit/address` — get your address on one of them.
3. Send one of that network's `currencies`, **on that network**, to that address.
4. Once the transfer is confirmed on-chain, your funding account is credited **1:1 in USD** — usually within 5 minutes of confirmation. Poll `account/balance`, or wait for the [`deposit.credited`](webhooks.md#deposit-events) webhook.

{% hint style="warning" %}
**Send only a currency the network lists in `currencies`, and only on the network you requested the address for.** Other tokens, or a token sent over a different network, may be lost.
{% endhint %}

In sandbox, skip the on-chain step with [`sandbox/deposit`](#sandbox-helpers).

***

## Funding-account balance

`POST /v1/account/balance`

**Request** `{}`

**Response**

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "currency": "USD",
    "available": "850",
    "frozen": "0",
    "onCards": { "available": "147", "frozen": "0" },
    "total": "997"
  }
}
```

In this example, 1,000 USD was deposited, then a card was issued with 50 and topped up with 100: 850 remains in the funding account, 147 is on the card, and the 3 USD difference is the deposit fees.

| Field       | Description                                                                 |
| ----------- | --------------------------------------------------------------------------- |
| `available` | What you can spend on issuing and topping up cards right now                |
| `frozen`    | Held in the funding account for an operation still in progress              |
| `onCards`   | The sum of your cards' balances — the same `available` / `frozen` split     |
| `total`     | Everything: funding account plus cards                                      |

This is the simplest authenticated call, and a good way to check a new key.

***

## Card limit

`POST /v1/account/limits`

**Request** `{}`

**Response**

```json
{ "code": 0, "message": "ok", "data": { "maxCards": 10, "used": 2, "remaining": 8 } }
```

`used` counts the cards that count toward your limit. When `remaining` is `0`, `card/create` returns `4003`. Contact your account manager if you need a higher limit.

***

## Account history

`POST /v1/account/transactions`

Everything that moved your money, newest first — the same list, in the same order, as the **Transactions** page in the dashboard: deposits and withdrawals, money moved onto and off your cards, card payments, fees, and transfers. For one card's payments with full merchant detail, see [`card/transactions`](transactions.md).

**Request**

```json
{ "type": "CARD_TOPUP", "pageNumber": 1, "pageSize": 20 }
```

| Field        | Required | Description                                  |
| ------------ | -------- | -------------------------------------------- |
| `type`       | no       | Only rows of this type — see below           |
| `pageNumber` | no       | 1-based. Default `1`.                        |
| `pageSize`   | no       | Default `20`, maximum `100`.                 |

**Response**

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "items": [
      {
        "id": "SBX4B8C56378C2546DEBE8A4E76",
        "type": "CARD_SPEND",
        "direction": "OUT",
        "amount": "5",
        "currency": "USD",
        "settledAmount": "5",
        "settledCurrency": "USD",
        "fee": null,
        "status": "COMPLETED",
        "network": null,
        "txHash": null,
        "counterparty": "GOOGLE ADS",
        "cardId": "2026092917906572283009",
        "last4": null,
        "createdAt": "2026-09-29T04:48:34Z"
      },
      {
        "id": "7c0e5a1f-2b9d-4e63-8f1a-6d2c9b0e4a17",
        "type": "CARD_TOPUP",
        "direction": "OUT",
        "amount": "30",
        "currency": "USD",
        "fee": null,
        "status": "COMPLETED",
        "network": null,
        "txHash": "ORD20260929…",
        "counterparty": null,
        "cardId": "2026092917906572283009",
        "last4": null,
        "createdAt": "2026-09-29T04:47:09Z"
      },
      {
        "id": "b6f0a1c2-7d3e-4f5a-9b8c-1d2e3f4a5b6c",
        "type": "DEPOSIT",
        "direction": "IN",
        "amount": "100",
        "currency": "USDT",
        "fee": "0",
        "status": "COMPLETED",
        "network": "TRX",
        "txHash": "4f3c…9a1e",
        "counterparty": "TQ7r…u2Kd",
        "cardId": null,
        "last4": null,
        "createdAt": "2026-09-29T04:47:01Z"
      }
    ],
    "pageNumber": 1,
    "pageSize": 20,
    "totalItems": 3,
    "totalPages": 1,
    "hasMore": false
  }
}
```

| Field            | Description                                                                                          |
| ---------------- | ---------------------------------------------------------------------------------------------------- |
| `type`           | What happened — see below                                                                            |
| `direction`      | `IN` or `OUT` — which way the money went. `amount` is always positive.                               |
| `amount`         | In `currency`. For a card payment abroad this is the **merchant's** currency.                        |
| `settledAmount`  | Card payments only: what the card paid, in `settledCurrency` (USD).                                 |
| `fee`            | The fee on this row, when there is one; otherwise `null`. Card deposit fees are in the `charges` of the `card/create` and `card/deposit` responses. |
| `status`         | `COMPLETED`, `PENDING` or `FAILED`                                                                   |
| `network`, `txHash` | On-chain rows: the network and transaction hash. For other rows `txHash` is StableMesh's own reference. |
| `counterparty`   | The other side, when known: the sending address of a deposit, the destination of a withdrawal, the merchant of a card payment, or the (masked) email of a transfer |
| `cardId`, `last4`| The card involved, if any. `last4` only when the card's issuer reports it on the row — otherwise use `card/detail`. |

| `type`             | Meaning                                                                    |
| ------------------ | -------------------------------------------------------------------------- |
| `DEPOSIT`          | Stablecoin received on-chain                                               |
| `WITHDRAW`         | On-chain withdrawal from your funding account (made in the dashboard)      |
| `CARD_TOPUP`       | Money moved from your funding account onto a card — issuing or top-up      |
| `CARD_REFUND`      | Money back from a card — a withdrawal, a cancellation, or a merchant refund |
| `CARD_SPEND`       | A payment made with a card                                                 |
| `CARD_FEE`         | A card opening fee                                                         |
| `SUBSCRIPTION_FEE` | A plan fee                                                                 |
| `REFERRAL_REWARD`  | Referral reward credited                                                   |
| `P2P_SENT`, `P2P_RECEIVED` | A transfer to or from another StableMesh user                      |
| `CREDIT`, `DEBIT`  | An adjustment made by StableMesh                                           |
| `OTHER`            | Anything else                                                              |

New types may be added — treat any you don't recognise by its `direction`.

{% hint style="info" %}
`CARD_TOPUP` and `CARD_REFUND` move money between your **own** balances — funding account and card. They change where your money is rather than how much you have, except for the card deposit fee, which is taken out of the amount moved onto the card.
{% endhint %}

***

## Deposit networks

`POST /v1/deposit/networks`

**Request** `{}`

**Response**

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "networks": [
      { "network": "TRX", "name": "TRON",     "currencies": ["USDT"] },
      { "network": "ETH", "name": "Ethereum", "currencies": ["USDC", "USDT"] },
      { "network": "ARB", "name": "Arbitrum", "currencies": ["USDC"] }
    ]
  }
}
```

The list above is an example. Always read it from the API — networks can be added or paused.

***

## Deposit address

`POST /v1/deposit/address`

**Request**

```json
{ "network": "TRX", "currency": "USDT" }
```

| Field      | Required | Description                                  |
| ---------- | -------- | -------------------------------------------- |
| `network`  | yes      | A `network` from `deposit/networks`          |
| `currency` | yes      | One of that network's `currencies`           |

**Response**

```json
{
  "code": 0,
  "message": "ok",
  "data": { "network": "TRX", "currency": "USDT", "address": "TXa1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6" }
}
```

The address is yours and doesn't change — calling again returns the same one. Error `11001` if the network isn't in `deposit/networks`; `400` if the network doesn't accept that currency.

***

## Sandbox helpers

These two endpoints exist **only in sandbox**. In production they return `404`.

### Simulate a deposit

`POST /v1/sandbox/deposit`

Credits your sandbox funding account as if a stablecoin deposit had arrived on-chain. You also receive `deposit.credited`.

**Request**

```json
{ "amount": "500" }
```

`amount` is at most `1000` per call.

**Response** — the new [balance](#funding-account-balance).

### Simulate a card payment

`POST /v1/sandbox/spend`

Runs a payment through a sandbox card, exactly as a real merchant charge would: the amount is held, then settled, and the transaction appears in `card/transactions` with its webhooks.

**Request**

```json
{ "cardId": "2026092917803181873402", "amount": "12.5", "merchantName": "GOOGLE ADS", "outcome": "COMPLETED" }
```

| Field          | Required | Description                                                                        |
| -------------- | -------- | ---------------------------------------------------------------------------------- |
| `cardId`       | yes      | An `ACTIVE` sandbox card                                                           |
| `amount`       | yes      | USD                                                                                |
| `merchantName` | no       | Default `SANDBOX MERCHANT`                                                         |
| `outcome`      | no       | `COMPLETED` (default), `PENDING` (held, never settled) or `FAILED` (declined)       |

A sandbox `PENDING` payment is never settled, so the card it was made on keeps that amount in `balance.frozen` and **cannot be cancelled** (`4007`). Try the pending case on a card you don't plan to cancel.

**Response** — the resulting [transaction](transactions.md#the-transaction-object).
