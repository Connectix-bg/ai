# Quickstart

# Quickstart

Send your first message from the sandbox in five minutes. Everything on this page runs against `https://api-sandbox.connectix.bg`: delivery is simulated and nothing is billed.

## 1. Get a sandbox token

In the Connectix console open **Settings → Applications**, pick an application and copy its **sandbox token**. Keep it in an environment variable; never paste it into source code:

```bash
export CONNECTIX_SANDBOX_TOKEN="paste-the-sandbox-token-here"
```

The sandbox token only works on `api-sandbox.connectix.bg`. The application also has a live token; that one is for production and is a human decision, see the go-live guide when you are ready.

## 2. Check who you are

```bash
curl "https://api-sandbox.connectix.bg/me" \
  -H "Authorization: $CONNECTIX_SANDBOX_TOKEN" \
  -H "Accept: application/json"
```

```json
{"name": "My shop"}
```

The token goes in the `Authorization` header as-is; `Authorization: Bearer <token>` is accepted too. Always send `Accept: application/json` so errors come back as JSON.

## 3. Pick a template

Messages are sent from templates approved in the console:

```bash
curl "https://api-sandbox.connectix.bg/templates" \
  -H "Authorization: $CONNECTIX_SANDBOX_TOKEN" \
  -H "Accept: application/json"
```

Use the `id` of a template whose `status` is `approved`. Placeholders in the template text (`{{ orderNumber }}`) are filled from `parameters`.

## 4. Send a message

```bash
curl -X POST "https://api-sandbox.connectix.bg/messages" \
  -H "Authorization: $CONNECTIX_SANDBOX_TOKEN" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "template": "3644d7ac-2a95-4068-863e-6352ed78d667",
    "phone": "+359888123456",
    "parameters": {"orderNumber": "10042"},
    "callbackUrl": "https://example.com/connectix/callback"
  }'
```

Phone numbers are E.164: country code, no spaces, no leading zeros (`+359888123456`). The response is the accepted message:

```json
{
  "id": "0d1c1c2e-8f2a-4c1e-9c2b-2d1a6d1f7e21",
  "type": "transactional",
  "phone": "+359888123456",
  "message": {"text": "Your order 10042 has shipped."},
  "price": 0.055,
  "parts": 1,
  "channel": "viber",
  "status": "received",
  "ttl": 7200,
  "callbackUrl": "https://example.com/connectix/callback",
  "createdAt": "2026-01-15T10:30:00+02:00"
}
```

`status: received` means accepted for delivery. The sandbox then fakes the rest of the lifecycle and POSTs each status change to `callbackUrl` (see the webhooks guide).

## 5. Read the errors that matter

| Status | Meaning | What to do |
|---|---|---|
| 401 | Missing or wrong token, or a token used on the wrong host | Check the header and that the sandbox token is used on the sandbox host |
| 402 | Insufficient balance | Top up in the console (the sandbox never returns this) |
| 422 | A field failed validation; the body maps field paths to messages | Fix the payload |
| 451 | No signed contract for the channel | Sign the channel contract in the console (the sandbox never returns this) |

The errors guide lists every status the API returns.

## Next steps

- Viber first, SMS if not delivered: `POST /messages/fallback`.
- One-time passwords: `POST /otp/send`, `/otp/verify`, `/otp/resend`.
- Delivery and inbound notifications: the webhooks guide.
- Let your AI coding tool read these docs: the build-with-ai guide.
- Postman collection: import `https://mcp.connectix.bg/connectix.postman_collection.json`, set the `token` variable to your sandbox token.

