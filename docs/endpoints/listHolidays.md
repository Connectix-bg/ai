# List public holidays

`GET /countries/holidays`

- Operation: `listHolidays`
- Tags: Account
- Product: account
- Base URL: https://api-sandbox.connectix.bg

Returns the non-working days of the current year for a country as `YYYY-MM-DD` dates - the same calendar Connectix uses to count working days for couriers and campaign scheduling. Only Bulgaria (`bg`) is supported; any other value returns 400.

Authenticate with the application token in the `Authorization` header: on `https://api-sandbox.connectix.bg` use the application's **sandbox token**; the production host uses the production token of the same application.

## Authentication

Send the token in the `Authorization` header. Application token sent as the raw value of the `Authorization` header (`Authorization: <token>`); `Bearer <token>` is also accepted. Use the sandbox token on `https://api-sandbox.connectix.bg`. Use the sandbox token from the console; the raw token and `Bearer <token>` are both accepted.

## Parameters

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `country` | query | yes | string, one of: bg | ISO 3166-1 alpha-2 country code, case-insensitive. Only `bg` is supported. |

## Responses

| Status | Description |
|---|---|
| 200 | Sorted list of dates. Returns array of string (date). |
| 400 | Invalid request. Either a required value is missing, the template is not approved / not found, the text exceeds the channel limit, or the body is not valid JSON (`{"error": "Invalid JSON: Syntax error"}`). Returns object (Error). |
| 401 | Missing or unknown token for this host, or the company is suspended. Check that you use the sandbox token on the sandbox host. Returns object (Error). |
| 429 | Rate limited. Either the per-application request budget for the current minute is spent (`{"message": "rate_limited", "retry_after": <seconds>}` with a `Retry-After` header; every response carries `X-RateLimit-Remaining`), or - on the OTP routes - the per-phone OTP limit was reached (plain string, default 5 OTPs per hour per application). Wait and retry; do not tighten the loop. Returns object (Error). |

### 200 response

```json
[
    "2026-01-01",
    "2026-03-03",
    "2026-04-10",
    "2026-04-13",
    "2026-05-01",
    "2026-05-06",
    "2026-05-25",
    "2026-09-07",
    "2026-09-22",
    "2026-12-24",
    "2026-12-25",
    "2026-12-28"
]
```

## Example request

```bash
curl -X GET "https://api-sandbox.connectix.bg/countries/holidays?country=bg" \
  -H "Authorization: $CONNECTIX_SANDBOX_TOKEN" \
  -H "Accept: application/json"
```
