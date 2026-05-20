# Payment Flow

Read this before the API reference — the endpoints will make much more sense as a glossary once you understand the flow end-to-end.

<div class="zi-info-row" markdown>
<div class="zi-info-box" markdown>
**Token window**

15 minutes from mint. After expiry, the token cannot be paid and will auto-cancel on next open.
</div>
<div class="zi-info-box" markdown>
**IPN retries**

Up to 2 attempts. If the first attempt fails, a retry is sent after 5 minutes.
</div>
<div class="zi-info-box" markdown>
**Confirmation**

Use both the IPN and a synchronous status check before fulfilling — they are both authoritative.
</div>
</div>

---

## Happy path — hosted page

```mermaid
sequenceDiagram
    autonumber
    participant C as Customer
    participant B as Your Backend
    participant G as ZiCharge Gateway
    participant I as Your IPN Listener

    C->>B: Checkout / add to cart
    B->>G: POST /merchant/generate-payment-token
    G-->>B: { code: 200, token: "tkn_..." }
    B->>C: HTTP 302 → ZiCharge hosted page
    C->>G: GET /merchant/payment?token=tkn_...
    G-->>C: Hosted payment page (mobile + PIN)
    C->>G: Submit credentials
    G->>G: Authenticate · debit · credit · mark USED
    G-->>C: Success screen + redirect to success_url
    G-)I: POST ipn_url (async, form-encoded, status=Success)
    I-->>G: HTTP 200 (after persistence)
    B->>G: POST /merchant/fetch-payment-status
    G-->>B: { code: 200, paid: true }
    B->>B: Fulfil order
```

!!! info "Two confirmation channels — both intentional"
    The **IPN callback** is your primary signal. The synchronous **`fetch-payment-status`** call is your fallback when the IPN is delayed or your listener was briefly down. Either one is authoritative on its own — using both makes your integration bulletproof.

---

## Customer cancels or token expires

```mermaid
sequenceDiagram
    autonumber
    participant C as Customer
    participant G as ZiCharge Gateway
    participant I as Your IPN Listener

    C->>G: Click Cancel on hosted page (or 15-min timer elapses)
    G->>G: Mark token CANCELLED (first-write-wins lock)
    G-->>C: Payment cancelled → redirect to cancel_url
    G-)I: POST ipn_url (async, status=Cancelled)
    I-->>G: HTTP 200 (after persistence)
```

!!! warning "Release the cart on a Cancelled IPN"
    When you receive `status=Cancelled`, release the hold on the customer's cart immediately. If your listener was down during the retry window, poll `fetch-payment-status` after your order timeout to recover.

---

## Token state machine

```mermaid
stateDiagram-v2
    direction LR

    [*] --> PENDING : Token minted\n(15-min window starts)
    PENDING --> USED : Customer pays\non hosted page
    PENDING --> CANCELLED : Customer clicks Cancel
    PENDING --> CANCELLED : Timer expires (15 min)
    PENDING --> CANCELLED : Customer opens\nalready-expired link
    USED --> [*] : IPN dispatched — Success
    CANCELLED --> [*] : IPN dispatched — Cancelled
```

State transitions are protected by three mechanisms:

<div class="zi-trust-grid" markdown>

<div class="zi-trust-card" markdown>
<div class="zi-trust-card__num">1</div>

**Pessimistic write lock**

Token row is locked during payment processing with a 3 s wait timeout — fail-fast under contention.
</div>

<div class="zi-trust-card" markdown>
<div class="zi-trust-card__num">2</div>

**Conditional UPDATE**

Cancel runs `WHERE cancelled_at IS NULL` — first-write-wins, no silent overwrites.
</div>

<div class="zi-trust-card" markdown>
<div class="zi-trust-card__num">3</div>

**Atomic dispatch claim**

IPN sends are claimed atomically before dispatch — duplicate IPNs across overlapping retry runs are physically impossible.
</div>

</div>

---

## IPN delivery & retry policy

```mermaid
flowchart TD
    A([Terminal state reached]) --> B[Dispatch IPN\nPOST to ipn_url]
    B --> C{Listener returned\nHTTP 2xx?}
    C -- Yes --> D([Delivered\nNo more retries])
    C -- No / timeout --> E{Attempt\nnumber < 2?}
    E -- Yes --> F[Wait 5 minutes]
    F --> B
    E -- No --> G([Max retries reached\nRecover via reconciliation API])
```

| Attempt | Timing |
|---------|--------|
| 1 | Immediately after database commit |
| 2 | Retry sent 5 minutes after the first failed attempt |
| Max | 2 attempts per terminal event |

!!! danger "Acknowledge only after you persist"
    Return HTTP **2xx only after** writing the IPN to your database. A 2xx permanently stops all retries. If your database is unavailable, return **5xx** and we will retry.

---

## Direct payment flow

For merchants running their own checkout surface — mobile app or in-store POS:

```mermaid
sequenceDiagram
    autonumber
    participant B as Your Backend
    participant G as ZiCharge Gateway
    participant I as Your IPN Listener

    B->>G: POST /merchant/generate-payment-token
    G-->>B: { token: "tkn_..." }
    B->>G: POST /merchant/payment/direct
    note right of B: { mobile_no, password, payment_token }
    G->>G: Authenticate · debit · credit · mark USED
    G-->>B: { code: 200, transactionInfo: { ... } } (synchronous)
    G-)I: POST ipn_url (async, status=Success)
    I-->>G: HTTP 200
```

!!! warning "PCI-equivalent controls required"
    The direct endpoint accepts the customer's wallet password. Your channel **must** be server-to-server only — never route customer credentials through a browser or mobile client, and never persist the password. Prefer the hosted page for any web or webview context.

---

## Reconciliation

Use these read-only endpoints when the IPN is late, your listener was down, or you need to confirm before fulfilling:

<div class="zi-env-grid" markdown>

<div class="zi-env-card zi-env-card--sandbox" markdown>
### fetch-payment-status
`POST /merchant/fetch-payment-status`

Fast yes/no check. Exact-match on `(merchant_id, order_id, amount)` at the database level.

**Use when:** you know the `order_id` and `amount` and need a quick paid/unpaid answer.
</div>

<div class="zi-env-card zi-env-card--prod" markdown>
### validate-payment
`POST /merchant/validate-payment`

Full transaction detail — `transaction_id`, `customer_account_no`, `received_at`, and `status`.

**Use when:** you need the canonical ZiCharge reference for your books or a support query.
</div>

</div>

Both endpoints are read-only and safe to call as often as needed.

---

## Timing reference

| Operation | Typical | P99 budget |
|-----------|:-------:|:----------:|
| `generate-payment-token` | < 100 ms | 300 ms |
| `validate-payment` / `fetch-payment-status` | < 80 ms | 250 ms |
| `payment/direct` (synchronous portion) | < 500 ms | 1.5 s |
| Token validity window | — | 15 min from mint |
| IPN dispatch after commit | < 1 s nominal | Up to 2 retries |

!!! tip "Set explicit HTTP client timeouts"
    Configure **2 s connect** and **10 s read** timeouts on all gateway calls. These are gateway-side budgets and exclude your own backend latency.
