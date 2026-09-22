# Verify a one-time password

`POST /otp/verify`

- Operation: `verifyOtp`
- Tags: OTP
- Product: otp
- Base URL: https://api-sandbox.connectix.bg

Checks the code the user entered against the OTP `id`. A wrong code returns **401** with `remainingAttempts`; once the attempts are exhausted every further call returns **403**; an expired or already verified OTP returns **410**. Successful verification is final - the OTP cannot be verified twice.

Authenticate with the application token in the `Authorization` header: on `https://api-sandbox.connectix.bg` use the application's **sandbox token**; the production host uses the production token of the same application.

## Authentication

Send the token in the `Authorization` header. Application token sent as the raw value of the `Authorization` header (`Authorization: <token>`); `Bearer <token>` is also accepted. Use the sandbox token on `https://api-sandbox.connectix.bg`. Use the sandbox token from the console; the raw token and `Bearer <token>` are both accepted.

## Sandbox

Sandbox host works against OTPs issued on the sandbox host; nothing is sent.

## Request body

Content type: `application/json`. Required.

| Property | Type | Required | Description |
|---|---|---|---|
| `id` | string (uuid) | yes | OTP ID from `sendOtp` or `resendOtp`. |
| `code` | string | yes | Digits entered by the user. |

Example:

```json
{
    "id": "c9d8e7f6-a5b4-4c3d-9e2f-1a0b9c8d7e6f",
    "code": "482913"
}
```

## Responses

| Status | Description |
|---|---|
| 200 | Code accepted. Returns object (OtpVerified). |
| 401 | Wrong code (`OtpRejected`), or missing/invalid application token (plain string). Returns any. |
| 403 | Access to this resource is disabled for the application (`access_restricted`), the TrustCheck eligibility rules are not met, or - on the production host only - an AI coding tool called a send/OTP route (`ai_tools_must_use_sandbox`: develop against the sandbox, going live is a human step). Returns object (Error). |
| 404 | The referenced object (integration, hook, OTP) does not belong to this application or does not exist. Returns object (Error). |
| 410 | The OTP has already expired or has already been used. Issue a new one with `sendOtp`. Returns object (Error). |
| 422 | Validation failed. The body maps each offending field path to a message. Unknown fields are rejected. Returns object (Error). |
| 429 | Rate limited. Either the per-application request budget for the current minute is spent (`{"message": "rate_limited", "retry_after": <seconds>}` with a `Retry-After` header; every response carries `X-RateLimit-Remaining`), or - on the OTP routes - the per-phone OTP limit was reached (plain string, default 5 OTPs per hour per application). Wait and retry; do not tighten the loop. Returns object (Error). |
| 500 | Unexpected server error while processing the OTP. Safe to retry once. Returns object (Error). |

### 200 response

| Property | Type | Description |
|---|---|---|
| `verified` | boolean, one of: 1 |  |
| `verifiedAt` | string (date-time), nullable |  |

```json
{
    "verified": true,
    "verifiedAt": "2026-09-20T10:16:02+03:00"
}
```

## Example request

```bash
curl -X POST "https://api-sandbox.connectix.bg/otp/verify" \
  -H "Authorization: $CONNECTIX_SANDBOX_TOKEN" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"id":"c9d8e7f6-a5b4-4c3d-9e2f-1a0b9c8d7e6f","code":"482913"}'
```
