# Initiate Payment

Generate a payment token for a customer order. The token powers the hosted checkout page — redirect the customer to it after minting.

---

## Endpoint

```
POST /merchant/generate-payment-token
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
    curl -X POST https://secure.zicharge.com/merchant/generate-payment-token \
      -H "Content-Type: application/json" \
      -H "Accept: application/json" \
      -d '{
        "merchant_mobile_no": "+9647712345678",
        "store_password": "your-store-password",
        "order_id": "ORD-2026-000123",
        "bill_amount": 5000,
        "success_url": "https://yoursite.com/orders/123/success",
        "cancel_url": "https://yoursite.com/orders/123/cancel",
        "ipn_url": "https://yoursite.com/webhooks/zicharge/ipn"
      }'
    ```

=== "Java"

    ```java
    HttpClient client = HttpClient.newHttpClient();

    String body = """
        {
          "merchant_mobile_no": "+9647712345678",
          "store_password": "your-store-password",
          "order_id": "ORD-2026-000123",
          "bill_amount": 5000,
          "success_url": "https://yoursite.com/orders/123/success",
          "cancel_url": "https://yoursite.com/orders/123/cancel",
          "ipn_url": "https://yoursite.com/webhooks/zicharge/ipn"
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
        'order_id'           => 'ORD-2026-000123',
        'bill_amount'        => 5000,
        'success_url'        => 'https://yoursite.com/orders/123/success',
        'cancel_url'         => 'https://yoursite.com/orders/123/cancel',
        'ipn_url'            => 'https://yoursite.com/webhooks/zicharge/ipn',
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
        headers: {
          'Content-Type': 'application/json',
          'Accept': 'application/json',
        },
        body: JSON.stringify({
          merchant_mobile_no: '+9647712345678',
          store_password: 'your-store-password',
          order_id: 'ORD-2026-000123',
          bill_amount: 5000,
          success_url: 'https://yoursite.com/orders/123/success',
          cancel_url: 'https://yoursite.com/orders/123/cancel',
          ipn_url: 'https://yoursite.com/webhooks/zicharge/ipn',
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
| `bill_amount` | number | Required | Amount in IQD. Minimum 250. Validated with `BigDecimal` precision — no floating-point rounding. |
| `success_url` | string | Required | URL the customer is redirected to after a successful payment. |
| `cancel_url` | string | Required | URL the customer is redirected to after cancelling. |
| `ipn_url` | string | Optional | Your IPN listener endpoint. Must be HTTPS. Overrides the default set in your dashboard. |

---

## Response

Every response returns HTTP 200. The logical outcome is in the `code` field.

```json
{
  "code": 200,
  "messages": ["Token generated successfully."],
  "data": {
    "payment_token": "tok_a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4",
    "redirect_url": "https://secure.zicharge.com/merchant/payment?token=tok_a1b2c3d4...",
    "order_id": "ORD-2026-000123",
    "expires_in_seconds": 900
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `data.payment_token` | string | Opaque token bound to this order. Valid for 15 minutes. |
| `data.redirect_url` | string | Hosted checkout URL. Redirect the customer here immediately after minting. |
| `data.order_id` | string | Echoes your submitted `order_id`. |
| `data.expires_in_seconds` | integer | Seconds until the token expires (900 = 15 minutes). |

---

## What to do next

After minting the token, redirect the customer to `redirect_url`:

```
https://secure.zicharge.com/merchant/payment?token=<payment_token>
```

The customer sees a one-page checkout: they enter their ZiCharge mobile number and PIN. Once paid, ZiCharge dispatches an [IPN callback](ipn-callback.md) to your listener.

!!! warning "Token expiry"
    The token is valid for **15 minutes**. If the customer opens an expired token, the gateway cancels it and fires a `Cancelled` IPN.

---

## Error codes

| `code` | Meaning | Action |
|--------|---------|--------|
| `200` | Token minted | Redirect customer to `redirect_url` |
| `401` | Invalid credentials | Check `merchant_mobile_no` and `store_password` |
| `409` | `order_id` already exists | Use a new unique `order_id` |
| `422` | Validation failure | Check `bill_amount` (min 250 IQD) and required fields |

[Full error code reference →](../06-error-codes.md)
