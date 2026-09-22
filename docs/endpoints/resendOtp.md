# Re-issue a one-time password

`POST /otp/resend`

- Operation: `resendOtp`
- Tags: OTP
- Product: otp
- Base URL: https://api-sandbox.connectix.bg

Expires the OTP identified by `id`, generates a new code with the same length, attempts and TTL, sends it by SMS and returns the **new** OTP - verify with the new `id`. Counts against the same hourly rate limit as `sendOtp`.

Authenticate with the application token in the `Authorization` header: on `https://api-sandbox.connectix.bg` use the application's **sandbox token**; the production host uses the production token of the same application.

**402** means the account balance cannot cover the message price (`Insufficient funds.`) - top up the account before retrying.

## Authentication

Send the token in the `Authorization` header. Application token sent as the raw value of the `Authorization` header (`Authorization: <token>`); `Bearer <token>` is also accepted. Use the sandbox token on `https://api-sandbox.connectix.bg`. Use the sandbox token from the console; the raw token and `Bearer <token>` are both accepted.

## Sandbox

Sandbox host works against OTPs issued on the sandbox host; nothing is sent.

## Request body

Content type: `application/json`. Required.

| Property | Type | Required | Description |
|---|---|---|---|
| `id` | string (uuid) | yes | ID of the OTP to replace. It is expired and a new OTP with a new `id` is issued. |

Example:

```json
{
    "id": "c9d8e7f6-a5b4-4c3d-9e2f-1a0b9c8d7e6f"
}
```

## Responses

| Status | Description |
|---|---|
| 200 | New OTP issued. Returns object (Otp). |
| 400 | Invalid request. Either a required value is missing, the template is not approved / not found, the text exceeds the channel limit, or the body is not valid JSON (`{"error": "Invalid JSON: Syntax error"}`). Returns object (Error). |
| 401 | Missing or unknown token for this host, or the company is suspended. Check that you use the sandbox token on the sandbox host. Returns object (Error). |
| 402 | Insufficient funds: the balance does not cover the price of the message. Top up the account before retrying. Returns object (Error). |
| 403 | Access to this resource is disabled for the application (`access_restricted`), the TrustCheck eligibility rules are not met, or - on the production host only - an AI coding tool called a send/OTP route (`ai_tools_must_use_sandbox`: develop against the sandbox, going live is a human step). Returns object (Error). |
| 404 | The referenced object (integration, hook, OTP) does not belong to this application or does not exist. Returns object (Error). |
| 406 | Traffic is suspended for the company, or no sender ID is configured for the recipient's country. Returns object (Error). |
| 410 | The OTP has already expired or has already been used. Issue a new one with `sendOtp`. Returns object (Error). |
| 422 | Validation failed. The body maps each offending field path to a message. Unknown fields are rejected. Returns object (Error). |
| 429 | Rate limited. Either the per-application request budget for the current minute is spent (`{"message": "rate_limited", "retry_after": <seconds>}` with a `Retry-After` header; every response carries `X-RateLimit-Remaining`), or - on the OTP routes - the per-phone OTP limit was reached (plain string, default 5 OTPs per hour per application). Wait and retry; do not tighten the loop. Returns object (Error). |
| 500 | Unexpected server error while processing the OTP. Safe to retry once. Returns object (Error). |
| 502 | The request could not be processed: the phone number could not be parsed, the recipient's country is not enabled for the application, no provider is available, or a storage error occurred. Returns object (Error). |

### 200 response

| Property | Type | Description |
|---|---|---|
| `id` | string (uuid) | OTP ID to pass to `verifyOtp` / `resendOtp`. |
| `expiresAt` | string (date-time), nullable |  |
| `maxAttempts` | integer |  |
| `channel` | string, one of: sms | OTPs are always sent over SMS. |
| `code` | string | The generated code. **Sandbox only** - never returned on production. |

```json
{
    "id": "0e1d2c3b-4a59-4687-9c0d-1e2f3a4b5c6d",
    "expiresAt": "2026-09-20T10:20:30+03:00",
    "maxAttempts": 3,
    "channel": "sms"
}
```

## Example request

```bash
curl -X POST "https://api-sandbox.connectix.bg/otp/resend" \
  -H "Authorization: $CONNECTIX_SANDBOX_TOKEN" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"id":"c9d8e7f6-a5b4-4c3d-9e2f-1a0b9c8d7e6f"}'
```
