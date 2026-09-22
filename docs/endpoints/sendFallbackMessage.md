# Send a message with fallback steps

`POST /messages/fallback`

- Operation: `sendFallbackMessage`
- Tags: Messages
- Product: sms
- Base URL: https://api-sandbox.connectix.bg

Sends a multi-step flow to one recipient: the first step is sent immediately and each following step is sent only if the previous one ends as `undelivered` (for example Viber first, SMS if the recipient is not on Viber). Steps are ordered by `position` and the TTL of a step bounds how long the next step waits. Every step message carries the same `fallbackId` in its `callback` webhooks.

Phone numbers must be in E.164 format. Authenticate with the application token in the `Authorization` header: on `https://api-sandbox.connectix.bg` use the application's **sandbox token**; the production host uses the production token of the same application.

**402** means the account balance cannot cover the message price (`Insufficient funds.`) - top up the account before retrying. **451** means the company has not signed the contract for the channel the template uses (`There is no signed contract for channel "viber"`) - a human must sign it in the console; retrying does not help.

## Authentication

Send the token in the `Authorization` header. Application token sent as the raw value of the `Authorization` header (`Authorization: <token>`); `Bearer <token>` is also accepted. Use the sandbox token on `https://api-sandbox.connectix.bg`. Use the sandbox token from the console; the raw token and `Bearer <token>` are both accepted.

## Sandbox

Sandbox host accepts the same payload and delivers nothing: delivery is simulated (each message ends as `delivered` or `undelivered` at random), price is always `0.0000`, balance and contract checks are skipped, a status webhook is posted to `callbackUrl` about 5 seconds later and a simulated inbound reply is posted to `inboundUrl` about 15 seconds after a delivered Viber message. The sandbox only ever calls `callbackUrl`/`inboundUrl`, never the integration's hooks.

## Parameters

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `integration` | query | no | string | Integration token obtained from `registerIntegration`. When present, the message is attributed to that integration and its active `callback`/`inbound` hooks receive the status and reply webhooks - but only when the body has no `callbackUrl`/`inboundUrl`: a per-message URL always wins and the integration hooks are then not called. |

## Request body

Content type: `application/json`. Required.

| Property | Type | Required | Description |
|---|---|---|---|
| `phone` | string | yes | Recipient mobile number. Send it in E.164 format (`+359881234567`). National formats are parsed against the countries enabled for the application, but E.164 is the only format that is guaranteed to work. Landline numbers are rejected with 422. |
| `contact` | object (Contact) | no | Optional contact record that is created or updated in the address book when the message is accepted. `firstName` and `lastName` are also injected into the template parameters as `first_name`/`last_name`. |
| `contact.firstName` | string | yes | Given name. Required when `contact` is present. |
| `contact.lastName` | string | no | Family name. |
| `contact.groups` | array of string | no | Contact group names the contact is added to. |
| `contact.addGroupIfMissing` | boolean | no | Create groups from `groups` that do not exist yet. |
| `contact.parameters` | object | no | Free-form custom fields stored on the contact. |
| `flow` | array of object (FallbackStep) | yes | At least two steps, typically Viber first and SMS second. |
| `flow[].template` | string (uuid) | yes | UUID of an approved template. |
| `flow[].position` | integer | yes | Order of the step, starting at 1. |
| `flow[].parameters` | object | no | Template parameters for this step. |
| `flow[].ttl` | integer | no | Seconds the provider may keep trying to deliver the message. Defaults to 7200 for Viber/Noti and 43200 for SMS. In a fallback flow the TTL of a step bounds how long the next step waits. |
| `flow[].callbackUrl` | string (uri) | no | HTTP(S) URL with a public TLD (HTTPS recommended) that receives the `callback` (delivery status) webhook for this message. |
| `flow[].inboundUrl` | string (uri) | no | HTTP(S) URL with a public TLD (HTTPS recommended) that receives the `inbound` (reply) webhook for this message. Only Viber and Noti recipients can reply. |

Example:

```json
{
    "phone": "+359881234567",
    "flow": [
        {
            "template": "3644d7ac-2a95-4068-863e-6352ed78d667",
            "position": 1,
            "parameters": {
                "order_id": "10422"
            },
            "ttl": 3600
        },
        {
            "template": "cdea4406-ae6c-4cbc-9571-40882795d17e",
            "position": 2,
            "parameters": {
                "order_id": "10422"
            }
        }
    ]
}
```

## Responses

| Status | Description |
|---|---|
| 200 | Flow accepted; contains the first step message. Returns object (FallbackResponse). |
| 400 | Invalid request. Either a required value is missing, the template is not approved / not found, the text exceeds the channel limit, or the body is not valid JSON (`{"error": "Invalid JSON: Syntax error"}`). Returns object (Error). |
| 401 | Missing or unknown token for this host, or the company is suspended. Check that you use the sandbox token on the sandbox host. Returns object (Error). |
| 402 | Insufficient funds: the balance does not cover the price of the message. Top up the account before retrying. Returns object (Error). |
| 403 | Access to this resource is disabled for the application (`access_restricted`), the TrustCheck eligibility rules are not met, or - on the production host only - an AI coding tool called a send/OTP route (`ai_tools_must_use_sandbox`: develop against the sandbox, going live is a human step). Returns object (Error). |
| 406 | Traffic is suspended for the company, or no sender ID is configured for the recipient's country. Returns object (Error). |
| 422 | Validation failed. The body maps each offending field path to a message. Unknown fields are rejected. Returns object (Error). |
| 429 | Rate limited. Either the per-application request budget for the current minute is spent (`{"message": "rate_limited", "retry_after": <seconds>}` with a `Retry-After` header; every response carries `X-RateLimit-Remaining`), or - on the OTP routes - the per-phone OTP limit was reached (plain string, default 5 OTPs per hour per application). Wait and retry; do not tighten the loop. Returns object (Error). |
| 451 | The company has not signed the contract for the channel of the template. A human must sign it in the console (Documents); retrying does not help. Returns object (Error). |
| 502 | The request could not be processed: the phone number could not be parsed, the recipient's country is not enabled for the application, no provider is available, or a storage error occurred. Returns object (Error). |

### 200 response

| Property | Type | Description |
|---|---|---|
| `id` | string (uuid) | Fallback flow ID (equals `fallbackId` in the step messages and webhooks). |
| `flow` | array of object (Message) | Messages created so far (the first step). |
| `flow[].id` | string (uuid) | Message ID. The same value is sent as `id` in the `callback` and `inbound` webhooks. |
| `flow[].type` | object (MessageType), one of: transactional, promotional | Message type; it follows the template/service configuration. Promotional messages always respect the blacklist. |
| `flow[].phone` | string | Recipient in E.164 format. |
| `flow[].message` | object (MessageContent) | Rendered content that was sent. |
| `flow[].message.text` | string, nullable | Rendered message text. |
| `flow[].message.imageUrl` | string (uri) | Image URL (Viber image/advanced layouts only). |
| `flow[].message.button` | object (MessageButton) |  |
| `flow[].price` | string | Price charged for the message as a decimal string with four decimals, in the account currency (EUR). Always `0.0000` on the sandbox. |
| `flow[].parts` | integer | Number of SMS parts (1 for Viber/Noti). |
| `flow[].channel` | object (Channel), one of: sms, viber, noti | Messaging channel. `noti` is the Connectix in-app notification channel. |
| `flow[].status` | object (MessageStatus), one of: pending, received, scheduled, processing, sent, delivered, seen, undelivered, failed, could_not_be_sent, not_used | Lifecycle: `received` (accepted by the API) -> `processing` -> `sent` -> `delivered` -> `seen` (Viber/Noti only). Terminal failures: `undelivered` (expired, not a Viber/Noti user, opted out), `failed`, `could_not_be_sent`. `scheduled` means the queue was unavailable and the message will be picked up by the scheduler. |
| `flow[].ttl` | integer, nullable | Effective delivery TTL in seconds. |
| `flow[].callbackUrl` | string, nullable | Status webhook URL attached to this message. |
| `flow[].inboundUrl` | string, nullable | Reply webhook URL. Present for Viber messages only. |
| `flow[].sender` | string, nullable | Sender ID used. Present for SMS messages only. |
| `flow[].createdAt` | string (date-time) | RFC 3339 timestamp. |
| `flow[].position` | integer | Step position. Present only for messages that belong to a fallback flow. |
| `flow[].fallbackId` | string (uuid) | ID of the fallback flow. Present only for messages that belong to a fallback flow. |
| `createdAt` | string (date-time) |  |

```json
{
    "id": "a1c3e5f7-9b2d-4f6e-8a0c-1d3e5f7a9b2c",
    "flow": [
        {
            "id": "8b2f1c0e-5d3a-4e7b-9f21-0c6a7d4e5b12",
            "type": "transactional",
            "phone": "+359881234567",
            "message": {
                "text": "Your order #10422 has been shipped.",
                "button": {
                    "label": "Track order",
                    "url": "https://shop.example.com/orders/10422"
                }
            },
            "price": "0.0550",
            "parts": 1,
            "channel": "viber",
            "status": "received",
            "ttl": 7200,
            "callbackUrl": "https://shop.example.com/webhooks/connectix/status",
            "inboundUrl": "https://shop.example.com/webhooks/connectix/inbound",
            "createdAt": "2026-09-20T10:15:30+03:00",
            "position": 1,
            "fallbackId": "a1c3e5f7-9b2d-4f6e-8a0c-1d3e5f7a9b2c"
        }
    ],
    "createdAt": "2026-09-20T10:15:30+03:00"
}
```

## Example request

```bash
curl -X POST "https://api-sandbox.connectix.bg/messages/fallback?integration=npu5K6DCMcsmLyDjLNpSkc8szq8iUSlY6Ga96vJjxsKOYWiAILbvQp9qXkKKGIfl" \
  -H "Authorization: $CONNECTIX_SANDBOX_TOKEN" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"phone":"+359881234567","flow":[{"template":"3644d7ac-2a95-4068-863e-6352ed78d667","position":1,"parameters":{"order_id":"10422"},"ttl":3600},{"template":"cdea4406-ae6c-4cbc-9571-40882795d17e","position":2,"parameters":{"order_id":"10422"}}]}'
```
