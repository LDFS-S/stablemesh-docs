# Errors

Every response carries an application `code` in its JSON body, alongside a standard HTTP status. **Handle errors by `code`**, not only by HTTP status.

```json
{ "code": 3001, "message": "Insufficient USD balance. Available: 12.5, Required: 50" }
```

`message` is written for a human and may change; `code` is stable.

***

## HTTP statuses

| HTTP  | Meaning                                                                          |
| ----- | -------------------------------------------------------------------------------- |
| `200` | Success. `code` is `0`.                                                          |
| `400` | The request was understood and refused — a validation or business error. **Nothing changed.** |
| `401` | Missing, unknown, revoked or expired API key                                     |
| `403` | Valid key, but the request came from an address outside its IP allowlist         |
| `404` | Unknown endpoint (including sandbox helpers in production)                       |
| `409` | Idempotency conflict                                                             |
| `429` | Rate limit exceeded — back off and retry                                         |
| `500` | We could not complete or confirm the operation — see `9999` below                |

***

## Codes

| `code`  | Name                         | HTTP  | Meaning and what to do                                                                                   |
| ------- | ---------------------------- | ----- | -------------------------------------------------------------------------------------------------------- |
| `0`     | SUCCESS                      | 200   | OK                                                                                                       |
| `400`   | INVALID_PARAMETER            | 400   | A required field is missing or malformed. `message` names it.                                            |
| `1111`  | INSUFFICIENT_FUNDS           | 400   | Like `3001`, when another request spent the funding balance first — two card calls at once with only enough for one. Check `account/balance` before retrying. |
| `3001`  | INSUFFICIENT_BALANCE         | 400   | Funding account or card balance too low, or a withdrawal above `maxWithdrawable`. `message` gives the numbers. |
| `4001`  | CARD_NOT_FOUND               | 400   | No such card on your account                                                                             |
| `4003`  | CARD_LIMIT_REACHED           | 400   | You've reached your card limit — see `account/limits`                                                    |
| `4004`  | PRODUCT_NOT_FOUND            | 400   | Unknown `binId`                                                                                          |
| `4005`  | PRODUCT_NOT_AVAILABLE        | 400   | This product isn't available to your account                                                             |
| `4007`  | CARD_HAS_PENDING_ACTIVITY    | 400   | A payment or withdrawal on the card hasn't settled yet. Wait and retry.                                  |
| `4008`  | CARD_IS_CANCELLING           | 400   | The card has a pending cancellation; only reading is allowed                                             |
| `4011`  | OPERATION_NOT_SUPPORTED      | 400   | This card doesn't accept that change through the API — see the card's `operations`; make it in the dashboard |
| `4012`  | CARDHOLDER_PENDING           | 400   | Cardholder verification is in progress. **Nothing was charged.** Retry in a few minutes.                 |
| `4013`  | CARDHOLDER_REJECTED          | 400   | This product needs a verified cardholder on your account — contact support                               |
| `4014`  | CARD_STATE_NOT_ALLOWED       | 400   | Not allowed in the card's current status — see [Cards](cards.md#statuses-and-what-you-can-do) |
| `4015`  | CARD_OPERATION_IN_PROGRESS   | 400   | A freeze or unfreeze is still processing. Retry in a few seconds.                                        |
| `4210`  | IDEMPOTENCY_KEY_REQUIRED     | 400   | Your account requires an `Idempotency-Key` on this call                                                  |
| `4211`  | IDEMPOTENCY_KEY_MISMATCH     | 409   | This `Idempotency-Key` was already used with a different request                                         |
| `4212`  | IDEMPOTENCY_KEY_IN_PROGRESS  | 409   | A request with this `Idempotency-Key` is still running. Retry shortly with the same key.                 |
| `8001`  | AMOUNT_TOO_SMALL             | 400   | Below the product minimum, or too small to cover fees (a new card must keep at least 10 USD after fees)  |
| `8102`  | WITHDRAWAL_NOT_FOUND         | 400   | No such `withdrawalId` on your account                                                                   |
| `11001` | NETWORK_NOT_SUPPORTED        | 400   | Deposits aren't available on this network — see `deposit/networks`                                       |
| `11002` | AUTH_FAILED                  | 401   | Missing, unknown, revoked or expired API key                                                             |
| `11003` | IP_NOT_ALLOWED               | 403   | Request came from an address not on the key's IP allowlist                                               |
| `11004` | RATE_LIMITED                 | 429   | More than 20 requests per second on this key                                                             |
| `9999`  | SYSTEM_ERROR                 | 500   | See below                                                                                                |

New codes may be added. Treat an unknown non-zero `code` as a failure and log its `message`.

***

## When you get `9999`

`9999` means we could not complete **or could not confirm** the operation. On read-only calls, simply retry with backoff.

On calls that move money, **don't retry blindly**:

1. Check the real state — `card/list` after `card/create`, `card/balance` after `card/deposit`, `card/withdraw/status` after `card/withdraw` (the error includes `data.withdrawalId`).
2. If the operation did not happen, retry **with a new** `Idempotency-Key`.

A `card/withdraw` that returns `9999` is never lost: its amount stays on hold on the card and is either completed or released automatically.
