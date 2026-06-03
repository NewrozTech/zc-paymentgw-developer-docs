---
template: home.html
---

## Integration Flows

<p class="zi-section-intro">Your backend mints a token or pushes a credit — ZiCharge handles the hosted checkout, wallet debit, and real-time confirmation.</p>

=== "Online Payment (Deposit)"

    <p class="zi-diagram-desc">Your backend mints a token → customer pays on hosted checkout → you receive an IPN → validate server-side to confirm.</p>

    ``` mermaid
    sequenceDiagram
        autonumber
        participant M as Your Server
        participant G as ZiCharge Gateway
        participant C as Customer

        M->>G: POST /merchant/generate-payment-token
        G-->>M: token + payment_url
        M->>C: Redirect customer to hosted checkout

        C->>G: Enter mobile number and PIN
        G-->>C: Payment confirmed

        G->>M: POST ipn_url — IPN notification
        M-->>G: HTTP 200 acknowledge

        M->>G: POST /merchant/payment/validation
        G-->>M: tx_unique_id + status confirmed
    ```

=== "Cashback (Withdrawal)"

    <p class="zi-diagram-desc">Your backend pushes a credit directly to the customer wallet in a single synchronous call — no redirect or IPN required.</p>

    ``` mermaid
    sequenceDiagram
        autonumber
        participant M as Your Server
        participant G as ZiCharge Gateway
        participant W as Customer Wallet

        M->>G: POST /api/v3/merchant/cash-back
        G->>W: Credit amount applied to wallet
        G-->>M: tx_unique_id + Success

        note over M,G: Optional — verify the transaction record
        M->>G: POST /merchant/payment/validation
        G-->>M: tx_unique_id, amount, received_at
    ```

---

## API Reference

<div class="zi-api-section" markdown>

<div class="zi-api-group" markdown>

<div class="zi-api-group-header">
  <div class="zi-api-group-icon zi-api-group-icon--deposit">↓</div>
  <div>
    <div class="zi-api-group-title">Deposit</div>
    <div class="zi-api-group-sub">Accept payments from customer wallets</div>
  </div>
</div>

<div class="zi-api-list" markdown>

<div class="zi-api-item" markdown>
<span class="zi-method zi-method--post">POST</span>
<div class="zi-api-item-body" markdown>
**[Initiate Payment](deposit/initiate-payment.md)**
Mint a payment token and redirect the customer to the hosted checkout page.
</div>
</div>

<div class="zi-api-item" markdown>
<span class="zi-method zi-method--post">POST</span>
<div class="zi-api-item-body" markdown>
**[IPN Callback](deposit/ipn-callback.md)**
Receive and verify real-time payment notifications at your listener.
</div>
</div>

<div class="zi-api-item" markdown>
<span class="zi-method zi-method--post">POST</span>
<div class="zi-api-item-body" markdown>
**[Validate Payment](deposit/validate-payment.md)**
Confirm the final payment status and obtain the canonical transaction ID.
</div>
</div>

</div>
</div>

<div class="zi-api-group" markdown>

<div class="zi-api-group-header">
  <div class="zi-api-group-icon zi-api-group-icon--withdrawal">↑</div>
  <div>
    <div class="zi-api-group-title">Withdrawal</div>
    <div class="zi-api-group-sub">Push credits to customer wallets</div>
  </div>
</div>

<div class="zi-api-list" markdown>

<div class="zi-api-item" markdown>
<span class="zi-method zi-method--post">POST</span>
<div class="zi-api-item-body" markdown>
**[Generate API Key](withdrawal/generate-api-key.md)**
Issue a merchant API key for Bearer token authentication on Cashback requests.
</div>
</div>

<div class="zi-api-item" markdown>
<span class="zi-method zi-method--post">POST</span>
<div class="zi-api-item-body" markdown>
**[Cashback](withdrawal/cashback.md)**
Credit a customer wallet synchronously — for refunds, rewards, and promotions.
</div>
</div>

<div class="zi-api-item" markdown>
<span class="zi-method zi-method--post">POST</span>
<div class="zi-api-item-body" markdown>
**[Validate Payment](withdrawal/validate-payment.md)**
Fetch the full transaction record for a completed cashback.
</div>
</div>

</div>
</div>

</div>

---

## API Explorer

<div class="zi-explorer-cta" markdown>

<div class="zi-explorer-cta-text" markdown>
### Interactive API Explorer

Browse every endpoint, inspect request and response schemas, and run live calls against the sandbox — directly in your browser. No setup required.

<div class="zi-explorer-cta-actions" markdown>
<a href="api-explorer/" class="zi-btn zi-btn--primary zi-btn--lg">Open API Explorer</a>
<a href="assets/postman/ZiCharge-Merchant-API.postman_collection.json" download class="zi-btn zi-btn--outline zi-btn--lg">Download Postman Collection</a>
</div>

<div class="zi-explorer-meta" markdown>
Covers 4 live endpoints · Sandbox & Production servers · Inline request builder · Example responses
</div>
</div>

<div class="zi-explorer-cta-preview" markdown>
<div class="zi-mock-explorer" markdown>
<div class="zi-mock-explorer-bar">
  <span class="zi-mock-dot zi-mock-dot--red"></span>
  <span class="zi-mock-dot zi-mock-dot--yellow"></span>
  <span class="zi-mock-dot zi-mock-dot--green"></span>
  <span class="zi-mock-url">developer.zicharge.com/api-explorer</span>
</div>
<div class="zi-mock-explorer-body">
  <div class="zi-mock-row"><span class="zi-mock-tag zi-mock-tag--post">POST</span><span class="zi-mock-path">/merchant/generate-payment-token</span></div>
  <div class="zi-mock-row"><span class="zi-mock-tag zi-mock-tag--post">POST</span><span class="zi-mock-path">/merchant/payment/validation</span></div>
  <div class="zi-mock-row"><span class="zi-mock-tag zi-mock-tag--post">POST</span><span class="zi-mock-path">/api/v3/merchant/generate-api-key</span></div>
  <div class="zi-mock-row"><span class="zi-mock-tag zi-mock-tag--post">POST</span><span class="zi-mock-path">/api/v3/merchant/cash-back</span></div>
</div>
</div>
</div>

</div>

---

## Environments

<div class="zi-env-grid" markdown>

<div class="zi-env-card zi-env-card--sandbox" markdown>
### Sandbox
`https://dev.zicharge.com`

Integration testing environment. No real funds move. Identical wire contract to production.

- Test wallets provisioned on request
- Data is persistent but not warranted
- Do not load-test this environment
</div>

<div class="zi-env-card zi-env-card--prod" markdown>
### Production
`https://secure.zicharge.com`

Live transaction environment. TLS 1.2+ enforced.

- Real funds movement
- 99.9% monthly uptime SLA
- Zero-downtime deployments
</div>

</div>

---

<div class="zi-merchant-section" id="become-a-merchant" markdown>

## Become a Merchant

Interested in integrating ZiCharge payments into your platform? Fill in your details and our integrations team will be in touch.

<form id="merchant-form" class="zi-merchant-form" novalidate>
  <div class="zi-form-row">
    <div class="zi-form-group">
      <label class="zi-form-label" for="business-name">Business Name <span class="zi-required">*</span></label>
      <input class="zi-form-input" type="text" id="business-name" placeholder="e.g. Al-Rashid Electronics" required>
    </div>
    <div class="zi-form-group">
      <label class="zi-form-label" for="website-url">Website URL <span class="zi-required">*</span></label>
      <input class="zi-form-input" type="url" id="website-url" placeholder="https://yoursite.com" required>
    </div>
  </div>
  <div class="zi-form-row">
    <div class="zi-form-group">
      <label class="zi-form-label" for="currencies">Available Currencies <span class="zi-required">*</span></label>
      <input class="zi-form-input" type="text" id="currencies" placeholder="e.g. IQD, USD" required>
    </div>
    <div class="zi-form-group">
      <label class="zi-form-label" for="contact-name">Contact Person Name <span class="zi-required">*</span></label>
      <input class="zi-form-input" type="text" id="contact-name" placeholder="Full name" required>
    </div>
  </div>
  <div class="zi-form-row zi-form-row--full">
    <div class="zi-form-group">
      <label class="zi-form-label" for="contact-email">Contact Email <span class="zi-required">*</span></label>
      <input class="zi-form-input" type="email" id="contact-email" placeholder="you@yoursite.com" required>
    </div>
  </div>
  <div class="zi-form-actions">
    <button type="submit" class="zi-btn zi-btn--primary zi-btn--lg">Submit Enquiry</button>
    <p class="zi-form-note">Our integrations team typically responds within 1–2 business days.</p>
  </div>
</form>

</div>

<script>
document.getElementById('merchant-form').addEventListener('submit', function(e) {
  e.preventDefault();
  var bn = document.getElementById('business-name').value.trim();
  var wu = document.getElementById('website-url').value.trim();
  var cu = document.getElementById('currencies').value.trim();
  var cn = document.getElementById('contact-name').value.trim();
  var ce = document.getElementById('contact-email').value.trim();
  if (!bn || !wu || !cu || !cn || !ce) { alert('Please fill in all required fields.'); return; }
  var subject = 'Merchant Integration Request — ' + bn;
  var body = 'Business Name: ' + bn + '\nWebsite URL: ' + wu + '\nAvailable Currencies: ' + cu + '\nContact Person: ' + cn + '\nContact Email: ' + ce;
  window.location.href = 'mailto:integrations@zicharge.com?subject=' + encodeURIComponent(subject) + '&body=' + encodeURIComponent(body);
});
</script>
