# Connectix API changelog

One entry per published contract version. The version equals `info.version` in `openapi.json`,
`VERSION` in the GitHub mirror and the plugin manifests. Dates are release dates of the docs tag.

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
