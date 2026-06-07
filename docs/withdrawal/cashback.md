# Cashback

Transfer funds from your merchant wallet to a customer's personal ZiCharge wallet. Settled synchronously — the customer's balance is updated in the same response.

Use for refunds, loyalty credits, and promotional disbursements.

---

## Endpoint

```
POST /api/v2/merchant/cash-back
```

| Environment | Base URL |
|-------------|----------|
| Sandbox | `https://dev.zicharge.com` |
| Production | `https://secure.zicharge.com` |

---

## Authentication

Two methods are supported. **Bearer token is recommended** — it keeps credentials out of the request body and simplifies key rotation.

<div class="zi-info-row" markdown>

<div class="zi-info-box" markdown>
**Bearer Token (Recommended)**

Generate a key via [Generate API Key](generate-api-key.md), then pass it in the `Authorization` header:

```
Authorization: Bearer <api_key>
```

When this header is present, `merchant_mobile_no` and `store_password` are **not required** in the body.
</div>

<div class="zi-info-box" markdown>
**Store Credentials (Fallback)**

Include `merchant_mobile_no` and `store_password` in the request body. No header required.

Use this if you haven't yet generated an API key.
</div>

</div>

---

## Request

=== "cURL — Bearer Token"

    ```bash
    curl -X POST https://secure.zicharge.com/api/v2/merchant/cash-back \
      -H "Content-Type: application/json" \
      -H "Accept: application/json" \
      -H "Authorization: Bearer your-api-key" \
      -d '{
        "receiver_mobile_no": "+9647701234567",
        "amount": 1000,
        "reference_id": "CB-2026-000001",
        "lang": "en"
      }'
    ```

=== "cURL — Store Credentials"

    ```bash
    curl -X POST https://secure.zicharge.com/api/v2/merchant/cash-back \
      -H "Content-Type: application/json" \
      -H "Accept: application/json" \
      -d '{
        "merchant_mobile_no": "+9647712345678",
        "store_password": "your-store-password",
        "receiver_mobile_no": "+9647701234567",
        "amount": 1000,
        "reference_id": "CB-2026-000001",
        "lang": "en"
      }'
    ```

=== "Java"

    ```java
    HttpClient client = HttpClient.newHttpClient();

    // Bearer token body (no credentials required in body)
    String body = """
        {
          "receiver_mobile_no": "+9647701234567",
          "amount": 1000,
          "reference_id": "CB-2026-000001",
          "lang": "en"
        }
        """;

    HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create("https://secure.zicharge.com/api/v2/merchant/cash-back"))
        .header("Content-Type", "application/json")
        .header("Accept", "application/json")
        .header("Authorization", "Bearer " + API_KEY)
        .POST(HttpRequest.BodyPublishers.ofString(body))
        .build();

    HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
    ```

=== "PHP"

    ```php
    // Bearer token auth — credentials not needed in body
    $payload = [
        'receiver_mobile_no' => '+9647701234567',
        'amount'             => 1000,
        'reference_id'       => 'CB-2026-000001',
        'lang'               => 'en',
    ];

    $ch = curl_init('https://secure.zicharge.com/api/v2/merchant/cash-back');
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_POST, true);
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Content-Type: application/json',
        'Accept: application/json',
        'Authorization: Bearer ' . API_KEY,
    ]);

    $response = json_decode(curl_exec($ch), true);
    curl_close($ch);
    ```

=== "JavaScript"

    ```javascript
    // Bearer token auth — credentials not needed in body
    const response = await fetch(
      'https://secure.zicharge.com/api/v2/merchant/cash-back',
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Accept': 'application/json',
          'Authorization': `Bearer ${API_KEY}`,
        },
        body: JSON.stringify({
          receiver_mobile_no: '+9647701234567',
          amount: 1000,
          reference_id: 'CB-2026-000001',
          lang: 'en',
        }),
      }
    );
    const data = await response.json();
    ```

---

## Request parameters

### Headers

| Header | Required | Description |
|--------|----------|-------------|
| `Authorization` | Optional | `Bearer <api_key>` — preferred auth method. When present, body credentials are ignored. |

### Body

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `receiver_mobile_no` | string | Required | Recipient's ZiCharge wallet number in E.164 format. Must be a **personal** account. |
| `amount` | number | Required | Amount in IQD to credit. |
| `reference_id` | string | Optional | Your unique reference for this cashback. Used as idempotency key — duplicate values are rejected. |
| `lang` | string | Optional | Response language: `en`, `ar`, or `ku`. Defaults to `en`. |
| `merchant_mobile_no` | string | Conditional | Required when not using Bearer token auth. |
| `store_password` | string | Conditional | Required when not using Bearer token auth. Never expose this client-side. |

---

## Response

=== "Success"

    ```json
    {
      "code": 200,
      "messages": ["The money has been sent."],
      "data": {
        "tx_unique_id": "CVIBAON916",
        "reference_id": "CB-2026-000001",
        "amount": 1000,
        "customer_account_no": "+9647701234567",
        "status": "Success",
        "sent_at": "2026-05-20 18:39:25"
      }
    }
    ```

| Field | Type | Description |
|-------|------|-------------|
| `data.tx_unique_id` | string | Public transaction reference. **Record this in your ledger.** |
| `data.reference_id` | string | Your submitted `reference_id`. |
| `data.amount` | integer | Amount credited in IQD. |
| `data.customer_account_no` | string | Recipient wallet number. |
| `data.status` | string | `"Success"` — settled synchronously. |
| `data.sent_at` | string | UTC timestamp `"yyyy-MM-dd HH:mm:ss"`. |

---

## Important constraints

!!! warning "Personal accounts only"
    `receiver_mobile_no` must be a **personal** ZiCharge wallet. Sending to a merchant or business account returns a `422` error.

!!! warning "Idempotency via `reference_id`"
    Once a `reference_id` is used in a successful cashback, reusing it returns a `422 Duplicate payment` error. Always generate a unique `reference_id` per disbursement.

---

## Error responses

| `code` | Message | Action |
|--------|---------|--------|
| `200` | `The money has been sent.` | Record `tx_unique_id` |
| `422` | `Invalid store credentials.` | Check `merchant_mobile_no` and `store_password`, or verify your API key |
| `422` | `Insufficient account balance.` | Top up your merchant wallet |
| `422` | `Invalid customer account. Only personal accounts can receive cashback.` | Verify `receiver_mobile_no` is a personal wallet |
| `422` | `Duplicate payment detected for the given reference ID.` | Use a different `reference_id` |
| `422` | `Transaction amount exceeds the allowed limit.` | Reduce the `amount` |

[Full error code reference →](../06-error-codes.md)
