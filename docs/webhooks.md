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
  "id": "0d1c1c2e-8f2a-4c1e-9c2b-2d1a6d1f7e21",
  "status": "delivered",
  "channel": "viber",
  "createdAt": "2026-01-15T10:30:12+02:00",
  "fallbackId": "7c2f0b1e-5a6d-4f3e-8b9a-1c2d3e4f5a6b"
}
```

`fallbackId` is present only for messages sent through `/messages/fallback`. Statuses: `pending`, `processing`, `received`, `scheduled`, `sent`, `delivered`, `seen`, `undelivered`, `failed`, `could_not_be_sent`, `fallback_error`.

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

## Receiving hooks safely

- Hooks are **unsigned** JSON POSTs today. Do not trust a payload by itself: look the `id` up in your own records (the message you sent, the hook you registered) before acting on it. A signature header is on the roadmap.
- Answer `2xx` within a few seconds and do the work asynchronously. A non-2xx answer or a timeout (5 s, 10 s for tracking hooks) is logged and **not retried**: if a delivery is missed, reconcile through the API (`checkBlacklist`, `listTemplates`, your own message records).
- Expect duplicates and out-of-order delivery; make handlers idempotent on `id` + `status`.
- Use HTTPS URLs with a valid certificate; `http://` is accepted but not recommended.
- In the sandbox the statuses are simulated, so test all transitions there before going live.

