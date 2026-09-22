# Remove an integration

`POST /integration/unregister`

- Operation: `unregisterIntegration`
- Tags: Integrations
- Product: account
- Base URL: https://api-sandbox.connectix.bg

Deletes the integration identified by `token` and `secret`, together with all of its hooks. Messages already attributed to it are kept.

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
| `secret` | string | yes | The secret used at registration. |

Example:

```json
{
    "token": "npu5K6DCMcsmLyDjLNpSkc8szq8iUSlY6Ga96vJjxsKOYWiAILbvQp9qXkKKGIfl",
    "secret": "VWAIKyRhzVmGkgDXTxepB1rO49CYKo3C"
}
```

## Responses

| Status | Description |
|---|---|
| 200 | Integration removed. Returns object (OkAnswer), one of: ok. |
| 400 | Invalid request. Either a required value is missing, the template is not approved / not found, the text exceeds the channel limit, or the body is not valid JSON (`{"error": "Invalid JSON: Syntax error"}`). Returns object (Error). |
| 401 | Missing or unknown token for this host, or the company is suspended. Check that you use the sandbox token on the sandbox host. Returns object (Error). |
| 403 | Access to this resource is disabled for the application (`access_restricted`), the TrustCheck eligibility rules are not met, or - on the production host only - an AI coding tool called a send/OTP route (`ai_tools_must_use_sandbox`: develop against the sandbox, going live is a human step). Returns object (Error). |
| 404 | The referenced object (integration, hook, OTP) does not belong to this application or does not exist. Returns object (Error). |
| 429 | Rate limited. Either the per-application request budget for the current minute is spent (`{"message": "rate_limited", "retry_after": <seconds>}` with a `Retry-After` header; every response carries `X-RateLimit-Remaining`), or - on the OTP routes - the per-phone OTP limit was reached (plain string, default 5 OTPs per hour per application). Wait and retry; do not tighten the loop. Returns object (Error). |

### 200 response

```json
"ok"
```

## Example request

```bash
curl -X POST "https://api-sandbox.connectix.bg/integration/unregister" \
  -H "Authorization: $CONNECTIX_SANDBOX_TOKEN" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"token":"npu5K6DCMcsmLyDjLNpSkc8szq8iUSlY6Ga96vJjxsKOYWiAILbvQp9qXkKKGIfl","secret":"VWAIKyRhzVmGkgDXTxepB1rO49CYKo3C"}'
```
