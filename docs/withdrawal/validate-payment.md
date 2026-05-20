# Validate Payment

Confirms the status of a cashback transaction by `order_id`. Returns the canonical `transaction_id` for your records. Safe to call multiple times.

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
        "order_id": "CB-2026-000001"
      }'
    ```

=== "Java"

    ```java
    HttpClient client = HttpClient.newHttpClient();

    String body = """
        {
          "merchant_mobile_no": "+9647712345678",
          "store_password": "your-store-password",
          "order_id": "CB-2026-000001"
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
        'order_id'           => 'CB-2026-000001',
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
          order_id: 'CB-2026-000001',
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
| `order_id` | string | Required | The `order_id` submitted when initiating the cashback. |

---

## Response

```json
{
  "code": 200,
  "messages": ["Cashback confirmed."],
  "data": {
    "transaction_id": 789012,
    "order_id": "CB-2026-000001",
    "amount": "1000",
    "customer_mobile_no": "+9647701234567",
    "status": "Success",
    "received_at": "2026-05-20 10:05:00"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `data.transaction_id` | integer | Canonical ZiCharge transaction ID. **Use this in your ledger.** |
| `data.order_id` | string | Your submitted `order_id`. |
| `data.amount` | string | Credited amount in IQD. |
| `data.customer_mobile_no` | string | Recipient wallet number. |
| `data.status` | string | `"Success"` for a settled cashback. |
| `data.received_at` | string | UTC timestamp `"yyyy-MM-dd HH:mm:ss"`. |

---

## When to call

Because cashback settles synchronously, `transaction_id` is already returned in the [Cashback](cashback.md) response. Validate Payment is useful when:

- You need to **re-fetch** the transaction record after the fact.
- You want a **single reconciliation pattern** consistent with the deposit flow.
- Your cashback call timed out and you need to check whether the credit was applied.

!!! tip "Always use `transaction_id` in your books"
    The `transaction_id` is the canonical ZiCharge reference. Include it in all support queries.

---

## Error codes

| `code` | Meaning | Action |
|--------|---------|--------|
| `200` | Validation complete | Read `data.status` |
| `401` | Invalid credentials | Check `merchant_mobile_no` and `store_password` |
| `404` | Order not found | Verify `order_id` is correct |

[Full error code reference →](../06-error-codes.md)
