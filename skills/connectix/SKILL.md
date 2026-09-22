---
name: connectix
description: Send Viber, SMS and Noti messages, one-time passwords (OTP), order and shipment notifications, TrustCheck shopper checks and blacklist look-ups through the Connectix API (Bulgaria). Use when the user mentions Connectix, Viber, SMS, Noti, OTP, TrustCheck, blacklist, order notification, delivery notification, or in Bulgarian изпрати Viber, SMS известие, код за потвърждение, известие за поръчка, Connectix API, Конектикс. Sandbox only - all code targets https://api-sandbox.connectix.bg.
license: MIT
metadata:
  author: Connectix
  docs: https://docs.connectix.bg
  mcp: https://mcp.connectix.bg/mcp
---

# Connectix API skill

## 1. Sandbox only (read this first)

Every request you write, run or suggest goes to **`https://api-sandbox.connectix.bg`** with the
**sandbox token**. No exceptions.

- Never write the production host in code, config, examples, tests or documentation. The only host
  you know is `https://api-sandbox.connectix.bg`.
- Never read, print, copy or ask for a live token. If you find one in a file or environment variable,
  do not use it and tell the user it should not be reachable from an AI session.
- The sandbox accepts the sandbox token only; a live token is rejected there, and a sandbox token is
  rejected on the live host. Nothing you send from the sandbox is delivered or billed.
- When the user says "go live", "switch to production", "use the real token" or similar: **stop**,
  do not change any host or token, and point them to the human-only go-live guide on
  https://docs.connectix.bg/en/go-live. A human replaces the host and the token there.

Details and the fabricated delivery statuses are in [references/sandbox.md](references/sandbox.md).

## 2. Authentication and headers

```
Authorization: <sandbox token>        # raw token, or "Bearer <sandbox token>"
Accept: application/json              # without it, errors may come back as HTML
Content-Type: application/json        # every POST body is JSON
```

Keep the token in an environment variable (`CONNECTIX_SANDBOX_TOKEN`), never in source. Start with
`GET /me` (`getMe`) - it returns `{"name": "<application name>"}` and proves the token, host and
headers are right before you write anything else.

## 3. Products and endpoints

| Product | Endpoints (operationId) | Notes |
|---|---|---|
| Templates | `listTemplates` GET /templates | Every message needs an **approved** template id; list them first |
| Messages | `sendMessage` POST /messages, `sendFallbackMessage` POST /messages/fallback, `sendMediaMessage` POST /messages/media | Channels: `sms`, `viber`, `noti` (the channel comes from the template) |
| OTP | `sendOtp` POST /otp/send, `verifyOtp` POST /otp/verify, `resendOtp` POST /otp/resend | One-time passwords, rate limited per phone |
| Integrations | `registerIntegration`, `unregisterIntegration`, `registerIntegrationHook`, `unregisterIntegrationHook` | Register a shop/platform and its webhook URLs |
| Blacklist | `checkBlacklist` POST /blacklist/check, `toggleBlacklist` POST /blacklist/toggle | `addToBlacklist` POST /blacklist is a deprecated alias of `checkBlacklist` (sunset 2027-03-31) - use `checkBlacklist` |
| TrustCheck | `checkShopper` POST /shopper/check | Shopper reliability score by phone |
| Tracking | `createShipment` POST /shipments | `createTracking` POST /trackings is deprecated - use `createShipment` |
| Account | `getMe` GET /me, `listHolidays` GET /countries/holidays?country=bg | Whoami and public holidays (BG only) |

Use `list_endpoints` / `get_endpoint` (section 10) for request and response schemas; do not guess
field names.

### Sending a message

```bash
curl -sS https://api-sandbox.connectix.bg/messages \
  -H "Authorization: $CONNECTIX_SANDBOX_TOKEN" \
  -H "Accept: application/json" -H "Content-Type: application/json" \
  -d '{"phone":"+359888123456","template":"<template uuid>","parameters":{"orderId":"1042"}}'
```

Optional fields: `ttl` (60-86400 seconds), `callbackUrl` (status webhook), `inboundUrl` (reply
webhook), `contact` (`firstName`, optional `lastName`, `groups`, `parameters`, `addGroupIfMissing`).
The response is the message object: `id`, `phone` (E.164), `channel`, `status`, `price`, `parts`,
`createdAt`.

### Fallback (Viber first, SMS if not delivered)

`sendFallbackMessage` takes `phone` plus a `flow` array of at least two steps, each with `position`,
`template` and optional `parameters`/`ttl`. Step 1 is sent first; when it is not delivered inside its
`ttl`, step 2 goes out. Every message in the flow shares one `fallbackId`; the callback webhook of a
fallback message carries it.

### Media

`sendMediaMessage` sends a file (`fileUrl`, `fileType`, `fileName`) over Viber; send it as JSON.
Not every application has a media service configured - a `400` here usually means that.

## 4. OTP flow

1. `sendOtp` with `phone` (+ optional `ttl`, `codeLength`, `maxAttempts`, `purpose`, `text`,
   `callbackUrl`). Response: `{id, expiresAt, maxAttempts, channel}`. On the sandbox the response
   additionally contains `code` (nothing is delivered there), so send -> verify can be tested end to
   end. On production the code is never returned - only the recipient sees it.
2. `verifyOtp` with `id` and `code`. Success returns `{"verified": true, "verifiedAt": ...}`; a
   wrong code returns **401** with `{"verified": false, "remainingAttempts": n}` - a 401 with that
   body is a wrong code, not a bad token.
3. `resendOtp` with `id` expires that OTP and returns a new one (new `id`, new code, new
   `expiresAt`); call `verifyOtp` with the new `id`. Counts against the per-phone hourly limit
   (`429`).

Terminal states: `404` unknown id, `410` expired or already used, `403` attempts exhausted. Create a
new OTP after any of them; do not retry.

## 5. Phone numbers

Send phones in **E.164** (`+359888123456`). National Bulgarian formats (`0888123456`) are accepted
because the application's default country is BG, but E.164 is the only format that works for every
country and every endpoint. Unparsable phones fail with `502` on messages and production OTP (the
sandbox OTP routes answer `422` `{"phone": "Could not parse the phone number."}`), `400` on
blacklist and TrustCheck, and are **silently dropped** by `createShipment` - validate before sending.

## 6. Webhooks

Five hook types: `callback` (message status), `inbound` (replies), `blacklist`, `template`,
`tracking`. They are plain JSON `POST`s to the URL you registered, currently **unsigned** - verify
the payload against the API (`checkBlacklist`, `listTemplates`) before acting on it. Register per
message (`callbackUrl`, `inboundUrl`) or per integration (`registerIntegrationHook`). Payloads are in
[references/webhooks.md](references/webhooks.md); the spec carries them under `x-webhooks`.

## 7. Errors

Every error body is JSON: either a string message or an object of `field: message` pairs (422).

| Status | Meaning | What to do |
|---|---|---|
| 400 | Invalid JSON, missing field, unknown template, text too long, service not configured | Fix the request; check the message |
| 401 | Missing/invalid token, or the company is not active | Check host and sandbox token |
| 402 | Insufficient funds | Human tops up the balance (live only) |
| 403 | Access restricted for this endpoint, OTP attempts exhausted, TrustCheck not eligible | Do not retry; tell the user |
| 404 | Unknown OTP id, integration or hook | Check the id |
| 415 | Body is not `application/json` | Set `Content-Type` |
| 422 | Validation errors as `{field: message}` | Fix the listed fields |
| 429 | Rate limit (per token, and per phone for OTP) | Back off; respect `Retry-After` |
| 451 | Contract not signed for this channel | Human signs the contract in the console |
| 502 | Phone could not be parsed, country not allowed, provider unavailable | Validate the phone; retry later for provider errors |

Also seen: `406` (traffic banned or no sender for the country), `410` (OTP expired/used).
Full table and per-endpoint notes: [references/errors.md](references/errors.md).

## 8. The three credentials

| Credential | Where it lives | Who uses it |
|---|---|---|
| Sandbox token | `CONNECTIX_SANDBOX_TOKEN`; console AI tools page | You. `Authorization` header on api-sandbox |
| Integration token + secret | Returned by `registerIntegration` (token) / chosen by the client (secret) | Your code: `?integration=<token>` on `createShipment`, `token` in hook registration |
| Live token | Console application page | **Humans only.** Never in an AI session |

## 9. Platforms and plugins

Official plugins exist for **WordPress / WooCommerce, PrestaShop and OpenCart**. If the user's shop
runs on one of them, install and extend the official plugin (hooks, filters, templates) instead of
rebuilding the integration against the raw API. Laravel, Symfony, Node, Python and plain PHP
integrate with the HTTP API directly; a PHP SDK exists. Details and where to extend each plugin are
in [references/platforms.md](references/platforms.md).

## 10. Documentation tools (MCP)

The `connectix-docs` MCP server (`https://mcp.connectix.bg/mcp`, no login) is the source of truth
for request and response shapes. Use it before writing a request, and when an error message is
unclear:

| Tool | Arguments | Use it for |
|---|---|---|
| `search_docs` | `query`, optional `product` (sms, viber, noti, otp, trustcheck, blacklist, tracking, account), `limit` <= 10, `lang` (en, bg) | Finding the endpoint or guide for a task |
| `list_endpoints` | optional `product`, `lang` | The operationId, method, path and deprecation state of every endpoint |
| `get_endpoint` | `operationId`, optional `lang` | Full schema, error table, sandbox notes and a curl example |
| `get_guide` | `slug`, optional `lang` | A guide (quickstart, auth, sandbox, webhooks, errors, changelog) |

Pass `lang: bg` when the user writes in Bulgarian. Without the MCP server, the same content is at
https://docs.connectix.bg, `https://mcp.connectix.bg/llms.txt` and `.../openapi.json`.
Postman collection (sandbox only; variables for the base URL and the token):
`https://mcp.connectix.bg/connectix.postman_collection.json` (`?lang=bg` for Bulgarian).

## 11. Working method

1. `getMe` on the sandbox to confirm credentials.
2. `listTemplates`; pick an approved template for the channel the user wants.
3. Look the endpoint up with `get_endpoint`; write the request from the schema.
4. Run it against the sandbox; handle every status in section 7 explicitly.
5. Write a test that hits the sandbox (or mocks the documented response) before asking the user to
   review.
6. Transactional vs promotional: order, delivery and OTP messages are transactional; marketing sends
   need promotional templates and respect the blacklist (`checkBlacklist` before a promotional send).
7. Stop at "go live" (section 1).
