# Build with AI

# Build with AI

Let your AI coding tool read the Connectix API contract directly: a docs MCP server, plain-text docs for any agent that can fetch a URL, and an installable skill. Everything the tools see targets the sandbox (`https://api-sandbox.connectix.bg`) and the sandbox token; going live stays a human step.

## The docs MCP server

`https://mcp.connectix.bg/mcp` — HTTP transport, no sign-in, read-only. Four tools:

| Tool | What it returns |
|---|---|
| `search_docs` | Ranked hits over endpoint summaries, descriptions and guide headings (`query`, optional `product`, `limit` ≤ 10, `lang`) |
| `list_endpoints` | operationId, method, path, summary, deprecated/sunset flags (optional `product`) |
| `get_endpoint` | One endpoint: schema, auth, error table, sandbox notes, a curl example against the sandbox (`operationId`, `lang`: `en` or `bg`) |
| `get_guide` | One guide as Markdown (`slug`: quickstart, authentication, sandbox, webhooks, errors, build-with-ai) |

### Install

**Claude Code**

```bash
claude mcp add --transport http connectix-docs https://mcp.connectix.bg/mcp
```

**Codex CLI / app**

```bash
codex mcp add connectix-docs --url https://mcp.connectix.bg/mcp
```

**Cursor** — `.cursor/mcp.json`:

```json
{"mcpServers": {"connectix-docs": {"url": "https://mcp.connectix.bg/mcp"}}}
```

Also add `https://connectix.bg/docs` under Settings › Features › Docs to use `@docs`.

**VS Code / GitHub Copilot** — `.vscode/mcp.json`:

```json
{"servers": {"connectix-docs": {"type": "http", "url": "https://mcp.connectix.bg/mcp"}}}
```

**Copilot CLI**

```bash
copilot mcp add --transport http connectix-docs https://mcp.connectix.bg/mcp
```

**ChatGPT (web)** — Settings › Developer mode › add a remote MCP server by URL (no authentication).

**Claude.ai, Claude Desktop, Cowork** — Customize › Connectors › Add custom connector › URL `https://mcp.connectix.bg/mcp`, "No sign-in".

**Any agent**

```bash
npx add-mcp https://mcp.connectix.bg/mcp
```

Other clients take the same URL: Kiro (`type: http`), Antigravity (`serverUrl`), Devin Desktop (`serverUrl`), Cline (`type: streamableHttp`), OpenCode, Zed and JetBrains (URL only).

## The skill and plugin

The skill teaches an agent the Connectix conventions in one file: sandbox-first, raw-or-Bearer auth, E.164 phones, the channel matrix, Viber → SMS fallback, the OTP flow, webhooks, what each error status means, and that official WordPress, PrestaShop and OpenCart plugins already exist.

**Skill only, any tool**

```bash
npx skills add connectix-bg/ai
```

**Claude Code plugin**

```bash
claude plugin marketplace add connectix-bg/ai
claude plugin install connectix@connectix
```

**Codex plugin**

```bash
codex plugin marketplace add connectix-bg/ai
codex plugin add connectix@connectix
```

**Cursor** — `npx skills add connectix-bg/ai`, then `/add-plugin connectix` after review.

**GitHub Copilot (VS Code)** — settings: `"chat.plugins.marketplaces": ["connectix-bg/ai"]`. Copilot coding agent: copy the `skills/` folder into `.github/skills/`.

## Plain-text docs for any agent

- `https://mcp.connectix.bg/llms.txt` — the index: guides, endpoints, links.
- `https://mcp.connectix.bg/llms-full.txt` — everything in one file.
- `https://mcp.connectix.bg/docs/{slug}.md` — one guide, e.g. `/docs/quickstart.md`; add `?lang=bg` for Bulgarian.
- `https://mcp.connectix.bg/openapi.json` and `/openapi.bg.json` — the OpenAPI 3.0 document, for Context7, SDK generators and `@docs`.
- `https://mcp.connectix.bg/connectix.postman_collection.json` — Postman collection (v2.1): every endpoint against the sandbox, with `baseUrl` and `token` variables.

## A note for your repo

Paste this into `AGENTS.md`, `CLAUDE.md` or `GEMINI.md` so every session starts with the right defaults:

```markdown
## Connectix messaging API
- Docs MCP: connectix-docs (https://mcp.connectix.bg/mcp); read llms.txt at https://mcp.connectix.bg/llms.txt before writing code.
- Use https://api-sandbox.connectix.bg with the sandbox token from the environment variable CONNECTIX_SANDBOX_TOKEN. Do not switch hosts or tokens; going live is done by a person following the go-live guide.
- Authorization header: raw token or "Bearer <token>". Always send Accept: application/json.
- Phone numbers are E.164. 451 = no signed contract for the channel; 402 = balance; 422 = validation, body lists the fields.
```

## Why sandbox only

An AI tool that can read these docs can write working code, and code that works in the sandbox works in production after a human swaps the host and the token. Keeping the production host and the live token out of every AI-facing surface means no generated snippet, tool result or example can send a real, billed message by accident. When you are ready, a person follows the go-live guide.

