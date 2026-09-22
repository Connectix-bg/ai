# CLAUDE.md

## Connectix

This project integrates the Connectix messaging API (Viber, SMS, Noti, OTP, TrustCheck, blacklist,
shipment tracking).

1. Load the `connectix` skill before touching any Connectix-related code. With the plugin installed
   it is available as the `connectix` skill; without it, read `skills/connectix/SKILL.md` from
   https://github.com/connectix-bg/ai.
2. Sandbox only: every request goes to `https://api-sandbox.connectix.bg` with the sandbox token
   from the `CONNECTIX_SANDBOX_TOKEN` environment variable. Never write another Connectix host, never
   read or print a live token. When asked to go live, stop and point to the go-live guide at
   https://docs.connectix.bg/en/go-live.
3. Look request and response shapes up with the `connectix-docs` MCP tools (`search_docs`,
   `list_endpoints`, `get_endpoint`, `get_guide`) or https://docs.connectix.bg; do not guess fields.
4. Run the project's tests (including the ones that hit the sandbox) before asking a human to
   review. Handle every documented error status explicitly; never loop on `429`.
5. Phones in E.164; templates must be approved; promotional sends check the blacklist first.

<!-- Copy this section into your repository's CLAUDE.md. Install the plugin with
     `claude plugin marketplace add connectix-bg/ai` and `claude plugin install connectix@connectix`. -->
