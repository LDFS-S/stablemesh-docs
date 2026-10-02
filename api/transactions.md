# Transactions

`POST /v1/card/transactions`

Payments made with your cards — purchases, refunds, reversals and card fees — newest first. Query one card, or all your cards at once.

For your whole account's history — deposits, card top-ups and withdrawals as well as card payments — see [`account/transactions`](wallet.md#account-history).

***

## Request

```json
{ "cardId": "2026092917803181873402", "pageNumber": 1, "pageSize": 20 }
```

| Field        | Required | Description                                          |
| ------------ | -------- | ---------------------------------------------------- |
| `cardId`     | no       | Restrict to one card. Omit for all your cards.       |
| `pageNumber` | no       | 1-based. Default `1`.                                |
| `pageSize`   | no       | Default `20`, maximum `100`.                         |

A `pageNumber` past the last page returns an empty `items` list.

***

## Response

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "items": [
      {
        "transactionId": "7392018475610293847",
        "cardId": "2026092917803181873402",
        "type": "PURCHASE",
        "status": "COMPLETED",
        "amount": "11.62",
        "currency": "EUR",
        "billAmount": "12.5",
        "billCurrency": "USD",
        "fee": "0.19",
        "merchant": { "name": "GOOGLE ADS", "country": "IE", "mcc": "7311" },
        "description": "GOOGLE*ADS1234",
        "transactionTime": "2026-09-29T10:15:03Z"
      }
    ],
    "pageNumber": 1,
    "pageSize": 20,
    "totalItems": 92,
    "totalPages": 5,
    "hasMore": true
  }
}
```

***

## The transaction object

| Field             | Description                                                                                   |
| ----------------- | --------------------------------------------------------------------------------------------- |
| `transactionId`   | Stable identifier. The same transaction keeps its id as its status changes.                   |
| `type`            | `PURCHASE`, `REFUND`, `REVERSAL`, `FEE` or `VERIFICATION` — see below                          |
| `status`          | See [Statuses](#statuses)                                                                     |
| `amount`, `currency` | What the merchant charged, in the merchant's currency                                      |
| `billAmount`, `billCurrency` | What it cost the card, in USD. This is what moved your card balance.               |
| `fee`             | Network fees on this transaction (FX, cross-border), in USD. `"0"` if none.                   |
| `merchant`        | `name`, `country` (ISO 3166-1 alpha-2) and `mcc` (merchant category code)                     |
| `description`     | The merchant descriptor as the network reported it                                            |
| `transactionTime` | When the transaction happened                                                                 |

| `type`         | Meaning                                                                   |
| -------------- | ------------------------------------------------------------------------- |
| `PURCHASE`     | A payment at a merchant                                                   |
| `REFUND`       | A merchant returned money to the card                                     |
| `REVERSAL`     | An authorisation was cancelled before it settled                          |
| `FEE`          | A card fee charged by the network                                         |
| `VERIFICATION` | A zero- or low-value check a merchant runs when you save the card         |

***

## Statuses

| Status      | Meaning                                                                         |
| ----------- | ------------------------------------------------------------------------------- |
| `PENDING`   | Authorised. The amount is **held** on the card (`balance.frozen`), not yet settled. |
| `COMPLETED` | Settled. The amount has left the card.                                          |
| `FAILED`    | Declined. Nothing was taken.                                                    |
| `REFUNDED`  | Money was returned to the card.                                                 |
| `REVERTED`  | The authorisation was reversed and the hold released.                           |

A transaction usually starts `PENDING` and moves to one of the others. Receive each change as a [`transaction.updated`](webhooks.md#transaction-events) webhook instead of polling.

***

## Example

```bash
curl -X POST https://api.stablemesh.io/v1/card/transactions \
  -H "X-API-KEY: $SM_KEY" -H "Content-Type: application/json" \
  -d '{"cardId":"2026092917803181873402","pageNumber":1,"pageSize":20}'
```
