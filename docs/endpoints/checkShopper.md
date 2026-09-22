# Check the TrustCheck risk rating of a shopper

`POST /shopper/check`

- Operation: `checkShopper`
- Tags: TrustCheck
- Product: trustcheck
- Base URL: https://api-sandbox.connectix.bg

Returns the risk that the shopper behind the phone number refuses or does not collect a cash-on-delivery parcel, computed from the shipment history tracked by Connectix across merchants. Use it before confirming a COD order.

TrustCheck requires automatic shipment tracking to be active for at least one courier profile and at least 10 tracked shipments per day; below 10 shipments/day for 7 consecutive days the service suspends and reactivates automatically once 10 shipments are reached in a day. 403 codes: `access_restricted` (disabled by staff), `trustcheck_no_tracking` (no courier profile with automatic tracking), `trustcheck_low_volume` (suspended by the volume rule).

Phone numbers must be in E.164 format; an unparsable phone returns 400 here. Authenticate with the application token in the `Authorization` header: on `https://api-sandbox.connectix.bg` use the application's **sandbox token**; the production host uses the production token of the same application.

## Authentication

Send the token in the `Authorization` header. Application token sent as the raw value of the `Authorization` header (`Authorization: <token>`); `Bearer <token>` is also accepted. Use the sandbox token on `https://api-sandbox.connectix.bg`. Use the sandbox token from the console; the raw token and `Bearer <token>` are both accepted.

## Sandbox

Same route on both hosts; returns the real rating and applies the same eligibility rules (nothing is mocked).

## Request body

Content type: `application/json`. Required.

| Property | Type | Required | Description |
|---|---|---|---|
| `phone` | string | yes | Recipient mobile number. Send it in E.164 format (`+359881234567`). National formats are parsed against the countries enabled for the application, but E.164 is the only format that is guaranteed to work. Landline numbers are rejected with 422. |

Example:

```json
{
    "phone": "+359881234567"
}
```

## Responses

| Status | Description |
|---|---|
| 200 | Risk rating. Returns object (RiskRating), one of: no_info, low, medium, high, ultra_high. |
| 400 | Invalid request. Either a required value is missing, the template is not approved / not found, the text exceeds the channel limit, or the body is not valid JSON (`{"error": "Invalid JSON: Syntax error"}`). Returns object (Error). |
| 401 | Missing or unknown token for this host, or the company is suspended. Check that you use the sandbox token on the sandbox host. Returns object (Error). |
| 403 | Access to this resource is disabled for the application (`access_restricted`), the TrustCheck eligibility rules are not met, or - on the production host only - an AI coding tool called a send/OTP route (`ai_tools_must_use_sandbox`: develop against the sandbox, going live is a human step). Returns object (Error). |
| 429 | Rate limited. Either the per-application request budget for the current minute is spent (`{"message": "rate_limited", "retry_after": <seconds>}` with a `Retry-After` header; every response carries `X-RateLimit-Remaining`), or - on the OTP routes - the per-phone OTP limit was reached (plain string, default 5 OTPs per hour per application). Wait and retry; do not tighten the loop. Returns object (Error). |

### 200 response

```json
"medium"
```

## Example request

```bash
curl -X POST "https://api-sandbox.connectix.bg/shopper/check" \
  -H "Authorization: $CONNECTIX_SANDBOX_TOKEN" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"phone":"+359881234567"}'
```
