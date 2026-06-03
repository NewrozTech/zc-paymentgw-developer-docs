# Validate Payment

Confirms the final status of a payment by `order_id`. Returns the canonical `tx_unique_id` to record in your ledger. Safe to call multiple times.

---

## Endpoint

```
POST /merchant/payment/validation
```

Also available at alias: `/merchant/validate-payment`

| Environment | Base URL |
|-------------|----------|
| Sandbox | `https://dev.zicharge.com` |
| Production | `https://secure.zicharge.com` |

---

## Request

=== "cURL"

    ```bash
    curl -X POST https://secure.zicharge.com/merchant/payment/validation \
      -H "Content-Type: application/json" \
      -H "Accept: application/json" \
      -d '{
        "merchant_mobile_no": "+9647712345678",
        "store_password": "your-store-password",
        "order_id": "ORD-2026-000001"
      }'
    ```

=== "Java"

    ```java
    HttpClient client = HttpClient.newHttpClient();

    String body = """
        {
          "merchant_mobile_no": "+9647712345678",
          "store_password": "your-store-password",
          "order_id": "ORD-2026-000001"
        }
        """;

    HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create("https://secure.zicharge.com/merchant/payment/validation"))
        .header("Content-Type", "application/json")
        .header("Accept", "application/json")
        .POST(HttpRequest.BodyPublishers.ofString(body))
        .build();

    HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
    ```

=== "PHP"

    ```php
    $payload = [
        'merchant_mobile_no' => '+9647712345678',
        'store_password'     => 'your-store-password',
        'order_id'           => 'ORD-2026-000001',
    ];

    $ch = curl_init('https://secure.zicharge.com/merchant/payment/validation');
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_POST, true);
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Content-Type: application/json',
        'Accept: application/json',
    ]);

    $response = json_decode(curl_exec($ch), true);
    curl_close($ch);
    ```

=== "JavaScript"

    ```javascript
    const response = await fetch(
      'https://secure.zicharge.com/merchant/payment/validation',
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json', 'Accept': 'application/json' },
        body: JSON.stringify({
          merchant_mobile_no: '+9647712345678',
          store_password: 'your-store-password',
          order_id: 'ORD-2026-000001',
        }),
      }
    );
    const data = await response.json();
    ```

---

## Request parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `merchant_mobile_no` | string | Required | Your merchant wallet number in E.164 format. |
| `store_password` | string | Required | Your store password. |
| `order_id` | string | Required | The `order_id` you submitted when minting the token. |

---

## Response

=== "Success"

    ```json
    {
      "code": 200,
      "messages": ["Please find your transaction details."],
      "data": {
        "tx_unique_id": "CVIBAON916",
        "order_id": "ORD-2026-000001",
        "bill_amount": 1000,
        "customer_account_no": "+9647701234567",
        "status": "Success",
        "received_at": "2026-05-20 10:00:00"
      }
    }
    ```

=== "Cancelled"

    ```json
    {
      "code": 200,
      "messages": ["Please find your transaction details."],
      "data": {
        "tx_unique_id": "CVIBAON916",
        "order_id": "ORD-2026-000001",
        "bill_amount": 1000,
        "customer_account_no": "+9647701234567",
        "status": "Cancelled",
        "received_at": "2026-05-20 10:05:00"
      }
    }
    ```

=== "Not found"

    ```json
    {
      "code": 404,
      "messages": ["Transaction not found."],
      "data": null
    }
    ```

| Field | Type | Description |
|-------|------|-------------|
| `data.tx_unique_id` | string | Public transaction reference. **Use this in your ledger and for support queries.** |
| `data.order_id` | string | Your original order ID. |
| `data.bill_amount` | integer | Amount in IQD. |
| `data.customer_account_no` | string | The paying customer's wallet number. |
| `data.status` | string | `"Success"` or `"Cancelled"`. |
| `data.received_at` | string | UTC timestamp `"yyyy-MM-dd HH:mm:ss"`. |

---

## When to call

1. **After receiving an IPN callback** — to independently confirm the payment and record `tx_unique_id`.
2. **Missed IPN recovery** — if your listener was down during both delivery attempts, poll this endpoint to reconcile.

!!! tip "Always record `tx_unique_id`"
    This is the canonical ZiCharge public reference. Always store it against your order and include it in any support requests.

---

## Error responses

| `code` | Message | Action |
|--------|---------|--------|
| `200` | — | Check `data.status` for `Success` or `Cancelled` |
| `404` | `Transaction not found.` | Verify `order_id` is correct and the token was ever paid |
| `422` | `Invalid store credentials.` | Check `merchant_mobile_no` and `store_password` |

[Full error code reference →](../06-error-codes.md)
