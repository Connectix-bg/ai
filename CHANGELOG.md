# Connectix API changelog

One entry per published contract version. The version equals `info.version` in `openapi.json`,
`VERSION` in the GitHub mirror and the plugin manifests. Dates are release dates of the docs tag.

## 1.2.0 — safe retries, references, signed and retried webhooks, read-back

- `Idempotency-Key` header on `POST /messages`, `/messages/fallback`, `/messages/media` and
  `/otp/send`: the 2xx answer of the first request is stored for 24 hours and replayed for every
  repeat of the same key (`Idempotent-Replayed: true`); a repeat with a different request answers
  422 `[idempotencyKey]`, a repeat while the first one is running answers 409 `in_progress` with
  `Retry-After`.
- `reference` (1-128 chars) on the same requests, echoed as `reference` in the answer and in the
  `callback` and `inbound` webhooks; a fallback flow carries one reference across all its steps.
- `GET /messages/{id}` (`getMessage`) and `GET /messages` (`listMessages`: `from`/`to` window of at
  most 7 days, `status`, `channel`, `reference`, cursor pages) return the message object plus
  `updatedAt`.
- Webhooks: every event carries a unique `eventId` (body and `X-Connectix-Event-Id`); deliveries to
  integration hooks are signed (`X-Connectix-Timestamp`, `X-Connectix-Signature: v1=<HMAC-SHA256>`
  keyed with the integration secret); failed deliveries are retried per URL with back-off, 12 times
  over about 32 hours; redirects are no longer followed. Payload schemas gained `eventId` (all) and
  `reference` (`callback`, `inbound`).
- Callback, inbound, hook and media URLs must be publicly reachable: private, loopback, link-local and
  cloud-metadata addresses are refused with 422.
- Unchanged for existing clients: every field and header is additive; receivers that ignore unknown
  keys and headers keep working. The only behavioural change is that a receiver answering non-2xx now
  sees retries.

## 1.1.0 — contract trimmed to the offered surface

- Removed from the public contract: `POST /trackings` (the deprecated alias of `POST /shipments`;
  the route keeps working for existing callers until its 2027-03-31 sunset) and
  `GET /countries/holidays` (an internal calendar helper, not part of the API offer). The
  `Holidays` schema went with it. 17 documented operations remain.
- No other change: requests, responses and webhooks of the remaining operations are as in 1.0.0.

## 1.0.0 — first public contract

- First public OpenAPI 3.0.3 contract for the Connectix API, in English (`openapi.json`) and
  Bulgarian (`openapi.bg.json`).
- 19 documented operations (17 endpoints plus two deprecated aliases) across Templates, Messages,
  OTP, Integrations, Blacklist, TrustCheck, Tracking and Account. `POST /messages/pure` is intentionally excluded.
- `Authorization` accepts the raw token and `Bearer <token>`.
- A per-application request budget of 300 requests per minute per host, reported in
  `X-RateLimit-Remaining`; enforcement (`429` with `Retry-After`) is being phased in. `415` for a
  non-JSON body on `POST /messages/media`.
- Webhook payloads (`callback`, `inbound`, `blacklist`, `template`, `tracking`) published under
  `x-webhooks`.
- Deprecated, with sunset dates in the spec (`x-sunset`): `POST /blacklist` (an alias of
  `POST /blacklist/check`; use that) and `POST /trackings` (use `POST /shipments`).
- New machine-readable surfaces: `llms.txt`, `llms-full.txt`, Markdown endpoint pages, the
  `connectix-docs` MCP server (`search_docs`, `list_endpoints`, `get_endpoint`, `get_guide`), the
  `connectix` skill and plugin manifests for Claude Code, Codex, Cursor and Agent Plugins 1.0.
- A Postman Collection v2.1 download (`connectix.postman_collection.json`, `?lang=bg` for
  Bulgarian) rendered from the spec, `{{baseUrl}}` pinned to the sandbox.
- Sandbox-only policy for AI tools: every AI-facing surface names `https://api-sandbox.connectix.bg`
  only; going live is a human-only guide.
