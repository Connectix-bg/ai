# Identify the application behind the token

`GET /me`

- Operation: `getMe`
- Tags: Account
- Product: account
- Base URL: https://api-sandbox.connectix.bg

Returns the name of the application the token belongs to. Use it to verify a token and the host you are talking to before doing anything else.

Authenticate with the application token in the `Authorization` header: on `https://api-sandbox.connectix.bg` use the application's **sandbox token**; the production host uses the production token of the same application.

## Authentication

Send the token in the `Authorization` header. Application token sent as the raw value of the `Authorization` header (`Authorization: <token>`); `Bearer <token>` is also accepted. Use the sandbox token on `https://api-sandbox.connectix.bg`. Use the sandbox token from the console; the raw token and `Bearer <token>` are both accepted.

## Responses

| Status | Description |
|---|---|
| 200 | The authenticated application. Returns object (Me). |
| 401 | Missing or unknown token for this host, or the company is suspended. Check that you use the sandbox token on the sandbox host. Returns object (Error). |
| 429 | Rate limited. Either the per-application request budget for the current minute is spent (`{"message": "rate_limited", "retry_after": <seconds>}` with a `Retry-After` header; every response carries `X-RateLimit-Remaining`), or - on the OTP routes - the per-phone OTP limit was reached (plain string, default 5 OTPs per hour per application). Wait and retry; do not tighten the loop. Returns object (Error). |

### 200 response

| Property | Type | Description |
|---|---|---|
| `name` | string | Name of the application the token belongs to. |

```json
{
    "name": "Web shop"
}
```

## Example request

```bash
curl -X GET "https://api-sandbox.connectix.bg/me" \
  -H "Authorization: $CONNECTIX_SANDBOX_TOKEN" \
  -H "Accept: application/json"
```
