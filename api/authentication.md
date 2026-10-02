# Authentication

Every request carries a secret API key in the `X-API-KEY` header. There is no request signing, no timestamp and no nonce — just the key, over HTTPS.

***

## Create a key

You create and manage keys yourself, in the dashboard:

1. Open **Settings → API keys** and choose **Create key**.
2. Fill in:
   * **Name** — a label for your own records, e.g. `billing-service-prod`.
   * **IP allowlist** *(optional)* — the addresses allowed to use this key. See [IP allowlist](#ip-allowlist).
   * **Expiry** *(optional)* — the key stops working after this date.
3. Confirm with your **card passcode**, or a code we send you by **email** or **Telegram**.
4. **Copy the key now.** It is shown exactly once. StableMesh stores only a hash of it and cannot show it to you again.

| Environment | Key prefix  | Example                          |
| ----------- | ----------- | -------------------------------- |
| Production  | `smk_live_` | `smk_live_7Hq2kLmN4pRtV8xZ…`     |
| Sandbox     | `smk_test_` | `smk_test_Ab3dEf6gHi9jKl2mNo…`   |

You can hold up to **5 active keys** per account.

***

## Send the key

```bash
curl -X POST https://api.stablemesh.io/v1/account/balance \
  -H "X-API-KEY: smk_live_7Hq2kLmN4pRtV8xZ…" \
  -H "Content-Type: application/json" \
  -d '{}'
```

***

## IP allowlist

Each key can be restricted to the addresses your servers call from. A request from any other address is refused with **HTTP 403, code `11003`** — even if the key itself is valid.

| Entry form               | Example         | Matches                     |
| ------------------------ | --------------- | --------------------------- |
| Exact IPv4 address       | `203.0.113.10`  | that address only           |
| Exact IPv6 address       | `2001:db8::1`   | that address only           |
| IPv4 prefix wildcard     | `203.0.113.*`   | `203.0.113.0`–`203.0.113.255` |

* Up to **20 entries** per key. CIDR notation (`203.0.113.0/24`) is not accepted — use the wildcard form or list addresses.
* An **empty list allows any address**. We strongly recommend an allowlist on every production key.
* You can edit a key's allowlist at any time without rotating the key. Changes apply to the next request.
* The address we check is your server's **public egress IP** — the one the internet sees. Behind NAT, a load balancer or an outbound proxy, allow that address, not the server's private one.

***

## Manage and revoke keys

**Settings → API keys** lists every key with its name, prefix, creation date, **last used** time, allowlist and expiry.

**Revoke** takes effect immediately: the next request with that key gets `401`. Revocation cannot be undone — create a new key instead.

## Rotate a key without downtime

1. Create a new key.
2. Deploy it to your servers.
3. Check in **Settings → API keys** that the new key shows a recent **last used** time and the old one has stopped.
4. Revoke the old key.

***

## Errors

| HTTP  | `code`  | When                                                                    |
| ----- | ------- | ----------------------------------------------------------------------- |
| `401` | `11002` | The `X-API-KEY` header is missing; the key is unknown, revoked or expired; or the account it belongs to is not active |
| `403` | `11003` | The key is valid, but the request came from an address not on its allowlist |
| `429` | `11004` | More than 20 requests per second on this key — back off and retry       |

```json
{ "code": 11002, "message": "Invalid or revoked API key" }
```

***

## Keep your keys safe

{% hint style="warning" %}
**A key has full access to your account through the API.** Anyone holding it can issue cards and move money between your funding account and your cards. It cannot withdraw funds to an external address and cannot reveal card numbers — but treat it like a password.

* Use keys **from your servers only**. Never put a key in a mobile app, a browser, or source control.
* Keep keys in a secret manager and restrict production keys with an **IP allowlist**.
* If a key may have been exposed, **revoke it immediately** in Settings → API keys.
{% endhint %}
