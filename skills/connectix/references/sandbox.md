# Sandbox reference

The sandbox is the only Connectix host an AI session may use: **`https://api-sandbox.connectix.bg`**.

## What it is

- Same routes, validators and response shapes as the live API, same `operationId`s, same error
  codes - code written against the sandbox works on the live host after a human swaps the host and
  the token.
- Authenticated with the **sandbox token** of an application. The token listener picks the token
  column by host: a sandbox token works only on `api-sandbox.connectix.bg`, a live token only on the
  live host. A mixed-up pair gets `401`.
- Nothing is delivered and nothing is billed. Contract, pricing and balance checks are skipped, so
  `402` and `451` do not occur on the sandbox (they will on the live host - handle them anyway).
- Delivery is fabricated by the worker so that callbacks can be tested: the final status is picked at random. A fallback flow therefore sometimes reaches step 2
  and sometimes does not - never assert a specific outcome in a test, assert the shape.

## Prerequisites the user must have (console, human)

- The company is **active** (a fresh, unverified account gets `401` on every call).
- At least one channel service is configured for the application; otherwise
  sends fail with `400` "service not found".
- At least one **approved** template for that channel (`listTemplates` shows `status`).
- The sandbox token is shown on the console's AI tools page. The live token is on the application
  page and is not for you.

## Not available on the sandbox

- `sendMediaMessage` (POST /messages/media) has no sandbox route today; the spec's
  `x-connectix-agent.sandbox` note on that operation says so. Write the code from the schema and
  tell the user it can only be exercised live, by them.

## OTP on the sandbox

Nothing is delivered, so the sandbox `sendOtp` and `resendOtp` responses include the generated
`code` - send -> `verifyOtp` can be exercised end to end. On production the code is never returned.
The sandbox OTP routes also answer `422` `{"phone": "Could not parse the phone number."}` for an
unparsable phone (production answers `502`).

## Rate limits

The per-token limiter applies on the sandbox too (`429` with `Retry-After`). OTP additionally
limits sends per phone. Do not loop on `429`; back off.

## Go-live is a human step

When the user wants production: stop, and send them to the go-live guide on
https://docs.connectix.bg/docs/go-live (it is marked human-only and is not served to AI tools). It
covers replacing the host and the token, the active contract, approved templates, a reachable
callback URL, blacklist handling and one real test message.
