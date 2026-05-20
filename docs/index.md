---
template: home.html
---

## Explore the docs

<div class="zi-cards" markdown>

<div class="zi-card" markdown>
:material-layers-outline: **Overview**

Supported flows, environments, base URLs, response envelope, and token lifecycle.

[Read overview →](01-overview.md)
</div>

<div class="zi-card" markdown>
:material-shield-lock-outline: **Authentication**

Merchant credentials, IP allow-listing, IPN HMAC, idempotency, and replay protection.

[Read authentication →](02-authentication.md)
</div>

<div class="zi-card" markdown>
:material-swap-horizontal: **Payment Flow**

End-to-end sequence diagrams, token state machine, and IPN retry policy.

[Read payment flow →](03-payment-flow.md)
</div>

<div class="zi-card" markdown>
:material-api: **API Reference**

Every endpoint — request schema, response schema, errors, and examples.

[Read API reference →](04-api-reference.md)
</div>

<div class="zi-card" markdown>
:material-webhook: **IPN Callbacks**

Callback payload, signature verification, delivery semantics, and retries.

[Read IPN callbacks →](05-ipn-callbacks.md)
</div>

<div class="zi-card" markdown>
:material-alert-circle-outline: **Error Codes**

Full catalogue of `code` values and recommended client handling.

[Read error codes →](06-error-codes.md)
</div>

<div class="zi-card" markdown>
:material-test-tube: **Testing & Go-Live**

Sandbox, test plan checklist, go-live checklist, and credential rotation.

[Read testing guide →](07-testing-and-go-live.md)
</div>

<div class="zi-card" markdown>
:material-information-outline: **Platform Notes**

API behaviour specifications, validation rules, and operational characteristics.

[Read platform notes →](08-migration-notes.md)
</div>

</div>

---

## Quick start

**Step 1 — Generate a payment token** from your backend

```bash
curl -X POST https://secure.zicharge.com/merchant/generate-payment-token \
  -d "merchant_mobile_no=+9647712345678" \
  -d "store_password=your-store-secret" \
  -d "order_id=ORD-2026-000123" \
  -d "bill_amount=5000" \
  -d "success_url=https://yoursite.com/orders/123/success" \
  -d "cancel_url=https://yoursite.com/orders/123/cancel"
```

**Step 2 — Redirect the customer** to the hosted payment page

```
https://secure.zicharge.com/merchant/payment?token=<token>
```

**Step 3 — Validate the payment** after receiving the IPN

```bash
curl -X POST https://secure.zicharge.com/merchant/payment/validation \
  -d "merchant_mobile_no=+9647712345678" \
  -d "store_password=your-store-secret" \
  -d "order_id=ORD-2026-000123"
```

---

## Environments

| Environment | Base URL | Purpose |
|-------------|----------|---------|
| **Sandbox** | `https://dev.zicharge.com` | Integration testing — no real funds |
| **Production** | `https://secure.zicharge.com` | Live transactions |

!!! tip "Support"
    Email **integrations@zicharge.com** — always include `X-ZiCharge-Request-Id`, `order_id`, and environment.
