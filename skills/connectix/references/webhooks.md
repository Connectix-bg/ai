# Webhooks reference

Connectix calls your HTTPS URL with a JSON `POST`. Reply `2xx` quickly; a non-2xx answer or a
timeout (5 s, 10 s for tracking) is logged and **not retried** - make the receiver idempotent and
fast, and reconcile via the API if a delivery is missed. Hooks are **unsigned** today (no HMAC
header) - treat the payload as a hint and confirm anything that matters with an API call
(`checkBlacklist`, `listTemplates`). The payload schemas are in the OpenAPI spec under `x-webhooks`
and via `get_guide` with slug `webhooks`.

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
{"id": "<message uuid>", "status": "delivered", "channel": "viber", "createdAt": "2026-09-20T10:15:30+03:00", "fallbackId": "<uuid, only for fallback flows>"}
```

`status` follows the message lifecycle: `processing`, `sent`, `delivered`, `seen`, `undelivered`
(also used for expired messages), `failed`. In a fallback flow every step shares the same
`fallbackId`; a `failed`/`undelivered` step 1 is followed by step 2.

### inbound - the recipient replied

```json
{"id": "<message uuid>", "phone": "+359888123456", "type": "text", "sentAt": "2026-09-20T10:15:30+03:00", "text": "STOP", "mediaUrl": null, "mediaName": null}
```

`type` is `text`, `image` or `file` (the sandbox simulator omits it - do not require it when
validating sandbox hooks). If blacklist handling is enabled for the application, an opt-out keyword
in `text` also creates a blacklist entry and fires the `blacklist` hook.

### blacklist - an entry was created or changed

```json
{"id": "<uuid>", "phone": "+359888123456", "active": true, "permanent": false, "global": false, "expire_at_for_transactional": "2026-09-27T10:15:30+03:00"}
```

`global` entries apply to every application of the company. `expire_at_for_transactional` is when
transactional messages may resume (`null` when permanent or when no validity is configured); this
field is snake_case, as is the payload of the `template` hook.

### template - a template's status changed (approval)

```json
{"id": "<uuid>", "media": null, "layout": "text", "name": "Order shipped", "channel": "viber", "text": "...", "button_label": null, "button_url": null, "status": "approved"}
```

Note: `listTemplates` returns the same fields in camelCase (`buttonLabel`, `buttonUrl`); the hook uses
snake_case.

### tracking - a shipment event

```json
{"action": "readyForDelivery", "trackingNumber": "1234567890", "deliveryType": "office", "office": "Sofia - Mladost 1"}
```

`action` is one of `dispatched`, `readyForDelivery` (with `deliveryType` and `courier` or `office`),
`notPickedUp` (with `workingDaysPassed` and `office`), `delivered`, `returned`, `returnedRefused`,
`returnedAfterCheck`, `returnedNotPickedUp`.

## Receiving hooks on the sandbox

Sandbox sends fire real `callback` hooks with fabricated statuses (see `sandbox.md`), so a public
URL (a tunnel is fine) is enough to test the receiver. Log the raw body, respond `200`, process
asynchronously.
