# Send a one-time password

`POST /otp/send`

- Operation: `sendOtp`
- Tags: OTP
- Product: otp
- Base URL: https://api-sandbox.connectix.bg

Generates a numeric code, sends it by SMS to the phone and returns the OTP `id` to use with `verifyOtp`. The code itself is never returned on production. Rate limited per phone and application (default 5 per hour, returns 429).

Phone numbers must be in E.164 format. Authenticate with the application token in the `Authorization` header: on `https://api-sandbox.connectix.bg` use the application's **sandbox token**; the production host uses the production token of the same application.

**402** means the account balance cannot cover the message price (`Insufficient funds.`) - top up the account before retrying.

## Authentication

Send the token in the `Authorization` header. Application token sent as the raw value of the `Authorization` header (`Authorization: <token>`); `Bearer <token>` is also accepted. Use the sandbox token on `https://api-sandbox.connectix.bg`. Use the sandbox token from the console; the raw token and `Bearer <token>` are both accepted.

## Sandbox

Sandbox host issues a real OTP record but sends no SMS; the response additionally contains `code` so the verify flow can be exercised end to end. The balance check is skipped (OTP has no contract check on either host). An unparsable phone returns 422 instead of 502.

## Request body

Content type: `application/json`. Required.

| Property | Type | Required | Description |
|---|---|---|---|
| `phone` | string | yes | Recipient mobile number. Send it in E.164 format (`+359881234567`). National formats are parsed against the countries enabled for the application, but E.164 is the only format that is guaranteed to work. Landline numbers are rejected with 422. |
| `text` | string | no | Custom SMS text; must contain the `{{code}}` placeholder. Defaults to the application's OTP template (`Connectix - Your verification code is: {{code}}`). |
| `ttl` | integer | no | Seconds until the code expires. Default 300. |
| `codeLength` | integer | no | Number of digits. Default 6. |
| `maxAttempts` | integer | no | Verification attempts allowed before the OTP is locked. Default 3. |
| `purpose` | string | no | Free-text label stored with the OTP (e.g. `login`, `checkout`). |
| `callbackUrl` | string (uri) | no | URL that receives the `callback` webhook for the SMS carrying the code. |

Example:

```json
{
    "phone": "+359881234567",
    "ttl": 300,
    "codeLength": 6,
    "maxAttempts": 3,
    "purpose": "checkout"
}
```

## Responses

| Status | Description |
|---|---|
| 200 | OTP issued and SMS queued. Returns object (Otp). |
| 400 | Invalid request. Either a required value is missing, the template is not approved / not found, the text exceeds the channel limit, or the body is not valid JSON (`{"error": "Invalid JSON: Syntax error"}`). Returns object (Error). |
| 401 | Missing or unknown token for this host, or the company is suspended. Check that you use the sandbox token on the sandbox host. Returns object (Error). |
| 402 | Insufficient funds: the balance does not cover the price of the message. Top up the account before retrying. Returns object (Error). |
| 403 | Access to this resource is disabled for the application (`access_restricted`), the TrustCheck eligibility rules are not met, or - on the production host only - an AI coding tool called a send/OTP route (`ai_tools_must_use_sandbox`: develop against the sandbox, going live is a human step). Returns object (Error). |
| 406 | Traffic is suspended for the company, or no sender ID is configured for the recipient's country. Returns object (Error). |
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
    "id": "c9d8e7f6-a5b4-4c3d-9e2f-1a0b9c8d7e6f",
    "expiresAt": "2026-09-20T10:20:30+03:00",
    "maxAttempts": 3,
    "channel": "sms"
}
```

## Example request

```bash
curl -X POST "https://api-sandbox.connectix.bg/otp/send" \
  -H "Authorization: $CONNECTIX_SANDBOX_TOKEN" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"phone":"+359881234567","ttl":300,"codeLength":6,"maxAttempts":3,"purpose":"checkout"}'
```
