# List messages

`GET /messages`

- Operation: `listMessages`
- Tags: Messages
- Product: sms
- Base URL: https://api-sandbox.connectix.bg

Returns the application's messages created inside a time window, newest first, in pages. Use it to reconcile your records when a `callback` webhook was missed, or to find a message by `reference`.

The window is `[from, to]` on `createdAt`, at most 7 days wide; without `from`/`to` it is the last 24 hours. Pages are cursor-based: repeat the same query with the `next` value from the previous answer until it is `null`. Items are the same shape as the `sendMessage` answer plus `updatedAt`.

Authenticate with the application token in the `Authorization` header: on `https://api-sandbox.connectix.bg` use the application's **sandbox token**; the production host uses the production token of the same application.

## Authentication

Send the token in the `Authorization` header. Application token sent as the raw value of the `Authorization` header (`Authorization: <token>`); `Bearer <token>` is also accepted. Use the sandbox token on `https://api-sandbox.connectix.bg`. Use the sandbox token from the console; the raw token and `Bearer <token>` are both accepted.

## Sandbox

Lists the sandbox messages of the application (the ones sent to the sandbox host), with the simulated statuses.

## Parameters

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `from` | query | no | string | Start of the window (inclusive): an RFC 3339 date-time or a `YYYY-MM-DD` date. Default: 24 hours before `to`. |
| `to` | query | no | string | End of the window (inclusive): an RFC 3339 date-time or a `YYYY-MM-DD` date. Default: now. |
| `status` | query | no | object (MessageStatus), one of: pending, received, scheduled, processing, sent, delivered, seen, undelivered, failed, could_not_be_sent, not_used | Only messages currently in this status. |
| `channel` | query | no | object (Channel), one of: sms, viber, noti | Only messages sent through this channel. |
| `reference` | query | no | string | Only messages created with exactly this `reference`. |
| `limit` | query | no | integer | Page size. |
| `cursor` | query | no | string | The `next` value of the previous page. The other query parameters must stay the same. |

## Responses

| Status | Description |
|---|---|
| 200 | A page of messages. Returns object (MessagePage). |
| 400 | A query parameter is invalid: unknown status or channel, a window wider than 7 days, `from` after `to`, a bad date, limit outside 1-100, or a cursor that was not issued by this endpoint. The body is the reason as a string. Returns object (ErrorMessage). |
| 401 | Missing or unknown token for this host, or the company is suspended. Check that you use the sandbox token on the sandbox host. Returns object (Error). |
| 429 | Rate limited. Either the per-application request budget for the current minute is spent (`{"message": "rate_limited", "retry_after": <seconds>}` with a `Retry-After` header; every response carries `X-RateLimit-Remaining`), or - on the OTP routes - the per-phone OTP limit was reached (plain string, default 5 OTPs per hour per application). Wait and retry; do not tighten the loop. Returns object (Error). |

### 200 response

| Property | Type | Description |
|---|---|---|
| `items` | array of object (Message) |  |
| `items[].id` | string (uuid) | Message ID. The same value is sent as `id` in the `callback` and `inbound` webhooks. |
| `items[].type` | object (MessageType), one of: transactional, promotional | Message type; it follows the template/service configuration. Promotional messages always respect the blacklist. |
| `items[].phone` | string | Recipient in E.164 format. |
| `items[].message` | object (MessageContent) | Rendered content that was sent. |
| `items[].message.text` | string, nullable | Rendered message text. |
| `items[].message.imageUrl` | string (uri) | Image URL (Viber image/advanced layouts only). |
| `items[].message.button` | object (MessageButton) |  |
| `items[].price` | string | Price charged for the message as a decimal string with four decimals, in the account currency (EUR). Always `0.0000` on the sandbox. |
| `items[].parts` | integer | Number of SMS parts (1 for Viber/Noti). |
| `items[].channel` | object (Channel), one of: sms, viber, noti | Messaging channel. `noti` is the Connectix in-app notification channel. |
| `items[].status` | object (MessageStatus), one of: pending, received, scheduled, processing, sent, delivered, seen, undelivered, failed, could_not_be_sent, not_used | Lifecycle: `received` (accepted by the API) -> `processing` -> `sent` -> `delivered` -> `seen` (Viber/Noti only). Terminal failures: `undelivered` (expired, not a Viber/Noti user, opted out), `failed`, `could_not_be_sent`. `scheduled` means the queue was unavailable and the message will be picked up by the scheduler. |
| `items[].ttl` | integer, nullable | Effective delivery TTL in seconds. |
| `items[].callbackUrl` | string, nullable | Status webhook URL attached to this message. |
| `items[].inboundUrl` | string, nullable | Reply webhook URL. Present for Viber messages only. |
| `items[].sender` | string, nullable | Sender ID used. Present for SMS messages only. |
| `items[].createdAt` | string (date-time) | RFC 3339 timestamp. |
| `items[].position` | integer | Step position. Present only for messages that belong to a fallback flow. |
| `items[].fallbackId` | string (uuid) | ID of the fallback flow. Present only for messages that belong to a fallback flow. |
| `items[].reference` | string, nullable | The `reference` the message was created with, or null. |
| `items[].updatedAt` | string (date-time) | When the message last changed (status included). Present in `getMessage` and `listMessages` answers only. |
| `next` | string, nullable | Opaque cursor of the next page; pass it as `cursor` with the same filters. `null` on the last page. |

```json
{
    "items": [
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
    ],
    "next": null
}
```

## Example request

```bash
curl -X GET "https://api-sandbox.connectix.bg/messages?limit=50" \
  -H "Authorization: $CONNECTIX_SANDBOX_TOKEN" \
  -H "Accept: application/json"
```
