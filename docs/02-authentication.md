# Authentication & Security

The gateway has three distinct trust boundaries. Understanding which one applies to each call is the most important thing to get right before you ship.

<div class="zi-trust-grid" markdown>

<div class="zi-trust-card" markdown>
<div class="zi-trust-card__num">1</div>

**Server-to-Server**

Your backend → ZiCharge Gateway

`merchant_mobile_no` + `store_password` over HTTPS. Optionally hardened with a source IP allow-list.
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

Two values provisioned at onboarding through the ZiCharge merchant dashboard:

<div class="zi-info-row" markdown>

<div class="zi-info-box" markdown>
**`merchant_mobile_no`**

Format: `+964XXXXXXXXXX`

Public identifier of your merchant wallet. Sent on every server-to-server call.
</div>

<div class="zi-info-box" markdown>
**`store_password`**

Format: opaque string

Shared secret bound to your store. **Server-side only — never expose this client-side.**
</div>

</div>

!!! danger "store_password must never leave your server"
    Never embed it in mobile binaries, browser-side JavaScript, or include it in any URL. All credentialed calls are server-to-server only. Credentials are validated as a **pair** — the gateway will not tell you which one failed (prevents account enumeration).

---

## Endpoint credential requirements

| Endpoint | `merchant_mobile_no` | `store_password` |
|----------|:--------------------:|:----------------:|
| `POST /merchant/generate-payment-token` | Required | Required |
| `POST /merchant/generate-qr-token` | Required | Required |
| `POST /merchant/validate-payment` | Required | Required |
| `POST /merchant/cash-back` | Required | Required |
| `POST /merchant/fetch-payment-status` | Required | Not required — order ID + amount serve as proof |
| `POST /merchant/fetch-payment-token-data` | Not required | Not required — token is the lookup key |
| `POST /merchant/payment/direct` | Not required | Not required — customer credentials, not merchant |

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

When payload signatures are enabled on your merchant config, every IPN includes three headers that prove the callback originated from ZiCharge:

| Header | Description |
|--------|-------------|
| `X-ZiCharge-Signature` | Hex HMAC-SHA256 of the **raw** form body, signed with your `ipn_secret` |
| `X-ZiCharge-Timestamp` | Unix epoch seconds at dispatch time — use to detect replay attacks |
| `X-ZiCharge-Request-Id` | Opaque request ID for deduplication and support correlation |

=== "Java"

    ```java
    import javax.crypto.Mac;
    import javax.crypto.spec.SecretKeySpec;
    import java.security.MessageDigest;
    import java.util.Map;

    public boolean verifyIpn(byte[] rawBody, Map<String, String> headers, String ipnSecret)
            throws Exception {
        // 1 — Signature check (constant-time comparison prevents timing attacks)
        Mac mac = Mac.getInstance("HmacSHA256");
        mac.init(new SecretKeySpec(ipnSecret.getBytes("UTF-8"), "HmacSHA256"));
        byte[] hmacBytes = mac.doFinal(rawBody);
        StringBuilder sb = new StringBuilder();
        for (byte b : hmacBytes) sb.append(String.format("%02x", b));
        String expected = sb.toString();
        String received = headers.get("X-ZiCharge-Signature");
        if (!MessageDigest.isEqual(expected.getBytes(), received.getBytes())) {
            return false;
        }

        // 2 — Reject stale requests (> 5 minute clock drift)
        long ts = Long.parseLong(headers.get("X-ZiCharge-Timestamp"));
        if (Math.abs(System.currentTimeMillis() / 1000L - ts) > 300) {
            return false;
        }

        return true;
    }
    ```

=== "PHP"

    ```php
    function verifyIpn(string $rawBody, array $headers, string $ipnSecret): bool {
        // 1 — Signature check
        $expected = hash_hmac('sha256', $rawBody, $ipnSecret);
        if (!hash_equals($expected, $headers['X-ZiCharge-Signature'])) {
            return false;
        }

        // 2 — Reject stale requests (> 5 minute clock drift)
        $ts = (int) $headers['X-ZiCharge-Timestamp'];
        if (abs(time() - $ts) > 300) {
            return false;
        }

        return true;
    }
    ```

=== "JavaScript"

    ```javascript
    const crypto = require('crypto');

    function verifyIpn(rawBody, headers, ipnSecret) {
        // 1 — Signature check (timingSafeEqual prevents timing attacks)
        const expected = crypto
            .createHmac('sha256', ipnSecret)
            .update(rawBody)
            .digest('hex');
        const received = Buffer.from(headers['x-zicharge-signature'] || '', 'utf8');
        if (!crypto.timingSafeEqual(Buffer.from(expected, 'utf8'), received)) {
            return false;
        }

        // 2 — Reject stale requests (> 5 minute clock drift)
        const ts = parseInt(headers['x-zicharge-timestamp'], 10);
        if (Math.abs(Date.now() / 1000 - ts) > 300) {
            return false;
        }

        return true;
    }
    ```

!!! warning "Always verify the raw body bytes"
    Compute the HMAC against the **raw request bytes** before any parsing. Re-encoding parsed form fields changes byte order and the signature will not match.

!!! tip "Zero-downtime secret rotation"
    The gateway supports **two active `ipn_secret` values** simultaneously. Provision the new secret, deploy your listener to accept both, then retire the old one — no IPNs are dropped during rotation.

---

## Idempotency

| Operation | Anchor | Duplicate behaviour |
|-----------|--------|---------------------|
| Token mint | `(merchant_mobile_no, order_id)` | Returns the existing token while still valid — no duplicate created |
| Direct payment | `payment_token` state machine | Second attempt returns `409 token_already_used` |
| IPN delivery | `X-ZiCharge-Request-Id` header | Deduplicate on your side — gateway may re-deliver on retry |
| Cash-back | `reference_id` | Same `reference_id` returns the same result |

---

## Replay protection

| Channel | Mechanism |
|---------|-----------|
| Server-to-server calls | Idempotency keys + 15-minute token validity window bound replay risk |
| IPN callbacks | `X-ZiCharge-Timestamp` + HMAC signature pair — reject any IPN older than ±5 minutes |

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
