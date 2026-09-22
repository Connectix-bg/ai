# Add, update or remove a phone in the blacklist

`POST /blacklist/toggle`

- Operation: `toggleBlacklist`
- Tags: Blacklist
- Product: blacklist
- Base URL: https://api-sandbox.connectix.bg

Creates or updates the blacklist entry for the phone. `active: true` blocks the number, `active: false` unblocks it; `permanent: true` keeps it blocked for transactional messages too; `global: true` makes the entry apply to every application of the company (their `checkBlacklist` answers `yes`) and announces the change to all of their `blacklist` hooks; otherwise it applies to this application only and only its `blacklist` hooks are notified.

Phone numbers must be in E.164 format. Authenticate with the application token in the `Authorization` header: on `https://api-sandbox.connectix.bg` use the application's **sandbox token**; the production host uses the production token of the same application.

## Authentication

Send the token in the `Authorization` header. Application token sent as the raw value of the `Authorization` header (`Authorization: <token>`); `Bearer <token>` is also accepted. Use the sandbox token on `https://api-sandbox.connectix.bg`. Use the sandbox token from the console; the raw token and `Bearer <token>` are both accepted.

## Sandbox

Same route on both hosts. With the sandbox token it operates on the application's real data (nothing is mocked).

## Request body

Content type: `application/json`. Required.

| Property | Type | Required | Description |
|---|---|---|---|
| `phone` | string | yes | Recipient mobile number. Send it in E.164 format (`+359881234567`). National formats are parsed against the countries enabled for the application, but E.164 is the only format that is guaranteed to work. Landline numbers are rejected with 422. |
| `active` | boolean | no | `true` blocks the number, `false` unblocks it. |
| `permanent` | boolean | no | When `true` the entry never expires, even for transactional messages. |
| `global` | boolean | no | When `true` the entry applies to every application of the company (their `checkBlacklist` answers `yes`) and the change is announced to all of their `blacklist` hooks; when `false` it applies to this application only. |

Example:

```json
{
    "phone": "+359881234567",
    "active": true,
    "permanent": false,
    "global": false
}
```

## Responses

| Status | Description |
|---|---|
| 200 | Current state of the entry. Returns object (BlacklistEntry). |
| 400 | Invalid request. Either a required value is missing, the template is not approved / not found, the text exceeds the channel limit, or the body is not valid JSON (`{"error": "Invalid JSON: Syntax error"}`). Returns object (Error). |
| 401 | Missing or unknown token for this host, or the company is suspended. Check that you use the sandbox token on the sandbox host. Returns object (Error). |
| 403 | Access to this resource is disabled for the application (`access_restricted`), the TrustCheck eligibility rules are not met, or - on the production host only - an AI coding tool called a send/OTP route (`ai_tools_must_use_sandbox`: develop against the sandbox, going live is a human step). Returns object (Error). |
| 429 | Rate limited. Either the per-application request budget for the current minute is spent (`{"message": "rate_limited", "retry_after": <seconds>}` with a `Retry-After` header; every response carries `X-RateLimit-Remaining`), or - on the OTP routes - the per-phone OTP limit was reached (plain string, default 5 OTPs per hour per application). Wait and retry; do not tighten the loop. Returns object (Error). |
| 502 | The request could not be processed: the phone number could not be parsed, the recipient's country is not enabled for the application, no provider is available, or a storage error occurred. Returns object (Error). |

### 200 response

| Property | Type | Description |
|---|---|---|
| `id` | string (uuid) |  |
| `phone` | string | E.164 phone number. |
| `active` | boolean |  |
| `permanent` | boolean |  |
| `global` | boolean |  |
| `expire_at_for_transactional` | string (date-time), nullable | When the entry stops blocking transactional messages; `null` for permanent entries or when the application has no blacklist validity. |

```json
{
    "id": "7d6c5b4a-3f2e-4d1c-8b0a-9f8e7d6c5b4a",
    "phone": "+359881234567",
    "active": true,
    "permanent": false,
    "global": false,
    "expire_at_for_transactional": "2026-09-27T10:15:30+03:00"
}
```

## Example request

```bash
curl -X POST "https://api-sandbox.connectix.bg/blacklist/toggle" \
  -H "Authorization: $CONNECTIX_SANDBOX_TOKEN" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"phone":"+359881234567","active":true,"permanent":false,"global":false}'
```
