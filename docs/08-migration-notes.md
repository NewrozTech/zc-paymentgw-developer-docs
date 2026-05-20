# Platform Notes

This page documents current API behaviour, validation rules, and operational characteristics of the ZiCharge payment gateway.

---

## API guarantees

The following surfaces are stable and will not change without advance notice:

<div class="zi-trust-grid" markdown>

<div class="zi-trust-card" markdown>
<div class="zi-trust-card__num">1</div>

**Endpoint paths**

All `/merchant/...` endpoint paths are stable. No redirects, no renames, no deprecations without a notice period.
</div>

<div class="zi-trust-card" markdown>
<div class="zi-trust-card__num">2</div>

**Request and response shapes**

All request field names and the response envelope `{ messages, data, code }` are stable. The body `code` field is always present.
</div>

<div class="zi-trust-card" markdown>
<div class="zi-trust-card__num">3</div>

**IPN contract**

The IPN delivery method, content type, field set, and status values are stable. Existing listeners will continue to receive identical payloads.
</div>

</div>

| Surface | Status |
|---------|--------|
| Endpoint paths (`/merchant/generate-payment-token`, `/merchant/payment/validation`, `/merchant/payment/direct`, `/merchant/payment?token=...`) | Stable |
| All request field names | Stable |
| Credential model — `merchant_mobile_no` + `store_password` | Stable |
| Payment token validity — 15 minutes from creation | Stable |
| Response envelope — `{ messages, data, code }`, HTTP 200 for all outcomes | Stable |
| IPN method, content type, field set, and status values | Stable |

---

## Validation rules

### Mobile number format

`merchant_mobile_no` and `customer_mobile_no` are validated against the format `+964XXXXXXXXXX`. Numbers that do not include the `+964` prefix are rejected at the validation layer with a `422` response and a descriptive message.

**Action:** Ensure all mobile numbers are submitted in `+964XXXXXXXXXX` format.

> **Note:** `receiver_mobile_no` on the cash-back endpoint is not subject to the format regex — it is validated by wallet account existence instead.

---

### Minimum payment amount

`bill_amount` must be at least **250 IQD**. Tokens with a sub-minimum amount are rejected at the time of token generation, not at payment time.

**Action:** Ensure your minimum order value is at least 250 IQD before calling the token generation endpoint.

---

### Direct payment error responses

All credential and account failures on the direct payment endpoint return a single unified message: `"Mobile No. & Password combination do not match."` This applies to invalid credentials, locked accounts, and other authentication failures.

**Action:** Branch on the `code` field (`422`), not on the message text.

---

### Validation endpoint — order not found

When `POST /merchant/payment/validation` cannot find the order, it returns `code 422` with `"Sorry! Invalid Order ID."`.

**Action:** Check for `code == 422` when handling an order-not-found condition. Do not rely on `404` for this endpoint.

---

### Credential error messages

Credential failures return a unified message: `"Sorry! Invalid Store Credentials."` The gateway does not indicate which individual field failed, as doing so would reveal whether a given mobile number exists in the system.

**Action:** Match on the unified credential message or on `code == 401`. Do not rely on field-specific messages.

---

## New capabilities

The following features are available and opt-in. Enabling them does not affect existing integrations.

| Capability | Description | How to enable |
|------------|-------------|--------------|
| **Signed IPN callbacks** | Every IPN includes `X-ZiCharge-Signature`, `X-ZiCharge-Timestamp`, and `X-ZiCharge-Request-Id` headers for authenticity verification | Enable in the merchant dashboard; configure `ipn_secret` |
| **IPN automatic retry** | At-least-once delivery with up to 2 attempts; retry sent 5 minutes after first failure | Make your listener idempotent on `X-ZiCharge-Request-Id` |
| **Cash-back idempotency** | Supplying a `reference_id` on cash-back requests makes them replay-safe | Always include a stable `reference_id` on cash-back calls |
| **QR token generation** | `POST /merchant/generate-qr-token` returns a token plus a server-generated QR image URL | Use this endpoint for in-store POS, kiosks, and printed media |

---

## Operational characteristics

| Characteristic | Detail |
|----------------|--------|
| **Uptime target** | 99.9% monthly on the production gateway |
| **Deployments** | Zero-downtime. Idempotent endpoints are safe to retry during any maintenance window. |
| **Maintenance windows** | Announced in advance through the integrations channel |
| **Token validity** | 15 minutes from creation. Expired tokens cannot be paid and trigger a `Cancelled` IPN when accessed. |
| **IPN retries** | Up to 2 attempts per terminal event; retry sent 5 minutes after first failure |
| **P99 latency** | See [Payment Flow](03-payment-flow.md) for timing reference |
