# IPN Callback

ZiCharge POSTs to your `ipn_url` when a payment reaches a terminal state. Your listener must verify the payload, persist the record, and return HTTP 200.

<div class="zi-info-row" markdown>
<div class="zi-info-box" markdown>
**Delivery**

At-least-once. Build your listener to be idempotent on `tx_unique_id` + `order_id`.
</div>
<div class="zi-info-box" markdown>
**Retries**

Up to 2 attempts per terminal event. If the first delivery fails, a retry is sent after 5 minutes.
</div>
<div class="zi-info-box" markdown>
**Signature**

HMAC-SHA256 in Stripe-style format. Enable on your dashboard for authenticity proof.
</div>
</div>

---

## Payload

ZiCharge POSTs `application/x-www-form-urlencoded` to your `ipn_url`.

| Field | Type | On `Success` | On `Cancelled` |
|-------|------|-------------|----------------|
| `tx_unique_id` | string | Public transaction reference e.g. `"CVIBAON916"` | `""` empty string |
| `transaction_id` | string | Legacy numeric internal ID | `""` empty string |
| `order_id` | string | Your original `order_id` | Same |
| `bill_amount` | string | e.g. `"1000.00"` | Same |
| `customer_account_no` | string | Customer mobile `"+9647701234567"` | Customer mobile if known, else `""` |
| `status` | string | `"Success"` | `"Cancelled"` |
| `received_at` | string | `"yyyy-MM-dd HH:mm:ss"` UTC | Same |
| `ipn` | string | `"1"` | `"1"` |

!!! tip "Use `tx_unique_id` for reconciliation"
    `tx_unique_id` is the public transaction reference and is preferred for reconciliation and support queries. `transaction_id` is a legacy numeric internal ID — include it for completeness but correlate primarily on `order_id` + `tx_unique_id`.

### Signature header

Included when payload signatures are enabled on your merchant config:

| Header | Description |
|--------|-------------|
| `X-ZiCharge-Signature` | `t=<epoch>,v1=<hex-hmac>` — Stripe-style compound signature. `t` is the dispatch Unix timestamp; `v1` is the HMAC-SHA256 hex digest. |

The signed payload is constructed as:

```
signed_payload = t + "." + raw_body
expected       = HMAC-SHA256(webhook_secret, signed_payload)
```

Reject the request if `|now - t| > 300` seconds (5-minute replay window).

---

## Listener implementation

=== "Java"

    ```java
    // Java (Spring Boot)
    @PostMapping("/webhooks/zicharge/ipn")
    public ResponseEntity<Void> handleIpn(
            HttpServletRequest request,
            @RequestHeader("X-ZiCharge-Signature") String signatureHeader) throws Exception {

        byte[] rawBodyBytes = request.getInputStream().readAllBytes();
        String rawBody = new String(rawBodyBytes, "UTF-8");

        // 1. Parse header: t=<epoch>,v1=<hmac>
        String timestamp    = signatureHeader.replaceFirst(".*t=(\\d+).*", "$1");
        String receivedHmac = signatureHeader.replaceFirst(".*v1=([a-f0-9]+).*", "$1");

        // 2. Timestamp drift — reject if older than 5 minutes
        long ts = Long.parseLong(timestamp);
        if (Math.abs(System.currentTimeMillis() / 1000L - ts) > 300) {
            return ResponseEntity.badRequest().build();
        }

        // 3. Reconstruct signed payload and verify
        String signedPayload = timestamp + "." + rawBody;
        Mac mac = Mac.getInstance("HmacSHA256");
        mac.init(new SecretKeySpec(WEBHOOK_SECRET.getBytes("UTF-8"), "HmacSHA256"));
        StringBuilder sb = new StringBuilder();
        for (byte b : mac.doFinal(signedPayload.getBytes("UTF-8"))) {
            sb.append(String.format("%02x", b));
        }
        if (!MessageDigest.isEqual(sb.toString().getBytes(), receivedHmac.getBytes())) {
            return ResponseEntity.badRequest().build();
        }

        // 4. Parse fields
        Map<String, String> fields = parseFormBody(rawBody);
        String status     = fields.get("status");
        String orderId    = fields.get("order_id");
        String txUniqueId = fields.get("tx_unique_id");

        // 5. Idempotency — deduplicate by order_id + status
        if (ipnLogRepository.exists(orderId, status)) {
            return ResponseEntity.ok().build();
        }

        // 6. Mark order as paid only on Success
        if ("Success".equals(status)) {
            orderService.markPaid(orderId, txUniqueId);
        }

        // 7. Persist IPN record
        ipnLogRepository.save(fields);

        // 8. Acknowledge ONLY after persistence
        return ResponseEntity.ok().build();
    }
    ```

=== "PHP"

    ```php
    function handleIpn(Request $request): Response {
        $body = $request->getRawBody();

        // 1. Parse signature header: t=<epoch>,v1=<hmac>
        $sigHeader = $request->header('X-ZiCharge-Signature');
        preg_match('/t=(\d+),v1=([a-f0-9]+)/', $sigHeader, $m);
        $timestamp    = $m[1];
        $receivedHmac = $m[2];

        // 2. Timestamp drift
        if (abs(time() - (int) $timestamp) > 300) {
            return new Response(400);
        }

        // 3. Verify signature
        $signedPayload = $timestamp . '.' . $body;
        $expected = hash_hmac('sha256', $signedPayload, WEBHOOK_SECRET);
        if (!hash_equals($expected, $receivedHmac)) {
            return new Response(400);
        }

        // 4. Parse
        parse_str($body, $fields);
        $status     = $fields['status'];
        $orderId    = $fields['order_id'];
        $txUniqueId = $fields['tx_unique_id'];

        // 5. Idempotency
        if (IpnLog::exists($orderId, $status)) {
            return new Response(200);
        }

        // 6. Mark order paid only on Success
        if ($status === 'Success') {
            Order::markPaid($orderId, $txUniqueId);
        }

        // 7. Persist and acknowledge
        IpnLog::store($fields);
        return new Response(200);
    }
    ```

=== "JavaScript"

    ```javascript
    const crypto = require('crypto');

    async function handleIpn(req, res) {
        const body = req.rawBody;

        // 1. Parse signature header: t=<epoch>,v1=<hmac>
        const sigHeader = req.headers['x-zicharge-signature'] || '';
        const match = sigHeader.match(/t=(\d+),v1=([a-f0-9]+)/);
        if (!match) return res.status(400).end();
        const [, timestamp, receivedHmac] = match;

        // 2. Timestamp drift
        if (Math.abs(Date.now() / 1000 - parseInt(timestamp, 10)) > 300) {
            return res.status(400).end();
        }

        // 3. Verify signature
        const signedPayload = `${timestamp}.${body.toString()}`;
        const expected = crypto
            .createHmac('sha256', WEBHOOK_SECRET)
            .update(signedPayload)
            .digest('hex');
        if (!crypto.timingSafeEqual(
            Buffer.from(expected, 'utf8'),
            Buffer.from(receivedHmac, 'utf8')
        )) {
            return res.status(400).end();
        }

        // 4. Parse
        const fields = Object.fromEntries(new URLSearchParams(body.toString()));
        const { status, order_id, tx_unique_id } = fields;

        // 5. Idempotency
        if (await isProcessed(order_id, status)) {
            return res.status(200).end();
        }

        // 6. Mark order paid only on Success
        if (status === 'Success') {
            await orderService.markPaid(order_id, tx_unique_id);
        }

        // 7. Persist and acknowledge
        await persistIpn(fields);
        res.status(200).end();
    }
    ```

!!! warning "Always verify the raw body bytes"
    Build `signed_payload` from the raw request bytes — **before any parsing**. Re-encoding parsed fields changes byte order and will break the signature match.

!!! danger "Acknowledge only after you persist"
    Return **2xx only after** writing the IPN to your database. A 2xx permanently stops retries. Return **5xx** if your database is unavailable — ZiCharge will retry.

---

## HTTP response guide

| Your response | What happens |
|---------------|-------------|
| **2xx** | Delivered. Retries permanently stopped. |
| 3xx | Not followed. Treated as failure. Retry scheduled. |
| 4xx | Failure. Retry scheduled. |
| **5xx** | Failure. Retry scheduled. Use this when your DB is down. |
| Timeout > 10 s | Failure. Retry scheduled. |

---

## Delivery guarantees

| Guarantee | Detail |
|-----------|--------|
| **At-least-once** | Deduplicate on `(order_id, status)` |
| **No Success after Cancelled** | One terminal outcome per token, guaranteed |
| **5-minute retry gap** | If the first attempt fails, one retry is sent after 5 minutes |
| **Max 2 attempts** | After exhaustion, recover with [Validate Payment](validate-payment.md) |
