# Validate Payment

Confirms the status of a cashback transaction by passing the `reference_id` as `order_id`. Returns the canonical `tx_unique_id` for your records. Safe to call multiple times.

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
        headers: { 'Content-Type': 'application/json', 'Accept': 'application/json' },
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
| `order_id` | string | Required | Pass the `reference_id` from the [Cashback](cashback.md) request as `order_id`. |

---

## Response

```json
{
  "code": 200,
  "messages": ["Please find your transaction details."],
  "data": {
    "tx_unique_id": "CVIBAON916",
    "order_id": "CB-2026-000001",
    "bill_amount": 1000,
    "customer_account_no": "+9647701234567",
    "status": "Success",
    "received_at": "2026-05-20 18:39:25"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `data.tx_unique_id` | string | Public transaction reference. **Use this in your ledger and for support queries.** |
| `data.order_id` | string | The `reference_id` you passed as `order_id`. |
| `data.bill_amount` | integer | Credited amount in IQD. |
| `data.customer_account_no` | string | Recipient wallet number. |
| `data.status` | string | `"Success"` for a settled cashback. |
| `data.received_at` | string | UTC timestamp `"yyyy-MM-dd HH:mm:ss"`. |

---

## When to call

Because cashback settles synchronously, `tx_unique_id` is already returned in the [Cashback](cashback.md) response. Validate Payment is useful when:

- You need to **re-fetch** the transaction record after the fact.
- Your cashback call **timed out** and you need to check whether the credit was applied before retrying.
- You want a **consistent reconciliation pattern** across both deposit and withdrawal flows.

!!! tip "Always record `tx_unique_id`"
    Include it in all support queries. It is the canonical ZiCharge reference for the transaction.

---

## Error responses

| `code` | Message | Action |
|--------|---------|--------|
| `200` | — | Check `data.status` |
| `404` | `Transaction not found.` | Verify `order_id` matches the `reference_id` used in Cashback |
| `422` | `Invalid store credentials.` | Check `merchant_mobile_no` and `store_password` |

[Full error code reference →](../06-error-codes.md)
