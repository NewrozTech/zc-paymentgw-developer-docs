# Cashback

Push a credit from your merchant wallet directly to a customer's ZiCharge wallet. Settled synchronously — the customer's balance is updated in the same response. Use for refunds, loyalty credits, and promotional disbursements.

---

## Endpoint

```
POST /merchant/payment/cashback
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
    curl -X POST https://secure.zicharge.com/merchant/payment/cashback \
      -H "Content-Type: application/json" \
      -H "Accept: application/json" \
      -d '{
        "merchant_mobile_no": "+9647712345678",
        "store_password": "your-store-password",
        "customer_mobile_no": "+9647701234567",
        "amount": 1000,
        "order_id": "CB-2026-000001",
        "remarks": "Refund for order ORD-2026-000123"
      }'
    ```

=== "Java"

    ```java
    HttpClient client = HttpClient.newHttpClient();

    String body = """
        {
          "merchant_mobile_no": "+9647712345678",
          "store_password": "your-store-password",
          "customer_mobile_no": "+9647701234567",
          "amount": 1000,
          "order_id": "CB-2026-000001",
          "remarks": "Refund for order ORD-2026-000123"
        }
        """;

    HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create("https://secure.zicharge.com/merchant/payment/cashback"))
        .header("Content-Type", "application/json")
        .header("Accept", "application/json")
        .POST(HttpRequest.BodyPublishers.ofString(body))
        .build();

    HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
    ```

=== "PHP"

    ```php
    $payload = [
        'merchant_mobile_no'  => '+9647712345678',
        'store_password'      => 'your-store-password',
        'customer_mobile_no'  => '+9647701234567',
        'amount'              => 1000,
        'order_id'            => 'CB-2026-000001',
        'remarks'             => 'Refund for order ORD-2026-000123',
    ];

    $ch = curl_init('https://secure.zicharge.com/merchant/payment/cashback');
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
      'https://secure.zicharge.com/merchant/payment/cashback',
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Accept': 'application/json',
        },
        body: JSON.stringify({
          merchant_mobile_no: '+9647712345678',
          store_password: 'your-store-password',
          customer_mobile_no: '+9647701234567',
          amount: 1000,
          order_id: 'CB-2026-000001',
          remarks: 'Refund for order ORD-2026-000123',
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
| `store_password` | string | Required | Your store password. Never expose this client-side. |
| `customer_mobile_no` | string | Required | Recipient's ZiCharge wallet number in E.164 format. |
| `amount` | number | Required | Amount in IQD to credit. Minimum 250. |
| `order_id` | string | Required | Your unique reference for this cashback. Used for idempotency and reconciliation. |
| `remarks` | string | Optional | Free-text note attached to the transaction (e.g., `"Refund for ORD-2026-000123"`). |

---

## Response

=== "Success"

    ```json
    {
      "code": 200,
      "messages": ["Cashback sent successfully."],
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

=== "Insufficient balance"

    ```json
    {
      "code": 422,
      "messages": ["Insufficient merchant balance."],
      "data": null
    }
    ```

| Field | Type | Description |
|-------|------|-------------|
| `data.transaction_id` | integer | Canonical ZiCharge transaction ID. Record this against your order. |
| `data.order_id` | string | Your submitted `order_id`. |
| `data.amount` | string | Amount credited in IQD. |
| `data.customer_mobile_no` | string | Recipient wallet number. |
| `data.status` | string | `"Success"` — cashback is settled synchronously. |
| `data.received_at` | string | UTC timestamp `"yyyy-MM-dd HH:mm:ss"`. |

---

## Settlement behaviour

Cashback settles **synchronously**. A `code: 200` response means the credit has already landed in the customer's wallet — there is no async confirmation step required. After a successful response, you may optionally call [Validate Payment](validate-payment.md) to fetch a full transaction record.

!!! warning "Merchant balance check"
    Ensure your merchant wallet has sufficient balance before initiating cashback. The gateway performs an atomic balance check — a `code: 422` means the debit failed and no funds were moved.

---

## Error codes

| `code` | Meaning | Action |
|--------|---------|--------|
| `200` | Cashback sent | Record `transaction_id` |
| `401` | Invalid credentials | Check `merchant_mobile_no` and `store_password` |
| `404` | Customer wallet not found | Verify `customer_mobile_no` |
| `409` | `order_id` already processed | Idempotent — check existing transaction |
| `422` | Validation failure or insufficient balance | Check amount (min 250 IQD) and merchant balance |

[Full error code reference →](../06-error-codes.md)
