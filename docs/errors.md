# Errors

# Errors

Send `Accept: application/json` on every request; without it some errors are rendered as HTML pages. Error bodies are small JSON values: usually a string, for validation failures an object.

```json
"Insufficient funds."
```

```json
{"[phone]": "This value is not a valid mobile phone number.", "[template]": "This field is missing."}
```

## Status codes

| Status | Meaning | What to do |
|---|---|---|
| 400 | Bad request: unknown template, template not approved, no channel service configured, text too long, a required field missing | Fix the request; check `GET /templates` for the template status |
| 401 | Missing or unknown token, token used on the wrong host, or the company is not active | See the authentication guide |
| 402 | **Insufficient balance**: the company account cannot pay for this message | Top up the balance in the console. The sandbox never returns 402 |
| 403 | The resource is restricted for this application, or a TrustCheck rule blocks it (`access_restricted`, `trustcheck_no_tracking`, `trustcheck_low_volume`); for OTP: maximum verification attempts exceeded | Contact support, or for TrustCheck enable automatic tracking / reach the daily volume |
| 404 | Integration, hook or OTP not found | Check the identifier and that it belongs to this application |
| 406 | Traffic for this company is banned, or there is no sender configured for the recipient's country | Contact support |
| 410 | The OTP has expired or was already used | Send a new OTP |
| 422 | Validation failed; the body maps field paths to messages | Fix the listed fields |
| 429 | OTP rate limit for this phone number | Wait before sending another OTP to the same number |
| 451 | **No signed contract for this channel** (Viber and Noti require one) | Sign the channel contract in the console under Documents. The sandbox never returns 451 |
| 500 | Unexpected error | Retry later; report the request id if it persists |
| 502 | Upstream problem: the phone number could not be parsed, the country is not allowed, or the provider is unavailable | Check the phone is E.164; retry with back-off |

## Validation errors (422)

Keys are the field paths of the request body in brackets, nested fields as `[contact][firstName]`, array items as `[flow][0][template]`. Every listed field must be fixed; the message is human-readable and not meant to be parsed.

Common causes:

- `phone` not in E.164 (`+359888123456`) or not a mobile number.
- `template` missing or not a UUID.
- `ttl` outside 60–86400 seconds (OTP: 60–3600).
- `callbackUrl` / `inboundUrl` / `url` not an absolute `http(s)://` URL with a hostname.
- Unknown properties in the body: the validators reject fields they do not know.

## OTP verification (401 with a body)

`POST /otp/verify` answers 401 with `{"verified": false, "remainingAttempts": 2}` for a wrong code. It is the only 401 that is not an authentication problem.

## Retrying

- Retry 5xx and network errors with exponential back-off; message sends are not idempotent, so keep your own record of what was accepted (the `id` in the 200 response) before retrying.
- Never retry 4xx without changing the request, except 429 after the wait.

