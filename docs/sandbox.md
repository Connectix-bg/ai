# Sandbox

# Sandbox

The sandbox is the same API on a different host, `https://api-sandbox.connectix.bg`, unlocked by the application's **sandbox token**. Build and test there; nothing you send reaches a phone and nothing is billed.

## What the sandbox does differently

| | Sandbox | Production |
|---|---|---|
| Host | `api-sandbox.connectix.bg` | the production host (go-live guide) |
| Token | sandbox token | live token |
| Message delivery | simulated: the worker picks a final status at random | real carriers |
| OTP | codes are created and can be verified, but no SMS is sent | real |
| Balance check (402) | skipped | enforced |
| Contract check (451) | skipped | enforced |
| Pricing | `price` is filled from the price list but nothing is charged | charged |
| Callbacks | sent, with the simulated statuses | sent |

Because delivery is faked, the sandbox is the right place to test every status transition of your callback handler: `received → sent → delivered`, `undelivered`, `failed`, and the Viber → SMS fallback (`could_not_be_sent` on the first step, then the next step is sent).

## What behaves exactly like production

Everything that is not "sending": the sandbox token authenticates the same application, so

- `GET /templates` returns the application's real templates; only `approved` ones can be sent.
- `POST /blacklist/toggle` writes to the real blacklist and `POST /blacklist/check` reads it.
- `POST /integration/register` and the hook endpoints create real integrations and hooks.
- `POST /shipments` registers real shipments for tracking.
- `POST /shopper/check` runs a real TrustCheck query (and needs the same eligibility).
- `Idempotency-Key`, `reference`, `GET /messages/{id}` and `GET /messages` work the same way (on the sandbox messages).
- Validation, phone parsing and the 4xx responses are identical, including the public-URL rule for `callbackUrl`/`inboundUrl`.

Treat those as real data, because they are.

## Prerequisites

A fresh account whose sandbox answers 401 or 400 is usually missing one of these:

1. The company is **active** (registration completed and verified in the console); otherwise every request is 401.
2. At least one **channel service** is configured for the application; otherwise sends answer 400 with a "service not found" message.
3. For template sends, a template in status `approved`.

## Not available in the sandbox yet

- `POST /messages/media` has no sandbox implementation; test media messages only after going live, with a test phone.

## Going live

Switching host and token is a human step with a checklist (contract, balance, approved templates, reachable callback URL). It is described in the go-live guide, which is written for people and is intentionally not part of the AI-readable docs.

