# Register a shipment for tracking

`POST /shipments`

- Operation: `createShipment`
- Tags: Tracking
- Product: tracking
- Base URL: https://api-sandbox.connectix.bg

Registers a courier shipment so Connectix tracks it, feeds TrustCheck and posts `tracking` webhooks on every state change. The courier is detected from the tracking number format. The shipment is attributed to the integration passed in `?integration=`. Re-registering an existing tracking number is idempotent. When the courier profile of the application has automatic shipment fetching enabled the shipment is not created here (it arrives automatically) but the call still answers `ok`.

Authenticate with the application token in the `Authorization` header: on `https://api-sandbox.connectix.bg` use the application's **sandbox token**; the production host uses the production token of the same application.

## Authentication

Send the token in the `Authorization` header. Application token sent as the raw value of the `Authorization` header (`Authorization: <token>`); `Bearer <token>` is also accepted. Use the sandbox token on `https://api-sandbox.connectix.bg`. Use the sandbox token from the console; the raw token and `Bearer <token>` are both accepted.

## Sandbox

Same route on both hosts. With the sandbox token it operates on the application's real data (nothing is mocked).

## Parameters

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `integration` | query | yes | string | Integration token obtained from `registerIntegration`. The shipment is attributed to that integration and its `tracking` hooks receive the events. (A deprecated `token` body field is still accepted instead.) |

## Request body

Content type: `application/json`. Required.

| Property | Type | Required | Description |
|---|---|---|---|
| `trackingNumber` | string | yes | Courier tracking number. The courier (Econt, Speedy, Evropat, Sameday, BoxNow) is detected from its format. |
| `phone` | string | no | Recipient phone in E.164 format. Used to link the shipment to a contact and to TrustCheck; an unparsable value is ignored. |
| `token` | string | no | Deprecated: integration token in the body. Use the `integration` query parameter instead. |

Example:

```json
{
    "trackingNumber": "1053912345678",
    "phone": "+359881234567"
}
```

## Responses

| Status | Description |
|---|---|
| 200 | Shipment registered (or already known). Returns object (OkAnswer), one of: ok. |
| 400 | Invalid request. Either a required value is missing, the template is not approved / not found, the text exceeds the channel limit, or the body is not valid JSON (`{"error": "Invalid JSON: Syntax error"}`). Returns object (Error). |
| 401 | Missing or unknown token for this host, or the company is suspended. Check that you use the sandbox token on the sandbox host. Returns object (Error). |
| 403 | Access to this resource is disabled for the application (`access_restricted`), the TrustCheck eligibility rules are not met, or - on the production host only - an AI coding tool called a send/OTP route (`ai_tools_must_use_sandbox`: develop against the sandbox, going live is a human step). Returns object (Error). |
| 404 | The referenced object (integration, hook, OTP) does not belong to this application or does not exist. Returns object (Error). |
| 429 | Rate limited. Either the per-application request budget for the current minute is spent (`{"message": "rate_limited", "retry_after": <seconds>}` with a `Retry-After` header; every response carries `X-RateLimit-Remaining`), or - on the OTP routes - the per-phone OTP limit was reached (plain string, default 5 OTPs per hour per application). Wait and retry; do not tighten the loop. Returns object (Error). |
| 502 | The request could not be processed: the phone number could not be parsed, the recipient's country is not enabled for the application, no provider is available, or a storage error occurred. Returns object (Error). |

### 200 response

```json
"ok"
```

## Example request

```bash
curl -X POST "https://api-sandbox.connectix.bg/shipments?integration=npu5K6DCMcsmLyDjLNpSkc8szq8iUSlY6Ga96vJjxsKOYWiAILbvQp9qXkKKGIfl" \
  -H "Authorization: $CONNECTIX_SANDBOX_TOKEN" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"trackingNumber":"1053912345678","phone":"+359881234567"}'
```
