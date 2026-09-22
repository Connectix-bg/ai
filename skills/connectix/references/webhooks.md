# Webhooks reference

Connectix calls your HTTPS URL with a JSON `POST` (`User-Agent: connectix-client`). Reply `2xx`
within the timeout (5 s, 10 s for tracking) and process afterwards. A non-2xx answer or a timeout is
**retried per URL** with back-off: 30 s, 1, 2, 5, 10, 15, 30 min, 1, 2, 4, 8, 16 h (12 attempts,
about 32 hours), then the event is dropped for that URL. Delivery is at-least-once and unordered.
The payload schemas are in the OpenAPI spec under `x-webhooks` and via `get_guide` with slug
`webhooks`.

## Every event

- `eventId` (body) = `X-Connectix-Event-Id` (header): globally unique, the deduplication key.
- `reference` on `callback` and `inbound` events: the value the message was sent with, when any.
- Deliveries to **integration hooks** are signed; deliveries to a per-message `callbackUrl`/`inboundUrl`
  are not (no secret exists for them) - validate the `id` against your own records there.

## Verifying the signature (integration hooks)

Headers: `X-Connectix-Timestamp` (Unix seconds) and `X-Connectix-Signature` (`v1=` + hex
HMAC-SHA256 of `"<timestamp>.<raw body>"` keyed with the integration `secret` from
`registerIntegration`). Compute on the raw request bytes, compare in constant time, reject
timestamps older than 300 s.

```php
$expected = 'v1=' . hash_hmac('sha256', $timestamp . '.' . $rawBody, $secret);
$ok = ctype_digit($timestamp) && abs(time() - (int) $timestamp) <= 300 && hash_equals($expected, $signature);
```

```js
const expected = 'v1=' + crypto.createHmac('sha256', secret).update(`${timestamp}.${rawBody}`).digest('hex');
const ok = Math.abs(Date.now() / 1000 - Number(timestamp)) <= 300
  && expected.length === signature.length
  && crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signature));
```

```python
expected = 'v1=' + hmac.new(secret.encode(), f'{timestamp}.'.encode() + raw_body, hashlib.sha256).hexdigest()
ok = timestamp.isdigit() and abs(time.time() - int(timestamp)) <= 300 and hmac.compare_digest(expected, signature)
```

## Where a hook URL comes from

| Hook | Per message | Per integration (`registerIntegrationHook` with `type`) |
|---|---|---|
| callback | `callbackUrl` on `sendMessage`, `sendFallbackMessage`, `sendMediaMessage`, `sendOtp` | `callback` |
| inbound | `inboundUrl` on the message endpoints | `inbound` |
| blacklist | - | `blacklist` |
| template | - | `template` |
| tracking | - | `tracking` |

Register an integration first (`registerIntegration` with `type` one of `ecommerce`, `automation`,
`custom`, a `platform` name and a `secret` you choose; it returns the 64-character integration
`token`), then hooks with `registerIntegrationHook` (`token`, `type`, `url`). Remove with
`unregisterIntegrationHook` and `unregisterIntegration` (`token` + `secret`).

Messages created from campaigns in the console never trigger `callback` or `inbound` hooks.

## Payloads

### callback - message status changed

```json
{"eventId": "<uuid>", "id": "<message uuid>", "status": "delivered", "channel": "viber", "createdAt": "2026-09-20T10:15:30+03:00", "reference": "<your reference, when given>", "fallbackId": "<uuid, only for fallback flows>"}
```

`status` follows the message lifecycle: `processing`, `sent`, `delivered`, `seen`, `undelivered`
(also used for expired messages), `failed`. In a fallback flow every step shares the same
`fallbackId`; a `failed`/`undelivered` step 1 is followed by step 2.

### inbound - the recipient replied

```json
{"eventId": "<uuid>", "id": "<message uuid>", "phone": "+359888123456", "type": "text", "sentAt": "2026-09-20T10:15:30+03:00", "text": "STOP", "mediaUrl": null, "mediaName": null, "reference": "<your reference, when given>"}
```

`type` is `text`, `image` or `file` (the sandbox simulator omits it - do not require it when
validating sandbox hooks). If blacklist handling is enabled for the application, an opt-out keyword
in `text` also creates a blacklist entry and fires the `blacklist` hook.

### blacklist - an entry was created or changed

```json
{"eventId": "<uuid>", "id": "<uuid>", "phone": "+359888123456", "active": true, "permanent": false, "global": false, "expire_at_for_transactional": "2026-09-27T10:15:30+03:00"}
```

`global` entries apply to every application of the company. `expire_at_for_transactional` is when
transactional messages may resume (`null` when permanent or when no validity is configured); this
field is snake_case, as is the payload of the `template` hook.

### template - a template's status changed (approval)

```json
{"eventId": "<uuid>", "id": "<uuid>", "media": null, "layout": "text", "name": "Order shipped", "channel": "viber", "text": "...", "button_label": null, "button_url": null, "status": "approved"}
```

Note: `listTemplates` returns the same fields in camelCase (`buttonLabel`, `buttonUrl`); the hook uses
snake_case.

### tracking - a shipment event

```json
{"eventId": "<uuid>", "action": "readyForDelivery", "trackingNumber": "1234567890", "deliveryType": "office", "office": "Sofia - Mladost 1"}
```

`action` is one of `dispatched`, `readyForDelivery` (with `deliveryType` and `courier` or `office`),
`notPickedUp` (with `workingDaysPassed` and `office`), `delivered`, `returned`, `returnedRefused`,
`returnedAfterCheck`, `returnedNotPickedUp`.

## Receiving hooks on the sandbox

Sandbox sends fire real `callback` hooks with fabricated statuses (see `sandbox.md`), so a public
URL (a tunnel is fine) is enough to test the receiver. Log the raw body, respond `200`, process
asynchronously.
