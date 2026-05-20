# Testing & Go-Live

<div class="zi-info-row" markdown>

<div class="zi-info-box" markdown>
**Sandbox URL**

`https://dev.zicharge.com`

Identical wire contract to production. No real funds movement. Test wallets provisioned on request.
</div>

<div class="zi-info-box" markdown>
**Production URL**

`https://secure.zicharge.com`

Live transactions. TLS 1.2+ enforced. 99.9% monthly uptime SLA.
</div>

<div class="zi-info-box" markdown>
**Go-Live Gate**

Complete the full sandbox test plan and go-live checklist before requesting production credentials.
</div>

</div>

---

## Sandbox environment

<div class="zi-env-grid" markdown>

<div class="zi-env-card zi-env-card--sandbox" markdown>
### What sandbox gives you

- No real funds — all settlement is simulated
- Same endpoint paths, request shapes, and IPN payload as production
- Persistent data (retained but not warranted)
- Test wallets provisioned on request from `integrations@zicharge.com`

**Do not load-test the sandbox environment.**
</div>

<div class="zi-env-card zi-env-card--prod" markdown>
### What production requires

- Credentials (`merchant_mobile_no` + `store_password`) stored in a **secrets manager**
- Production `ipn_url` — HTTPS, publicly reachable, behind your ingress
- Source IP allow-list configured for your egress range *(recommended)*
- IPN signature verification enabled *(recommended)*
- Reconciliation job for timed-out orders

Never go live without completing the full test plan below.
</div>

</div>

---

## Test plan checklist

Run this entire list against sandbox before requesting production credentials.

### Functional flows

- [x] Mint a token with the canonical JSON request — verify success (`code 200`, token returned)
- [x] Mint a token with `application/x-www-form-urlencoded` — verify both content types work
- [x] Customer pays on the hosted page — verify IPN arrives with `status=Success` and `fetch-payment-status` returns `paid: true`
- [x] Customer cancels on the hosted page — verify cancel IPN arrives with `status=Cancelled`
- [x] Let a token expire (15 min) — verify no IPN fires while unopened; opening after expiry triggers a system cancel + cancel IPN
- [x] Direct payment with a valid token — verify success envelope and IPN
- [x] Direct payment with an already-used token — verify `code 409`
- [x] Direct payment with an expired token — verify `code 410`
- [x] `fetch-payment-status` for an unpaid order — verify `paid: false` and `code 404` in body
- [x] `fetch-payment-status` for a paid order — verify `paid: true` and `code 200`
- [x] `validate-payment` for a paid order — verify the full detail payload
- [x] Cash-back to a known wallet — verify success
- [x] Cash-back retry with the same `reference_id` — verify idempotency (same result returned)
- [x] List transactions and page through more than one page

### Negative cases

- [x] Wrong `store_password` — `code 401`, generic message
- [x] Malformed `merchant_mobile_no` — `code 422`
- [x] `bill_amount` < 250 — `code 422`
- [x] Missing required field — `code 422`, message names the field
- [x] Direct payment with an invalid customer password — `code 422`, unified message

### IPN listener

- [x] Listener returns `2xx` **only after** persistence to your database
- [x] Listener handles repeated delivery of the same `X-ZiCharge-Request-Id` without double-processing
- [x] Signature mismatch is rejected *(if signatures enabled)*
- [x] Valid signature is accepted *(if signatures enabled)*
- [x] Timestamp drift > 5 minutes is rejected
- [x] Listener survives a brief outage and accepts the eventual retry delivery

### Resilience

- [x] Inject a `5xx` from your listener — verify the gateway retries within the documented window
- [x] After listener recovery, verify the retry succeeds and no duplicate is stored

### Reconciliation

- [x] Disable your IPN listener entirely, complete a payment, then use `fetch-payment-status` to recover the outcome — verify your reconciliation path works without IPN

!!! warning "All boxes must be green before go-live"
    Do not request production credentials until every item in the list above has been verified on sandbox.

---

## Go-live checklist

<div class="zi-env-grid" markdown>

<div class="zi-env-card zi-env-card--sandbox" markdown>
### Credentials & secrets

- [x] Production `merchant_mobile_no` and `store_password` provisioned and stored in your **secrets manager** — never in source control
- [x] `ipn_secret` stored in secrets manager *(if IPN signatures enabled)*
- [x] No credentials in environment files committed to version control
</div>

<div class="zi-env-card zi-env-card--prod" markdown>
### Network & security

- [x] Production `ipn_url` configured — HTTPS, publicly reachable
- [x] IPN signature verification enabled on merchant dashboard *(recommended)*
- [x] Source IP allow-list configured; egress IPs registered with integrations support *(recommended)*
- [x] TLS certificate on your IPN listener is valid and not expiring within 30 days
</div>

</div>

<div class="zi-env-grid" markdown>

<div class="zi-env-card zi-env-card--sandbox" markdown>
### Observability

- [x] `X-ZiCharge-Request-Id` logged for every outbound gateway call and every inbound IPN
- [x] `order_id` correlated in your logs alongside the ZiCharge request ID
- [x] Non-`2xx` rate alert configured on your IPN listener
- [x] "No IPN within N minutes after token mint" alert on your order pipeline
</div>

<div class="zi-env-card zi-env-card--prod" markdown>
### Operations

- [x] Reconciliation cron job configured for orders that time out without an IPN
- [x] On-call runbook documents: how to re-check a payment via `fetch-payment-status`, how to trigger a manual reconciliation sweep
- [x] Maintenance window notification channel subscribed
</div>

</div>

---

## Operational expectations

| | |
|---|---|
| **Uptime target** | 99.9% monthly on the production gateway |
| **Maintenance windows** | Announced in advance through the integrations channel |
| **Deploys** | Zero-downtime. Deployments are rolling and non-disruptive. Idempotent read endpoints are safe to retry during any maintenance window. |
| **P99 latency targets** | See [Payment Flow — Timing Reference](03-payment-flow.md) |

---

## Credential rotation

| Credential | Rotation mechanism | Zero-downtime? |
|------------|-------------------|:--------------:|
| `store_password` | Rotate from merchant dashboard | Yes — brief overlap window supported |
| `ipn_secret` | Rotate from merchant dashboard | Yes — two active secrets supported simultaneously |
| Customer credentials | Customers rotate their own from the ZiCharge app | — |

!!! tip "Zero-downtime `ipn_secret` rotation"
    The gateway supports **two active `ipn_secret` values** simultaneously. Provision the new secret, deploy your listener to accept both signatures, then retire the old one — no IPNs are dropped.

---

## Support

!!! info "Getting help"
    **Integration questions:** `integrations@zicharge.com`

    **Production incidents:** follow the escalation path in your merchant agreement.

    **Always include in every support request:**

    - `X-ZiCharge-Request-Id` header value
    - `order_id`
    - Approximate timestamp (UTC)
    - Environment: **sandbox** or **production**
