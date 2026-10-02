# Idempotency

Network calls fail. When a request times out you often can't tell whether it went through — retrying `card/create` could issue **two** cards, retrying `card/deposit` could fund a card **twice**. An idempotency key makes a retry safe: a repeated request returns the **original** response instead of running again.

***

## How it works

Send an `Idempotency-Key` header with a unique value — a UUID is ideal — for each operation you want to be retry-safe:

```http
POST /v1/card/create HTTP/1.1
Host: api.stablemesh.io
X-API-KEY: smk_live_…
Idempotency-Key: 1f0c8a2e-4b9d-4f3a-8b21-7c6e5d4a3b21
Content-Type: application/json

{ "binId": "MC-WSB-512631", "amount": "50" }
```

* The **first** request with a key runs normally, and its response is stored.
* A **retry with the same key and the same body** returns the stored response unchanged — no second card, no second charge. It carries the response header `Idempotent-Replayed: true`.
* A key is any string up to **255 characters**; longer is refused with `400`.
* Keys are scoped to **your account** and kept for **24 hours**. The key you sent is echoed back in the `Idempotency-Key` response header.

### Which responses are stored

| First response                        | Stored? | A retry with the same key…                               |
| ------------------------------------- | ------- | -------------------------------------------------------- |
| `2xx` success                         | yes     | returns the same success                                 |
| `5xx` (code `9999`)                   | yes     | returns the same error — see below                       |
| `4xx` (validation or business error)  | no      | runs again. Nothing happened the first time, so it is safe. |

A `5xx` means **we could not confirm the outcome** — the operation may or may not have happened. That is exactly when a blind retry is dangerous, so the error is stored and replayed. Check the actual state first (`card/list`, `card/balance`, `card/withdraw/status`), and only then decide whether to try again **with a new key**.

***

## Where it applies

| Endpoint                                  | Without a key, a retry could…                         |
| ----------------------------------------- | ----------------------------------------------------- |
| `POST /v1/card/create`                    | issue a second card and charge you twice              |
| `POST /v1/card/deposit`                   | top the card up twice                                 |
| `POST /v1/card/withdraw`                  | start a second withdrawal                             |
| `POST /v1/card/freeze` · `unfreeze` · `cancel` | send a second state change                      |

Read-only endpoints don't need a key; they are always safe to repeat.

***

## Edge cases

| Situation                                     | Result                                                                               |
| --------------------------------------------- | ------------------------------------------------------------------------------------ |
| Same key, **identical** request               | The stored response, with `Idempotent-Replayed: true`                                |
| Same key, **different** body                  | `409`, code `4211` — a key belongs to one exact request                              |
| Same key, first request **still running**     | `409`, code `4212` — back off briefly and retry with the same key                    |
| Same key, more than 24 hours later            | Treated as a new request                                                             |

***

## Best practices

* Generate **one key per logical operation** ("fund card X with 100"), and reuse it for every retry of that operation.
* Use a **random UUID** — never a predictable or shared value.
* Retry with the same key, with backoff, until you get a definitive answer: a `2xx` (possibly replayed) means done; `409 / 4212` means the original is still running.

> The key is **optional** by default: send it and it's honoured, omit it and the request behaves normally. We can make it **required** on your account's mutating calls — a missing key then returns `400` with code `4210`.
