# Send a message from a template

`POST /messages`

- Operation: `sendMessage`
- Tags: Messages
- Product: sms
- Base URL: https://api-sandbox.connectix.bg

Sends one SMS, Viber or Noti message to one recipient using an approved template. The response is returned as soon as the message is accepted (`status: received`); delivery happens asynchronously and is reported through the `callback` webhook (`callbackUrl` or integration hooks). Replies arrive on the `inbound` webhook.

Phone numbers must be in E.164 format. Authenticate with the application token in the `Authorization` header: on `https://api-sandbox.connectix.bg` use the application's **sandbox token**; the production host uses the production token of the same application.

**402** means the account balance cannot cover the message price (`Insufficient funds.`) - top up the account before retrying. **451** means the company has not signed the contract for the channel the template uses (`There is no signed contract for channel "viber"`) - a human must sign it in the console; retrying does not help.

Send an `Idempotency-Key` header (a fresh UUID per logical send) so a network timeout can be retried without a second message; the repeat returns the stored answer with `Idempotent-Replayed: true`.

## Authentication

Send the token in the `Authorization` header. Application token sent as the raw value of the `Authorization` header (`Authorization: <token>`); `Bearer <token>` is also accepted. Use the sandbox token on `https://api-sandbox.connectix.bg`. Use the sandbox token from the console; the raw token and `Bearer <token>` are both accepted.

## Sandbox

Sandbox host accepts the same payload and delivers nothing: delivery is simulated (each message ends as `delivered` or `undelivered` at random), price is always `0.0000`, balance and contract checks are skipped, a status webhook is posted to `callbackUrl` about 5 seconds later and a simulated inbound reply is posted to `inboundUrl` about 15 seconds after a delivered Viber message. The sandbox only ever calls `callbackUrl`/`inboundUrl`, never the integration's hooks.

## Parameters

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `integration` | query | no | string | Integration token obtained from `registerIntegration`. When present, the message is attributed to that integration and its active `callback`/`inbound` hooks receive the status and reply webhooks - but only when the body has no `callbackUrl`/`inboundUrl`: a per-message URL always wins and the integration hooks are then not called. |
| `Idempotency-Key` | header | no | string | Makes a retry of this request safe. The first request with a key runs normally and its 2xx answer is stored for 24 hours; every repeat of the same key from the same application on the same host returns that stored answer unchanged, with `Idempotent-Replayed: true`, and nothing is sent again. A repeat with a different body, path or host answers **422**; a repeat while the first request is still running answers **409** `in_progress` (retry after `retry_after` seconds). Non-2xx answers are not stored, so the same key can be retried after the request is fixed. Use a fresh UUID per logical send. |

## Request body

Content type: `application/json`. Required.

| Property | Type | Required | Description |
|---|---|---|---|
| `phone` | string | yes | Recipient mobile number. Send it in E.164 format (`+359881234567`). National formats are parsed against the countries enabled for the application, but E.164 is the only format that is guaranteed to work. Landline numbers are rejected with 422. |
| `template` | string (uuid) | yes | UUID of an **approved** template from `listTemplates`. The template decides the channel and the message type. |
| `parameters` | object | no | Values for the `{{ placeholder }}` variables used in the template text, button label and button URL. |
| `ttl` | integer | no | Seconds the provider may keep trying to deliver the message. Defaults to 7200 for Viber/Noti and 43200 for SMS. In a fallback flow the TTL of a step bounds how long the next step waits. |
| `callbackUrl` | string (uri) | no | HTTP(S) URL with a public TLD (HTTPS recommended) that receives the `callback` (delivery status) webhook for this message. Must be publicly reachable: hosts that resolve to private, loopback, link-local or metadata addresses are refused with 422 (`The URL must be a public http(s) address.`). |
| `inboundUrl` | string (uri) | no | HTTP(S) URL with a public TLD (HTTPS recommended) that receives the `inbound` (reply) webhook for this message. Only Viber and Noti recipients can reply. Must be publicly reachable: hosts that resolve to private, loopback, link-local or metadata addresses are refused with 422 (`The URL must be a public http(s) address.`). |
| `contact` | object (Contact) | no | Optional contact record that is created or updated in the address book when the message is accepted. `firstName` and `lastName` are also injected into the template parameters as `first_name`/`last_name`. |
| `contact.firstName` | string | yes | Given name. Required when `contact` is present. |
| `contact.lastName` | string | no | Family name. |
| `contact.groups` | array of string | no | Contact group names the contact is added to. |
| `contact.addGroupIfMissing` | boolean | no | Create groups from `groups` that do not exist yet. |
| `contact.parameters` | object | no | Free-form custom fields stored on the contact. |
| `reference` | string | no | Your own reference for this message (an order id, an event id...). Echoed as `reference` in the answer and in every `callback`/`inbound` webhook of the message, so you can match them without keeping a mapping table. Free to repeat across messages; use the `Idempotency-Key` header, not the reference, to make a retry safe. |

Example:

```json
{
    "phone": "+359881234567",
    "template": "3644d7ac-2a95-4068-863e-6352ed78d667",
    "parameters": {
        "order_id": "10422"
    },
    "callbackUrl": "https://shop.example.com/webhooks/connectix/status",
    "inboundUrl": "https://shop.example.com/webhooks/connectix/inbound",
    "contact": {
        "firstName": "Ivan",
        "lastName": "Ivanov",
        "groups": [
            "Customers"
        ],
        "addGroupIfMissing": true,
        "parameters": {
            "customerId": "10422"
        }
    }
}
```

## Responses

| Status | Description |
|---|---|
| 200 | Message accepted. Returns object (Message). |
| 400 | Invalid request. Either a required value is missing, the template is not approved / not found, the text exceeds the channel limit, or the body is not valid JSON (`{"error": "Invalid JSON: Syntax error"}`). Returns object (Error). |
| 401 | Missing or unknown token for this host, or the company is suspended. Check that you use the sandbox token on the sandbox host. Returns object (Error). |
| 402 | Insufficient funds: the balance does not cover the price of the message. Top up the account before retrying. Returns object (Error). |
| 403 | Access to this resource is disabled for the application (`access_restricted`), the TrustCheck eligibility rules are not met, or - on the production host only - an AI coding tool called a send/OTP route (`ai_tools_must_use_sandbox`: develop against the sandbox, going live is a human step). Returns object (Error). |
| 406 | Traffic is suspended for the company, or no sender ID is configured for the recipient's country. Returns object (Error). |
| 409 | The same `Idempotency-Key` is still being processed by another request. Returns object (InProgressError). |
| 422 | Validation failed. The body maps each offending field path to a message. Unknown fields are rejected. Also answered when an `Idempotency-Key` is reused with a different request (`[idempotencyKey]`). Returns object (Error). |
| 429 | Rate limited. Either the per-application request budget for the current minute is spent (`{"message": "rate_limited", "retry_after": <seconds>}` with a `Retry-After` header; every response carries `X-RateLimit-Remaining`), or - on the OTP routes - the per-phone OTP limit was reached (plain string, default 5 OTPs per hour per application). Wait and retry; do not tighten the loop. Returns object (Error). |
| 451 | The company has not signed the contract for the channel of the template. A human must sign it in the console (Documents); retrying does not help. Returns object (Error). |
| 502 | The request could not be processed: the phone number could not be parsed, the recipient's country is not enabled for the application, no provider is available, or a storage error occurred. Returns object (Error). |

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
    "reference": null
}
```

## Example request

```bash
curl -X POST "https://api-sandbox.connectix.bg/messages?integration=npu5K6DCMcsmLyDjLNpSkc8szq8iUSlY6Ga96vJjxsKOYWiAILbvQp9qXkKKGIfl" \
  -H "Authorization: $CONNECTIX_SANDBOX_TOKEN" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"phone":"+359881234567","template":"3644d7ac-2a95-4068-863e-6352ed78d667","parameters":{"order_id":"10422"},"callbackUrl":"https://shop.example.com/webhooks/connectix/status","inboundUrl":"https://shop.example.com/webhooks/connectix/inbound","contact":{"firstName":"Ivan","lastName":"Ivanov","groups":["Customers"],"addGroupIfMissing":true,"parameters":{"customerId":"10422"}}}'
```
