# Authentication & Security

The gateway has three distinct trust boundaries. Understanding which one applies to each call is the most important thing to get right before you ship.

<div class="zi-trust-grid" markdown>

<div class="zi-trust-card" markdown>
<div class="zi-trust-card__num">1</div>

**Server-to-Server**

Your backend → ZiCharge Gateway

`merchant_mobile_no` + `store_password` in body, or `Authorization: Bearer <api_key>` header. Optionally hardened with a source IP allow-list.
</div>

<div class="zi-trust-card" markdown>
<div class="zi-trust-card__num">2</div>

**Customer-Facing**

Customer browser → ZiCharge Hosted Page

`payment_token` in the redirect URL. Customer authenticates with their own wallet mobile + PIN.
</div>

<div class="zi-trust-card" markdown>
<div class="zi-trust-card__num">3</div>

**Asynchronous Callback**

ZiCharge Gateway → Your IPN Listener

Form-POST to your `ipn_url`. Authenticated with HMAC-SHA256 signature (recommended).
</div>

</div>

---

## Merchant credentials

### Store credentials

Two values provisioned at onboarding through the ZiCharge merchant dashboard:

<div class="zi-info-row" markdown>

<div class="zi-info-box" markdown>
**`merchant_mobile_no`**

Format: `+964XXXXXXXXXX`

Public identifier of your merchant wallet. Sent on every server-to-server call (when not using Bearer auth).
</div>

<div class="zi-info-box" markdown>
**`store_password`**

Format: opaque string

Shared secret bound to your store. **Server-side only — never expose this client-side.**
</div>

</div>

!!! danger "store_password must never leave your server"
    Never embed it in mobile binaries, browser-side JavaScript, or include it in any URL. All credentialed calls are server-to-server only. Credentials are validated as a **pair** — the gateway will not tell you which one failed (prevents account enumeration).

### API key (Bearer token)

For Cashback requests, you can generate an API key via `POST /api/v3/merchant/generate-api-key` and pass it as:

```
Authorization: Bearer <api_key>
```

When this header is present, `merchant_mobile_no` and `store_password` are **not required** in the request body. This is the recommended approach — it keeps secrets out of every request body and enables clean key rotation.

!!! danger "Treat the API key like store_password"
    Store in an environment variable or secrets manager. Never log it, never commit it to source control, never expose it client-side. Regenerating immediately invalidates the previous key.

---

## Endpoint credential requirements

| Endpoint | `merchant_mobile_no` | `store_password` | Bearer `api_key` |
|----------|:--------------------:|:----------------:|:----------------:|
| `POST /merchant/generate-payment-token` | Required | Required | — |
| `POST /merchant/generate-qr-token` | Required | Required | — |
| `POST /merchant/payment/validation` | Required | Required | — |
| `POST /api/v3/merchant/generate-api-key` | Required | Required | — |
| `POST /api/v3/merchant/cash-back` | Conditional | Conditional | Preferred — replaces body credentials |

---

## Network controls

Both controls are configured from the merchant dashboard and are **strongly recommended for production**.

<div class="zi-env-grid" markdown>

<div class="zi-env-card zi-env-card--sandbox" markdown>
### Source IP Allow-List

Register your server's egress IP range in the dashboard. The gateway rejects any server-to-server call arriving from an IP outside your registered list.

Ideal if your platform issues outbound calls from a stable egress range.
</div>

<div class="zi-env-card zi-env-card--prod" markdown>
### TLS Enforcement

HTTPS is the only accepted scheme — there is no HTTP fallback or downgrade listener. TLS 1.2 minimum.

This is enforced at the load balancer layer and is not configurable.
</div>

</div>

---

## IPN signature verification

When payload signatures are enabled on your merchant config, every IPN includes one header that proves the callback originated from ZiCharge:

| Header | Description |
|--------|-------------|
| `X-ZiCharge-Signature` | `t=<epoch>,v1=<hex-hmac>` — Stripe-style compound value. `t` is the Unix dispatch timestamp; `v1` is the HMAC-SHA256 hex digest over `t + "." + raw_body`. |

The signed payload is:

```
signed_payload = t + "." + raw_body
expected       = HMAC-SHA256(webhook_secret, signed_payload)
```

Reject any IPN where `|now - t| > 300` seconds.

=== "Java"

    ```java
    public boolean verifyIpn(String rawBody, String signatureHeader, String webhookSecret)
            throws Exception {
        // 1 — Parse header: t=<epoch>,v1=<hmac>
        String timestamp    = signatureHeader.replaceFirst(".*t=(\\d+).*", "$1");
        String receivedHmac = signatureHeader.replaceFirst(".*v1=([a-f0-9]+).*", "$1");

        // 2 — Timestamp drift check
        long ts = Long.parseLong(timestamp);
        if (Math.abs(System.currentTimeMillis() / 1000L - ts) > 300) {
            return false;
        }

        // 3 — Build signed payload and compute HMAC
        String signedPayload = timestamp + "." + rawBody;
        Mac mac = Mac.getInstance("HmacSHA256");
        mac.init(new SecretKeySpec(webhookSecret.getBytes("UTF-8"), "HmacSHA256"));
        StringBuilder sb = new StringBuilder();
        for (byte b : mac.doFinal(signedPayload.getBytes("UTF-8"))) {
            sb.append(String.format("%02x", b));
        }

        // 4 — Constant-time comparison (prevents timing attacks)
        return MessageDigest.isEqual(sb.toString().getBytes(), receivedHmac.getBytes());
    }
    ```

=== "PHP"

    ```php
    function verifyIpn(string $rawBody, string $sigHeader, string $webhookSecret): bool {
        // 1 — Parse header: t=<epoch>,v1=<hmac>
        preg_match('/t=(\d+),v1=([a-f0-9]+)/', $sigHeader, $m);
        $timestamp    = $m[1];
        $receivedHmac = $m[2];

        // 2 — Timestamp drift check
        if (abs(time() - (int) $timestamp) > 300) {
            return false;
        }

        // 3 — Verify signature
        $signedPayload = $timestamp . '.' . $rawBody;
        $expected = hash_hmac('sha256', $signedPayload, $webhookSecret);
        return hash_equals($expected, $receivedHmac);
    }
    ```

=== "JavaScript"

    ```javascript
    const crypto = require('crypto');

    function verifyIpn(rawBody, sigHeader, webhookSecret) {
        // 1 — Parse header: t=<epoch>,v1=<hmac>
        const match = (sigHeader || '').match(/t=(\d+),v1=([a-f0-9]+)/);
        if (!match) return false;
        const [, timestamp, receivedHmac] = match;

        // 2 — Timestamp drift check
        if (Math.abs(Date.now() / 1000 - parseInt(timestamp, 10)) > 300) {
            return false;
        }

        // 3 — Verify signature (timingSafeEqual prevents timing attacks)
        const signedPayload = `${timestamp}.${rawBody}`;
        const expected = crypto
            .createHmac('sha256', webhookSecret)
            .update(signedPayload)
            .digest('hex');
        return crypto.timingSafeEqual(
            Buffer.from(expected, 'utf8'),
            Buffer.from(receivedHmac, 'utf8')
        );
    }
    ```

!!! warning "Always verify the raw body bytes"
    Build `signed_payload` from the **raw request bytes** before any parsing. Re-encoding parsed form fields changes byte order and the signature will not match.

!!! tip "Zero-downtime secret rotation"
    The gateway supports **two active `webhook_secret` values** simultaneously. Provision the new secret, deploy your listener to accept both, then retire the old one — no IPNs are dropped during rotation.

---

## Idempotency

| Operation | Anchor | Duplicate behaviour |
|-----------|--------|---------------------|
| Token mint | `(merchant_mobile_no, order_id)` | Returns the existing token while still valid — no duplicate created |
| Direct payment | `payment_token` state machine | Second attempt returns `409 token_already_used` |
| IPN delivery | `(order_id, status)` pair | Deduplicate on your side — gateway may re-deliver on retry |
| Cash-back | `reference_id` | Same `reference_id` returns the same result |

---

## Replay protection

| Channel | Mechanism |
|---------|-----------|
| Server-to-server calls | Idempotency keys + 15-minute token validity window bound replay risk |
| IPN callbacks | `X-ZiCharge-Signature` (`t=<epoch>,v1=<hmac>`) — reject any IPN where `\|now - t\| > 300s` |

---

## Sensitive data rules

!!! danger "Rules that must never be broken"
    | What | Rule |
    |------|------|
    | `store_password` | Never log it, never put it in a URL, never send it to a browser |
    | `payment_token` | Never log it, never put it in a URL parameter |
    | Customer password | Never persist it — not even temporarily |
    | All of the above | Transmission is POST body only, over HTTPS, server-to-server |

The gateway redacts sensitive fields in its own structured logs as a defence-in-depth measure. That is not a substitute for your own discipline.

---

## Internal endpoints

A small set of `/internal/merchant/*` endpoints exist for scheduler use — IPN retry dispatch and reconciliation jobs. They are:

- **Not part of the public surface** — do not call them from your integration
- Protected by a separate `X-Internal-Auth` secret compared in constant time
- Restricted at the load balancer / ingress layer independent of the secret

They are noted here only so you do not waste time investigating them in network traces.
