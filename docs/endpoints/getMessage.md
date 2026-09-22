# Get a message

`GET /messages/{id}`

- Operation: `getMessage`
- Tags: Messages
- Product: sms
- Base URL: https://api-sandbox.connectix.bg

Returns one message of the application by its `id` (the UUID from the `sendMessage` answer and from the webhooks) with its current status. The reliable way to catch up after a missed `callback` webhook.

Authenticate with the application token in the `Authorization` header: on `https://api-sandbox.connectix.bg` use the application's **sandbox token**; the production host uses the production token of the same application.

## Authentication

Send the token in the `Authorization` header. Application token sent as the raw value of the `Authorization` header (`Authorization: <token>`); `Bearer <token>` is also accepted. Use the sandbox token on `https://api-sandbox.connectix.bg`. Use the sandbox token from the console; the raw token and `Bearer <token>` are both accepted.

## Sandbox

Reads a sandbox message of the application.

## Parameters

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `id` | path | yes | string (uuid) | Message ID. |

## Responses

| Status | Description |
|---|---|
| 200 | The message. Returns object (Message). |
| 401 | Missing or unknown token for this host, or the company is suspended. Check that you use the sandbox token on the sandbox host. Returns object (Error). |
| 404 | No message with this id belongs to the application. Returns object (ErrorMessage). |
| 429 | Rate limited. Either the per-application request budget for the current minute is spent (`{"message": "rate_limited", "retry_after": <seconds>}` with a `Retry-After` header; every response carries `X-RateLimit-Remaining`), or - on the OTP routes - the per-phone OTP limit was reached (plain string, default 5 OTPs per hour per application). Wait and retry; do not tighten the loop. Returns object (Error). |

### 200 response

| Property | Type | Description |
|---|---|---|
| `id` | string (uuid) | Message ID. The same value is sent as `id` in the `callback` and `inbound` webhooks. |
| `type` | object (MessageType), one of: transactional, promotional | Message type; it follows the template/service configuration. Promotional messages always respect the blacklist. |
| `phone` | string | Recipient in E.164 format. |
| `message` | object (MessageContent) | Rendered content that was sent. |
| `message.text` | string, nullable | Rendered message text. |
| `message.imageUrl` | string (uri) | Image URL (Viber image/advanced layouts only). |
| `message.button` | object (MessageButton) |  |
| `message.button.label` | string |  |
| `message.button.url` | string (uri) |  |
| `price` | string | Price charged for the message as a decimal string with four decimals, in the account currency (EUR). Always `0.0000` on the sandbox. |
| `parts` | integer | Number of SMS parts (1 for Viber/Noti). |
| `channel` | object (Channel), one of: sms, viber, noti | Messaging channel. `noti` is the Connectix in-app notification channel. |
| `status` | object (MessageStatus), one of: pending, received, scheduled, processing, sent, delivered, seen, undelivered, failed, could_not_be_sent, not_used | Lifecycle: `received` (accepted by the API) -> `processing` -> `sent` -> `delivered` -> `seen` (Viber/Noti only). Terminal failures: `undelivered` (expired, not a Viber/Noti user, opted out), `failed`, `could_not_be_sent`. `scheduled` means the queue was unavailable and the message will be picked up by the scheduler. |
| `ttl` | integer, nullable | Effective delivery TTL in seconds. |
| `callbackUrl` | string, nullable | Status webhook URL attached to this message. |
| `inboundUrl` | string, nullable | Reply webhook URL. Present for Viber messages only. |
| `sender` | string, nullable | Sender ID used. Present for SMS messages only. |
| `createdAt` | string (date-time) | RFC 3339 timestamp. |
| `position` | integer | Step position. Present only for messages that belong to a fallback flow. |
| `fallbackId` | string (uuid) | ID of the fallback flow. Present only for messages that belong to a fallback flow. |
| `reference` | string, nullable | The `reference` the message was created with, or null. |
| `updatedAt` | string (date-time) | When the message last changed (status included). Present in `getMessage` and `listMessages` answers only. |

```json
{
    "id": "8b2f1c0e-5d3a-4e7b-9f21-0c6a7d4e5b12",
    "type": "transactional",
    "phone": "+359881234567",
    "message": {
        "text": "Your order 1234 was shipped."
    },
    "price": "0.0000",
    "parts": 1,
    "channel": "viber",
    "status": "delivered",
    "ttl": 7200,
    "callbackUrl": null,
    "reference": "order-1234-shipped",
    "createdAt": "2026-09-22T10:15:30+03:00",
    "inboundUrl": null,
    "updatedAt": "2026-09-22T10:15:41+03:00"
}
```

## Example request

```bash
curl -X GET "https://api-sandbox.connectix.bg/messages/3644d7ac-2a95-4068-863e-6352ed78d667" \
  -H "Authorization: $CONNECTIX_SANDBOX_TOKEN" \
  -H "Accept: application/json"
```
