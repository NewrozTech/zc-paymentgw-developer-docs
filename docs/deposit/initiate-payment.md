# Initiate Payment

Mint a single-use payment token and checkout URL for a customer order. Redirect the customer to `payment_url` or display it as a QR code. The token expires after `expires_in_minutes` minutes.

---

## Endpoint

```
POST /merchant/generate-payment-token
```

Also available at alias: `/merchant/generate-qr-token`

| Environment | Base URL |
|-------------|----------|
| Sandbox | `https://dev.zicharge.com` |
| Production | `https://secure.zicharge.com` |

---

## Request

=== "cURL"

    ```bash
    curl -X POST https://secure.zicharge.com/merchant/generate-payment-token \
      -H "Content-Type: application/json" \
      -H "Accept: application/json" \
      -d '{
        "merchant_mobile_no": "+9647712345678",
        "store_password": "your-store-password",
        "order_id": "ORD-2026-000001",
        "bill_amount": 1000,
        "success_url": "https://yoursite.com/payment/success",
        "cancel_url": "https://yoursite.com/payment/cancel",
        "fail_url": "https://yoursite.com/payment/fail",
        "customer_mobile_no": "+9647701234567"
      }'
    ```

=== "Java"

    ```java
    HttpClient client = HttpClient.newHttpClient();

    String body = """
        {
          "merchant_mobile_no": "+9647712345678",
          "store_password": "your-store-password",
          "order_id": "ORD-2026-000001",
          "bill_amount": 1000,
          "success_url": "https://yoursite.com/payment/success",
          "cancel_url": "https://yoursite.com/payment/cancel",
          "fail_url": "https://yoursite.com/payment/fail",
          "customer_mobile_no": "+9647701234567"
        }
        """;

    HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create("https://secure.zicharge.com/merchant/generate-payment-token"))
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
        'bill_amount'        => 1000,
        'success_url'        => 'https://yoursite.com/payment/success',
        'cancel_url'         => 'https://yoursite.com/payment/cancel',
        'fail_url'           => 'https://yoursite.com/payment/fail',
        'customer_mobile_no' => '+9647701234567',
    ];

    $ch = curl_init('https://secure.zicharge.com/merchant/generate-payment-token');
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
      'https://secure.zicharge.com/merchant/generate-payment-token',
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json', 'Accept': 'application/json' },
        body: JSON.stringify({
          merchant_mobile_no: '+9647712345678',
          store_password: 'your-store-password',
          order_id: 'ORD-2026-000001',
          bill_amount: 1000,
          success_url: 'https://yoursite.com/payment/success',
          cancel_url: 'https://yoursite.com/payment/cancel',
          fail_url: 'https://yoursite.com/payment/fail',
          customer_mobile_no: '+9647701234567',
        }),
      }
    );
    const data = await response.json();
    ```

---

## Request parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `merchant_mobile_no` | string | Required | Your merchant wallet number in E.164 format, e.g. `+9647712345678`. |
| `store_password` | string | Required | Your store password. Never expose this in client-side code. |
| `order_id` | string | Required | Your unique order identifier. The same `order_id` will not create a duplicate charge. |
| `bill_amount` | number | Required | Amount in IQD. Minimum 250. |
| `success_url` | string | Required | URL the customer is redirected to after a successful payment. |
| `cancel_url` | string | Required | URL the customer is redirected to after cancellation. |
| `fail_url` | string | Optional | URL the customer is redirected to after a failed payment attempt. |
| `customer_mobile_no` | string | Optional | Pre-fills the customer's wallet number on the checkout page. |

---

## Response

```json
{
  "code": 200,
  "messages": [],
  "data": {
    "token": "PMTKN7X2K9Q",
    "payment_url": "https://dev.zicharge.com/merchant/payment?token=PMTKN7X2K9Q",
    "expires_in_minutes": 15
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `data.token` | string | Single-use payment token bound to this order. |
| `data.payment_url` | string | Hosted checkout URL. Redirect the customer here, or render as a QR code. |
| `data.expires_in_minutes` | integer | Minutes until the token expires (typically 15). |

---

## What to do next

After minting the token:

- **Web checkout** — redirect the customer to `payment_url`
- **QR / in-store** — render `payment_url` as a QR code

Once the customer pays, ZiCharge posts an [IPN callback](ipn-callback.md) to your `ipn_url`. Call [Validate Payment](validate-payment.md) to independently confirm the result.

!!! warning "Token expiry"
    The token is valid for `expires_in_minutes` minutes. If a customer opens an expired token, the gateway cancels it and fires a `Cancelled` IPN.

---

## Error responses

| `code` | Message | Action |
|--------|---------|--------|
| `200` | — | Token minted. Redirect customer to `payment_url`. |
| `422` | `Invalid store credentials.` | Check `merchant_mobile_no` and `store_password`. |
| `422` | `Account is inactive.` | Contact integrations@zicharge.com. |
| `422` | `merchant_mobile_no must not be blank` | Add the missing required field. |

[Full error code reference →](../06-error-codes.md)
