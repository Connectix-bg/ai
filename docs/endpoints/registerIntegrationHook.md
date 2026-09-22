# Register a webhook hook on an integration

`POST /integration/register/hook`

- Operation: `registerIntegrationHook`
- Tags: Integrations
- Product: account
- Base URL: https://api-sandbox.connectix.bg

Subscribes a URL to one event type of the integration. Register one hook per `type` you need: `callback` (delivery status), `inbound` (replies), `template` (template changes), `tracking` (shipment events), `blacklist` (blacklist changes). Each hook receives HTTP POSTs with the JSON payload documented in the webhook definitions.

Authenticate with the application token in the `Authorization` header: on `https://api-sandbox.connectix.bg` use the application's **sandbox token**; the production host uses the production token of the same application.

## Authentication

Send the token in the `Authorization` header. Application token sent as the raw value of the `Authorization` header (`Authorization: <token>`); `Bearer <token>` is also accepted. Use the sandbox token on `https://api-sandbox.connectix.bg`. Use the sandbox token from the console; the raw token and `Bearer <token>` are both accepted.

## Sandbox

Same route on both hosts. With the sandbox token it operates on the application's real data (nothing is mocked).

## Request body

Content type: `application/json`. Required.

| Property | Type | Required | Description |
|---|---|---|---|
| `token` | string | yes | Integration token returned by `registerIntegration`. |
| `type` | object (HookType), one of: callback, inbound, template, tracking, blacklist | yes | `callback` = message delivery status, `inbound` = reply from a recipient, `template` = a template was created or changed, `tracking` = shipment tracking event, `blacklist` = a blacklist entry changed. See the webhook definitions for the payloads. |
| `url` | string (uri) | yes | HTTP(S) URL with a public TLD (HTTPS recommended) that receives the webhook POST. |

Example:

```json
{
    "token": "npu5K6DCMcsmLyDjLNpSkc8szq8iUSlY6Ga96vJjxsKOYWiAILbvQp9qXkKKGIfl",
    "type": "callback",
    "url": "https://shop.example.com/webhooks/connectix/status"
}
```

## Responses

| Status | Description |
|---|---|
| 200 | Hook registered. Returns object (Hook). |
| 400 | Invalid request. Either a required value is missing, the template is not approved / not found, the text exceeds the channel limit, or the body is not valid JSON (`{"error": "Invalid JSON: Syntax error"}`). Returns object (Error). |
| 401 | Missing or unknown token for this host, or the company is suspended. Check that you use the sandbox token on the sandbox host. Returns object (Error). |
| 403 | Access to this resource is disabled for the application (`access_restricted`), the TrustCheck eligibility rules are not met, or - on the production host only - an AI coding tool called a send/OTP route (`ai_tools_must_use_sandbox`: develop against the sandbox, going live is a human step). Returns object (Error). |
| 404 | The referenced object (integration, hook, OTP) does not belong to this application or does not exist. Returns object (Error). |
| 429 | Rate limited. Either the per-application request budget for the current minute is spent (`{"message": "rate_limited", "retry_after": <seconds>}` with a `Retry-After` header; every response carries `X-RateLimit-Remaining`), or - on the OTP routes - the per-phone OTP limit was reached (plain string, default 5 OTPs per hour per application). Wait and retry; do not tighten the loop. Returns object (Error). |

### 200 response

| Property | Type | Description |
|---|---|---|
| `id` | string (uuid) | Hook ID; pass it to `unregisterIntegrationHook`. |

```json
{
    "id": "f4212b48-a763-42b8-8e43-0ed413255edd"
}
```

## Example request

```bash
curl -X POST "https://api-sandbox.connectix.bg/integration/register/hook" \
  -H "Authorization: $CONNECTIX_SANDBOX_TOKEN" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"token":"npu5K6DCMcsmLyDjLNpSkc8szq8iUSlY6Ga96vJjxsKOYWiAILbvQp9qXkKKGIfl","type":"callback","url":"https://shop.example.com/webhooks/connectix/status"}'
```
