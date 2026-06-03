# Generate API Key

Generate (or regenerate) a merchant API key for authenticating cashback requests. The key is returned **once only** — store it securely as you would a password.

!!! tip "Use Bearer auth for cashback"
    Once you have an API key, pass it as `Authorization: Bearer <api_key>` on every Cashback request. You no longer need to include `merchant_mobile_no` or `store_password` in the request body — the key is sufficient.

---

## Endpoint

```
POST /api/v3/merchant/generate-api-key
```

| Environment | Base URL |
|-------------|----------|
| Sandbox | `https://dev.zicharge.com` |
| Production | `https://secure.zicharge.com` |

---

## Request

=== "cURL"

    ```bash
    curl -X POST https://secure.zicharge.com/api/v3/merchant/generate-api-key \
      -H "Content-Type: application/json" \
      -H "Accept: application/json" \
      -d '{
        "merchant_mobile_no": "+9647712345678",
        "store_password": "your-store-password"
      }'
    ```

=== "Java"

    ```java
    HttpClient client = HttpClient.newHttpClient();

    String body = """
        {
          "merchant_mobile_no": "+9647712345678",
          "store_password": "your-store-password"
        }
        """;

    HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create("https://secure.zicharge.com/api/v3/merchant/generate-api-key"))
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
    ];

    $ch = curl_init('https://secure.zicharge.com/api/v3/merchant/generate-api-key');
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
      'https://secure.zicharge.com/api/v3/merchant/generate-api-key',
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json', 'Accept': 'application/json' },
        body: JSON.stringify({
          merchant_mobile_no: '+9647712345678',
          store_password: 'your-store-password',
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

---

## Response

=== "Success"

    ```json
    {
      "code": 200,
      "messages": [],
      "data": {
        "api_key": "a3f8e2c1d5b94f7e2a1c8d3e6f9b0c4d7e2a1b5c8d3f6e9a0b4c7d2e5f8a1b4"
      }
    }
    ```

| Field | Type | Description |
|-------|------|-------------|
| `data.api_key` | string | Your merchant API key. **Shown once — copy it now.** Use as `Authorization: Bearer <api_key>` on Cashback requests. |

---

## Using the API key

Once generated, include the key in the `Authorization` header on every [Cashback](cashback.md) request:

```
Authorization: Bearer a3f8e2c1d5b94f7e2a1c8d3e6f9b0c4d7e2a1b5c8d3f6e9a0b4c7d2e5f8a1b4
```

When the Bearer token is present, `merchant_mobile_no` and `store_password` are **not required** in the Cashback request body.

---

## Important notes

!!! warning "Key is shown once"
    The `api_key` is returned only in this response. If you lose it, regenerate a new one — the old key is immediately invalidated.

!!! danger "Regeneration invalidates the previous key"
    Calling this endpoint again issues a new key and **immediately invalidates** the previous one. Any in-flight Cashback requests using the old key will fail. Plan key rotations carefully.

!!! danger "Treat the API key like store_password"
    Store it in an environment variable or secrets manager. Never log it, never expose it client-side, never commit it to source control.

---

## Error responses

| `code` | Message | Action |
|--------|---------|--------|
| `200` | — | Copy `data.api_key` immediately |
| `422` | `Invalid store credentials.` | Check `merchant_mobile_no` and `store_password` |

[Full error code reference →](../06-error-codes.md)
