# Validate Payment

Confirms the final status of a payment by `order_id`. Call this after receiving an IPN callback to obtain the canonical `transaction_id` for your records. Safe to call multiple times.

---

## Endpoint

```
POST /merchant/payment/validation
```

**Base URLs**

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
        "order_id": "ORD-2026-000123"
      }'
    ```

=== "Java"

    ```java
    HttpClient client = HttpClient.newHttpClient();

    String body = """
        {
          "merchant_mobile_no": "+9647712345678",
          "store_password": "your-store-password",
          "order_id": "ORD-2026-000123"
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
        'order_id'           => 'ORD-2026-000123',
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
        headers: {
          'Content-Type': 'application/json',
          'Accept': 'application/json',
        },
        body: JSON.stringify({
          merchant_mobile_no: '+9647712345678',
          store_password: 'your-store-password',
          order_id: 'ORD-2026-000123',
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
| `order_id` | string | Required | The order ID you submitted when minting the token. |

---

## Response

=== "Success"

    ```json
    {
      "code": 200,
      "messages": ["Successfully Deposited."],
      "data": {
        "transaction_id": 123456,
        "order_id": "ORD-2026-000123",
        "bill_amount": "5000",
        "customer_account_no": "+9647701234567",
        "status": "Success",
        "received_at": "2026-05-20 10:00:00"
      }
    }
    ```

=== "Cancelled / Not paid"

    ```json
    {
      "code": 200,
      "messages": ["Payment not completed."],
      "data": {
        "order_id": "ORD-2026-000123",
        "status": "Cancelled"
      }
    }
    ```

| Field | Type | Description |
|-------|------|-------------|
| `data.transaction_id` | integer | Canonical ZiCharge transaction ID. **Use this in your ledger.** |
| `data.order_id` | string | Your original order ID. |
| `data.bill_amount` | string | Amount in IQD as a string. |
| `data.customer_account_no` | string | The paying customer's mobile number. |
| `data.status` | string | `"Success"` or `"Cancelled"`. |
| `data.received_at` | string | UTC timestamp of the transaction `"yyyy-MM-dd HH:mm:ss"`. |

---

## When to call

Call Validate Payment in two scenarios:

1. **After receiving an IPN callback** — to confirm the transaction and record `transaction_id` in your books.
2. **Missed IPN recovery** — if your IPN listener was unavailable during both delivery attempts, poll this endpoint to reconcile the order status.

!!! tip "Always use `transaction_id` in your books"
    The `transaction_id` returned here is the canonical ZiCharge reference. Record it against your order for support queries and reconciliation.

---

## Error codes

| `code` | Meaning | Action |
|--------|---------|--------|
| `200` | Validation complete | Read `data.status` for the outcome |
| `401` | Invalid credentials | Check `merchant_mobile_no` and `store_password` |
| `404` | Order not found | Verify `order_id` is correct |

[Full error code reference →](../06-error-codes.md)
