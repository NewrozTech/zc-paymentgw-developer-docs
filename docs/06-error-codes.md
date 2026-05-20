# Error Codes

Every API endpoint returns **HTTP 200**. The logical outcome is always in the body `code` field. This catalogue covers every value you should handle in production.

<div class="zi-info-row" markdown>

<div class="zi-info-box" markdown>
**Always HTTP 200**

Transport layer always returns 200. Inspect the body `code` field for the actual outcome — including failures.
</div>

<div class="zi-info-box" markdown>
**Match on `code`, not messages**

Message strings are localized and may change. Build all client branching logic against the numeric `code` value.
</div>

<div class="zi-info-box" markdown>
**`messages[]` is always an array**

Even single-message responses wrap in an array. On `422`, each failed field gets its own entry.
</div>

</div>

---

## Success

| `code` | Meaning | Action |
|--------|---------|--------|
| `200` | Request fulfilled. `data` contains the endpoint payload. | Proceed. |

---

## Client errors

| `code` | Meaning | Endpoints | Recommended action |
|--------|---------|-----------|-------------------|
| `400` | Malformed request body or unsupported content type | All | Fix the request. Do not retry as-is. |
| `401` | Invalid merchant credentials | All credentialed endpoints | Verify `merchant_mobile_no` + `store_password`. Do not auto-retry — exponential lockout protects accounts. |
| `404` | Resource not found | `fetch-payment-status`, `fetch-payment-token-data` | Treat as "not yet paid / unknown token". Retry per your reconciliation policy. |
| `409` | Conflict — state machine refused the transition | Token mint (duplicate order), direct-payment (token already used/cancelled) | Do not retry. Inspect with `fetch-payment-token-data`. |
| `410` | Gone — token expired | Direct payment, hosted page | Mint a new token. |
| `422` | Validation failure or business-rule violation | All endpoints | Read `messages[]` — one entry per failed rule. Fix and retry. |
| `429` | Rate-limited | Any credentialed endpoint | Back off exponentially. Contact support if you legitimately exceed limits. |

!!! warning "401 lockout behaviour"
    Repeated `401` responses trigger an exponential account lockout on the gateway side. Do not implement auto-retry on `401` — fix the credentials first.

---

## Server errors

| `code` | Meaning | Recommended action |
|--------|---------|-------------------|
| `500` | Unexpected gateway error | Retry **idempotent reads** (`fetch-payment-status`, `validate-payment`) with backoff. For writes, verify with `fetch-payment-status` before retrying. Include `X-ZiCharge-Request-Id` in support tickets. |
| `502` / `503` / `504` | Transient infrastructure failure | Retry with exponential backoff (1 s → 2 s → 4 s, cap at 30 s). Expected only during deploys. |

---

## Common `messages` strings

Messages are localized. The strings below are the canonical English forms.

!!! note "Match on `code`, not message text"
    The gateway returns **generic messages** for credential failures to prevent account enumeration. Never branch on message strings in production — only the `code` field is stable.

| Message | Trigger |
|---------|---------|
| `"Sorry! Invalid Store Credentials."` | Merchant credential pair did not match. Generic to prevent enumeration. |
| `"Sorry! Invalid Order ID."` | Credentials valid but order not found. |
| `"Mobile No. & Password combination do not match."` | Direct-payment customer credentials invalid or account blocked. |
| `"Congratulation! Successfully Deposited."` | Payment confirmed by `fetch-payment-status`. |
| `"Awaiting payment for order id #<order_id>"` | `fetch-payment-status` — no completed match found. |
| `"The merchant mobile no format is invalid."` | `merchant_mobile_no` failed the `+964XXXXXXXXXX` regex. |
| `"The customer mobile no format is invalid."` | `customer_mobile_no` failed the `+964XXXXXXXXXX` regex. |
| `"Bill amount must be at least 250."` | Sub-minimum `bill_amount` at token generation. |
| `"This payment link has expired."` | Hosted page accessed after the 15-minute validity window. |
| `"Insufficient balance."` | Customer wallet has insufficient funds. |
| `"Daily limit exceeded."` / `"Monthly limit exceeded."` | Customer or merchant transaction cap reached. |
| `"Unable to process service!"` | Generic fallback for unexpected server errors. |

---

## Validation envelope

When a request fails field validation, the response contains **one entry per failed field**:

```json
{
  "code": 422,
  "messages": [
    "The merchant mobile no format is invalid.",
    "The order id must not be blank."
  ],
  "data": null
}
```

!!! tip "Stable message strings"
    Message wording is stable — any existing client-side string matching on these values will continue to work.

---

## Retry guide

| Operation | Idempotent? | Safe to retry on 5xx? | Notes |
|-----------|:-----------:|:---------------------:|-------|
| `generate-payment-token` | Required | Required | Same `order_id` returns the existing token |
| `validate-payment` | Yes (read-only) | Yes | |
| `fetch-payment-status` | Yes (read-only) | Yes | |
| `fetch-payment-token-data` | Yes (read-only) | Yes | |
| `payment/direct` | Required | Required | Verify with `fetch-payment-token-data` first |
| `cash-back` | Required | Required | Same `reference_id` returns same result |
| `transactions` list | Yes (read-only) | Yes | |

!!! tip "Always use exponential backoff with jitter"
    On any retriable error: 1 s → 2 s → 4 s → 8 s — cap at 30 s. Never retry in a tight loop. Add random jitter (±20 %) to prevent thundering-herd during a recovery window.
