# Webhooks

# Webhooks

Connectix notifies your system with plain JSON `POST` requests: message status changes, inbound replies, blacklist changes, template changes and shipment tracking events. Two ways to receive them:

- **Per message**: pass `callbackUrl` (status changes) and `inboundUrl` (Viber replies) in `POST /messages` or `/messages/fallback`.
- **Per integration**: register an integration once, then register one hook URL per event type. Hooks receive every event of that type for the integration.

Both work in the sandbox; the sandbox sends the simulated statuses.

## Registering hooks

1. Register the integration (once), with a 32-character secret you generate and keep:

```bash
curl -X POST "https://api-sandbox.connectix.bg/integration/register" \
  -H "Authorization: $CONNECTIX_SANDBOX_TOKEN" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"type": "ecommerce", "platform": "MyShop", "secret": "VWAIKyRhzVmGkgDXTxepB1rO49CYKo3C"}'
```

```json
{"name": "My shop", "token": "npu5K6DCMcsmLyDjLNpSkc8szq8iUSlY6Ga96vJjxsKOYWiAILbvQp9qXkKKGIfl"}
```

2. Register a hook per event type with the integration `token`:

```bash
curl -X POST "https://api-sandbox.connectix.bg/integration/register/hook" \
  -H "Authorization: $CONNECTIX_SANDBOX_TOKEN" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"token": "npu5K6DCMcsmLyDjLNpSkc8szq8iUSlY6Ga96vJjxsKOYWiAILbvQp9qXkKKGIfl", "type": "callback", "url": "https://example.com/connectix/callback"}'
```

```json
{"id": "f4212b48-a763-42b8-8e43-0ed413255edd"}
```

Remove a hook with `POST /integration/unregister/hook` and `{"id": "..."}`; remove the whole integration with `POST /integration/unregister` and the token + secret.

## Event types and payloads

Hook `type` values: `callback`, `inbound`, `blacklist`, `template`, `tracking`.

### callback — message status

Sent on every status change of a message (not for campaign messages).

```json
{
  "eventId": "c1f4b7a2-3d5e-4f60-8a9b-1c2d3e4f5a6b",
  "id": "0d1c1c2e-8f2a-4c1e-9c2b-2d1a6d1f7e21",
  "status": "delivered",
  "channel": "viber",
  "createdAt": "2026-01-15T10:30:12+02:00",
  "reference": "order-10042-shipped",
  "fallbackId": "7c2f0b1e-5a6d-4f3e-8b9a-1c2d3e4f5a6b"
}
```

`eventId` is unique per delivery attempt group (see below). `reference` is the value you sent with the message and is present only then. `fallbackId` is present only for messages sent through `/messages/fallback`. Statuses: `pending`, `processing`, `received`, `scheduled`, `sent`, `delivered`, `seen`, `undelivered`, `failed`, `could_not_be_sent`, `fallback_error`.

### inbound — a reply from the recipient (Viber)

```json
{
  "id": "5f0a2b3c-1d2e-4f5a-9b8c-7d6e5f4a3b2c",
  "phone": "+359888123456",
  "sentAt": "2026-01-15T10:35:00+02:00",
  "text": "Thanks!",
  "mediaUrl": null,
  "mediaName": null
}
```

### blacklist — a number was added, changed or removed

```json
{
  "id": "f4212b48-a763-42b8-8e43-0ed413255edd",
  "phone": "+359888123456",
  "active": true,
  "permanent": false,
  "global": false,
  "expire_at_for_transactional": "2026-01-20T10:30:00+02:00"
}
```

Note the snake_case `expire_at_for_transactional`; it is `null` for permanent entries.

### template — a template was created, changed or approved

The template payload uses snake_case for the button fields (`button_label`, `button_url`), unlike `GET /templates`, which returns `buttonLabel` and `buttonUrl`. Read both when you sync templates.

```json
{
  "id": "3644d7ac-2a95-4068-863e-6352ed78d667",
  "name": "Order shipped",
  "channel": "viber",
  "layout": "button",
  "text": "Your order {{ orderNumber }} has shipped.",
  "button_label": "Track it",
  "button_url": "https://example.com/track",
  "status": "approved"
}
```

### tracking — a shipment event

```json
{
  "action": "delivered",
  "trackingNumber": "1234567890",
  "deliveryType": "office",
  "office": "Sofia - Mladost 1",
  "workingDaysPassed": 2
}
```

`deliveryType`, `courier`/`office` and `workingDaysPassed` are present when the courier reports them.

## Every event: `eventId` and headers

Every delivery carries a globally unique `eventId` in the body and the same value in the `X-Connectix-Event-Id` header. Deliveries are **at-least-once**: a receiver that timed out after processing gets the same event again, so drop what you have already seen by `eventId`. The order of events is not guaranteed; order by the event's own timestamp (`createdAt`, `sentAt`).

## Signature

Deliveries to an **integration hook** are signed with the integration `secret` you chose at `POST /integration/register`. Two headers come with each request:

| Header | Value |
|---|---|
| `X-Connectix-Timestamp` | Unix time (seconds) of the delivery |
| `X-Connectix-Signature` | `v1=` + hex HMAC-SHA256 of `"<timestamp>.<raw body>"` keyed with the secret |

Verify before you parse the JSON, on the raw bytes of the body:

```php
$timestamp = $_SERVER['HTTP_X_CONNECTIX_TIMESTAMP'] ?? '';
$signature = $_SERVER['HTTP_X_CONNECTIX_SIGNATURE'] ?? '';
$body = file_get_contents('php://input');

$expected = 'v1=' . hash_hmac('sha256', $timestamp . '.' . $body, $secret);
if (!ctype_digit($timestamp) || abs(time() - (int) $timestamp) > 300 || !hash_equals($expected, $signature)) {
    http_response_code(401);
    exit;
}
```

```js
const crypto = require('node:crypto');
const expected = 'v1=' + crypto.createHmac('sha256', secret).update(`${timestamp}.${rawBody}`).digest('hex');
const fresh = Math.abs(Date.now() / 1000 - Number(timestamp)) <= 300;
const valid = fresh && expected.length === signature.length && crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signature));
```

Reject timestamps older than 5 minutes (replay window). Deliveries to a per-message `callbackUrl`/`inboundUrl` have no secret and are **not signed**: keep those URLs hard to guess and validate the `id` against your own records.

## Retries

Answer any `2xx` within the timeout (5 s, 10 s for tracking events) and do the work afterwards. A non-2xx answer (429 included) or a timeout is retried for that URL with growing delays: 30 s, 1 min, 2, 5, 10, 15, 30 min, 1, 2, 4, 8 and 16 hours. That is 12 more attempts over about 32 hours; after the last one the event is dropped for that URL and the hook stays active. Redirects are not followed. Each attempt is recorded and visible in the console under API usage.

If you missed an event anyway, `GET /messages/{id}` and `GET /messages` read the current state of your messages back.

## Receiving hooks safely

- Verify the signature on integration hooks; validate the `id` against your own records on per-message URLs.
- Deduplicate by `eventId` (or by `id` + `status` for `callback` events) and do not rely on the order of events.
- Use HTTPS URLs with a valid certificate; `http://` is accepted but not recommended. The URL must be publicly reachable: hosts that resolve to private, loopback, link-local or cloud-metadata addresses are refused with 422 when you register them.
- In the sandbox the statuses are simulated, so test all transitions there before going live.

