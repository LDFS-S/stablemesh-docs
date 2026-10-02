# Webhooks

Webhooks push events to your server so you don't have to poll. StableMesh sends a signed `POST` to your endpoint when a card is issued or changes state, when a withdrawal finishes, when a card transaction is created or updated, and when a deposit reaches your funding account.

***

## Configure your endpoint

One endpoint per account, managed with your API key.

| Endpoint                        | Purpose                                                   |
| ------------------------------- | --------------------------------------------------------- |
| `POST /v1/webhook/get`          | View the current configuration (secret shown as a prefix); `configured: false` if none |
| `POST /v1/webhook/set`          | Create or update the endpoint                             |
| `POST /v1/webhook/rotateSecret` | Replace the signing secret                                |
| `POST /v1/webhook/delete`       | Remove the endpoint and stop all deliveries               |

**Create / update**

```bash
curl -X POST https://api.stablemesh.io/v1/webhook/set \
  -H "X-API-KEY: $SM_KEY" -H "Content-Type: application/json" \
  -d '{"url":"https://yourapp.example/stablemesh/webhook","enabled":true}'
```

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "configured": true,
    "url": "https://yourapp.example/stablemesh/webhook",
    "enabled": true,
    "secretPrefix": "whsec_AbCd12",
    "secret": "whsec_AbCd12…"
  }
}
```

| Field     | Required | Description                                                     |
| --------- | -------- | --------------------------------------------------------------- |
| `url`     | yes      | An **HTTPS** URL on the public internet, max 512 characters     |
| `enabled` | no       | `false` pauses deliveries without losing the configuration. Default `true`. |

{% hint style="warning" %}
**The full `secret` is returned only twice: when the endpoint is first created, and by `rotateSecret`.** Store it — you need it to verify every delivery. Later calls show only `secretPrefix`.
{% endhint %}

URLs that point at a private or internal network (localhost, `10.x`, `192.168.x`, cloud metadata addresses, …) are refused.

***

## Delivery

* Each event is sent as a JSON `POST` with a 10-second timeout. Reply with any **2xx** to acknowledge — do slow work after you reply.
* A non-2xx answer or a timeout is retried: **up to 6 attempts** in total, about 1 minute, 5 minutes, 30 minutes, 2 hours and 5 hours after each failure.
* Delivery is **at-least-once**: the same event can arrive more than once. **Deduplicate on the event `id`.**
* Events can arrive **out of order**. Use the `status` in the payload, not the order of arrival, to decide what is current — or re-read the object with the API.

Each request carries:

| Header                  | Value                                          |
| ----------------------- | ---------------------------------------------- |
| `StableMesh-Signature`  | `t=<unix seconds>,v1=<hex HMAC-SHA256>`        |
| `StableMesh-Event-Id`   | The event `id`                                 |
| `StableMesh-Event-Type` | The event `type`, e.g. `transaction.updated`   |

***

## Verify the signature

The signature is HMAC-SHA256, keyed with your `whsec_…` secret, over the string `"<t>.<raw request body>"`. Compute it over the **raw bytes** of the body, before any JSON parsing, compare in constant time, and reject timestamps older than 5 minutes.

**Python**

```python
import hmac, hashlib, time

def verify(secret: str, header: str, raw_body: bytes, tolerance: int = 300) -> bool:
    parts = dict(p.split("=", 1) for p in header.split(",") if "=" in p)
    t, v1 = parts.get("t"), parts.get("v1")
    if not t or not v1 or abs(time.time() - int(t)) > tolerance:
        return False
    expected = hmac.new(secret.encode(), f"{t}.".encode() + raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, v1)
```

**Node.js** (use `express.raw({ type: "*/*" })` so the body stays raw)

```js
const crypto = require("crypto");

function verify(secret, header, rawBody, tolerance = 300) {
  const parts = Object.fromEntries(
    header.split(",").map((p) => [p.slice(0, p.indexOf("=")), p.slice(p.indexOf("=") + 1)])
  );
  if (!parts.t || !parts.v1) return false;
  if (Math.abs(Date.now() / 1000 - Number(parts.t)) > tolerance) return false;
  const expected = crypto.createHmac("sha256", secret).update(`${parts.t}.`).update(rawBody).digest("hex");
  const a = Buffer.from(expected), b = Buffer.from(parts.v1);
  return a.length === b.length && crypto.timingSafeEqual(a, b);
}
```

***

## Event envelope

Every event has the same shape; `data` depends on `type`.

```json
{
  "id": "evt_9aF2b3c4d5E6f7G8h9I0jK1l",
  "type": "card.activated",
  "created": 1759134615,
  "data": { }
}
```

| Event type                  | When                                                                  |
| --------------------------- | --------------------------------------------------------------------- |
| `card.created`              | `card/create` succeeded. The card is usually still `PENDING`; for a product that is `ACTIVE` at once, `card.activated` follows immediately. |
| `card.activated`            | The card is `ACTIVE` and ready to use; `last4` is now known.          |
| `card.frozen`               | A freeze completed                                                    |
| `card.unfrozen`             | An unfreeze completed                                                 |
| `card.cancelled`            | A cancellation settled and the balance was returned                   |
| `card.withdrawal.completed` | A withdrawal reached your funding account                             |
| `card.withdrawal.failed`    | A withdrawal was declined and its hold released                       |
| `transaction.created`       | A new card transaction appeared (usually `PENDING`)                   |
| `transaction.updated`       | A card transaction changed status                                     |

Transaction events are sent for cards whose `operations` holds all five changes. For other cards, read their payments with `card/transactions`.
| `deposit.credited`          | An on-chain deposit was credited to your funding account              |

New event types may be added — acknowledge and ignore any you don't handle.

***

## Card events

`card.created`, `card.activated`, `card.frozen`, `card.unfrozen`:

```json
{
  "id": "evt_…", "type": "card.activated", "created": 1759134615,
  "data": { "cardId": "2026092917803181873402", "binId": "MC-WSB-512631", "status": "ACTIVE", "last4": "6654" }
}
```

`last4` is `null` on `card.created` while the card is still provisioning.

`card.cancelled`:

```json
{
  "id": "evt_…", "type": "card.cancelled", "created": 1761726615,
  "data": { "cardId": "2026092917803181873402", "status": "CANCELLED", "returnedAmount": "126.5", "currency": "USD" }
}
```

***

## Withdrawal events

Both carry the [withdrawal object](cards.md#withdrawal-status) — the same one `card/withdraw/status` returns.

```json
{
  "id": "evt_…", "type": "card.withdrawal.completed", "created": 1759137300,
  "data": {
    "withdrawalId": "3f2b8c1e-5d4a-4f9b-9c1e-7a6d5b4c3e21",
    "cardId": "2026092917803181873402",
    "amount": "20",
    "status": "COMPLETED",
    "receivedAmount": "20",
    "fee": "0",
    "failureReason": null,
    "createdAt": "2026-09-29T09:12:40Z",
    "completedAt": "2026-09-29T09:15:02Z"
  }
}
```

`card.withdrawal.failed` has `status: "FAILED"`, `receivedAmount: null` and a `failureReason`.

***

## Transaction events

`transaction.created` and `transaction.updated` carry a full [transaction object](transactions.md#the-transaction-object). `transaction.updated` adds `previousStatus`.

```json
{
  "id": "evt_…", "type": "transaction.updated", "created": 1759140903,
  "data": {
    "transactionId": "7392018475610293847",
    "cardId": "2026092917803181873402",
    "type": "PURCHASE",
    "status": "COMPLETED",
    "previousStatus": "PENDING",
    "amount": "11.62", "currency": "EUR",
    "billAmount": "12.5", "billCurrency": "USD",
    "fee": "0.19",
    "merchant": { "name": "GOOGLE ADS", "country": "IE", "mcc": "7311" },
    "description": "GOOGLE*ADS1234",
    "transactionTime": "2026-09-29T10:15:03Z"
  }
}
```

***

## Deposit events

```json
{
  "id": "evt_…", "type": "deposit.credited", "created": 1759133402,
  "data": { "amount": "500", "currency": "USD", "asset": "USDT", "network": "TRX", "txHash": "4f3c…9a1e" }
}
```
