# List message templates

`GET /templates`

- Operation: `listTemplates`
- Tags: Templates
- Product: sms
- Base URL: https://api-sandbox.connectix.bg

Returns every template of the company (all channels, all statuses). Use it to find the `id` of an **approved** template before calling `sendMessage`; the template determines the channel, the message type and which `{{ placeholder }}` parameters must be supplied. Templates are created and approved by humans in the console - there is no API to create them.

Authenticate with the application token in the `Authorization` header: on `https://api-sandbox.connectix.bg` use the application's **sandbox token**; the production host uses the production token of the same application.

## Authentication

Send the token in the `Authorization` header. Application token sent as the raw value of the `Authorization` header (`Authorization: <token>`); `Bearer <token>` is also accepted. Use the sandbox token on `https://api-sandbox.connectix.bg`. Use the sandbox token from the console; the raw token and `Bearer <token>` are both accepted.

## Sandbox

Same route on both hosts. With the sandbox token it operates on the application's real data (nothing is mocked).

## Responses

| Status | Description |
|---|---|
| 200 | Templates of the company. Returns array of object (Template). |
| 401 | Missing or unknown token for this host, or the company is suspended. Check that you use the sandbox token on the sandbox host. Returns object (Error). |
| 403 | Access to this resource is disabled for the application (`access_restricted`), the TrustCheck eligibility rules are not met, or - on the production host only - an AI coding tool called a send/OTP route (`ai_tools_must_use_sandbox`: develop against the sandbox, going live is a human step). Returns object (Error). |
| 429 | Rate limited. Either the per-application request budget for the current minute is spent (`{"message": "rate_limited", "retry_after": <seconds>}` with a `Retry-After` header; every response carries `X-RateLimit-Remaining`), or - on the OTP routes - the per-phone OTP limit was reached (plain string, default 5 OTPs per hour per application). Wait and retry; do not tighten the loop. Returns object (Error). |

### 200 response

| Property | Type | Description |
|---|---|---|
| `[].id` | string (uuid) | Template ID to pass as `template` when sending. |
| `[].media` | string, nullable | Public URL of the template image/file, if any. |
| `[].layout` | string, one of: text, button, image, advanced, file, carousel |  |
| `[].name` | string |  |
| `[].channel` | object (Channel), one of: sms, viber, noti | Messaging channel. `noti` is the Connectix in-app notification channel. |
| `[].text` | string, nullable | Text with `{{ placeholder }}` variables that must be supplied in `parameters`. |
| `[].buttonLabel` | string, nullable |  |
| `[].buttonUrl` | string, nullable |  |
| `[].status` | string, one of: pending, approved, waiting_information, rejected | Only `approved` templates can be sent; other statuses are rejected with 400 `The template was not found.` |

```json
[
    {
        "id": "3644d7ac-2a95-4068-863e-6352ed78d667",
        "media": null,
        "layout": "button",
        "name": "Order shipped",
        "channel": "viber",
        "text": "Your order #{{ order_id }} has been shipped.",
        "buttonLabel": "Track order",
        "buttonUrl": "https://shop.example.com/orders/{{ order_id }}",
        "status": "approved"
    }
]
```

## Example request

```bash
curl -X GET "https://api-sandbox.connectix.bg/templates" \
  -H "Authorization: $CONNECTIX_SANDBOX_TOKEN" \
  -H "Accept: application/json"
```
