# Errors reference

Send `Accept: application/json` on every request; without it some errors are rendered as HTML.
Error bodies are JSON: a string (`"OTP has expired."`) or, for `422`, an object keyed by field
(`{"phone": "This value is not a valid phone number."}`).

## Status codes

| Status | Where | Meaning | Action |
|---|---|---|---|
| 400 | all | Invalid JSON body; missing required field (`createShipment`, integrations, blacklist, TrustCheck, holidays); template not found; message text too long; channel service not configured for the application; unsupported country for `listHolidays` (only `bg`) | Fix the request. The message says which |
| 401 | all | No `Authorization` header, unknown token, token for the other host, company not active or traffic-banned. On `verifyOtp` also a wrong code (`{"verified": false, "remainingAttempts": n}`) | Check host and token; if the token is right, the account needs a human in the console |
| 402 | send, OTP | Insufficient funds (live only) | Stop; the balance is a human task |
| 403 | blacklist, TrustCheck, OTP verify | Endpoint restricted for this application; TrustCheck not eligible (body says why); OTP maximum attempts exceeded | Do not retry. For OTP, create a new one |
| 404 | OTP, integrations, tracking | Unknown OTP `id`; integration not found for the token/secret; hook not found | Check the id or token |
| 406 | send, OTP | Traffic banned, or no sender configured for the destination country | Human task in the console |
| 410 | OTP | OTP expired, or already used | Create a new OTP |
| 415 | POST endpoints | Body is not `application/json` | Set `Content-Type: application/json` |
| 422 | send, OTP | Validation errors, one entry per field | Fix every listed field, then resend |
| 429 | all; OTP per phone | Rate limit exceeded (per token; OTP also per phone) | Back off, honour `Retry-After`; never loop |
| 451 | send, OTP | The channel's contract is not signed (live only) | Human signs it in the console |
| 500 | all | Unexpected server error | Retry once later; report if it persists |
| 502 | send, OTP, blacklist | Phone could not be parsed (production; the sandbox OTP routes answer `422` `{"phone": ...}` instead); country not allowed; provider unavailable | Validate the phone (E.164). For provider errors retry with backoff |

## Per-endpoint notes

- `sendMessage`, `sendFallbackMessage`: `422` lists each invalid field, including nested `flow[n].field`
  entries. A `template` must be a UUID of an **approved** template.
- `sendMediaMessage`: `400` for a missing media service; `415` for non-JSON bodies.
- `sendOtp`, `resendOtp`: `429` is per phone as well as per token. On the sandbox an unparsable
  phone is `422` with a bare `phone` key (not the bracketed `[phone]` path of validator errors).
- `verifyOtp`: a wrong code returns `401` with `{"verified": false, "remainingAttempts": n}` - tell it
  apart from a token `401` by the object body. Success is `200` `{"verified": true, "verifiedAt": "..."}`.
- `checkBlacklist`: returns the strings `"yes"` or `"no"`; `403` when blacklist access is restricted.
- `checkShopper`: `403` with an `error` field when the application is not eligible for TrustCheck;
  `"no_info"` (200) when nothing is known about the phone.
- `createShipment`: `400` when `trackingNumber` or the integration token is missing; `404` unknown
  integration. An unparsable `phone` is dropped silently, not rejected.
- `registerIntegrationHook`: the integration `token` is exactly 64 characters; `type` is one of
  `callback`, `inbound`, `template`, `tracking`, `blacklist`; `url` must be an absolute URL with a TLD.

## Handling pattern

```
2xx            -> use the body
422            -> surface the field errors to the caller, do not retry
400/404/415    -> bug in the request, fix and redeploy
401/402/403/406/451 -> stop and tell the user; a human resolves it in the console
429            -> sleep(Retry-After or 60s), retry once
502/500        -> retry with backoff (max 3), then fail
```
