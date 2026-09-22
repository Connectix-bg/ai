# Connectix for AI coding tools

Skill, plugin manifests and documentation server for the Connectix API - Viber, SMS and Noti
messaging, OTP, TrustCheck, blacklist and shipment tracking for Bulgarian businesses.

**Sandbox only.** Everything here targets `https://api-sandbox.connectix.bg` with the sandbox token
from the Connectix console. Going live is a human step described at
https://docs.connectix.bg/en/go-live.

This repository is a build artefact mirrored from GitLab by CI on every `docs-v*` tag. Do not open
pull requests here; see [CONTRIBUTING.md](CONTRIBUTING.md). The version in [VERSION](VERSION) equals
the `info.version` of the published OpenAPI spec.

## Install

| Tool | Skill + plugin | Docs MCP server |
|---|---|---|
| Claude Code | `claude plugin marketplace add connectix-bg/ai` then `claude plugin install connectix@connectix` | `claude mcp add --transport http connectix-docs https://mcp.connectix.bg/mcp` |
| Codex CLI / ChatGPT | `codex plugin marketplace add connectix-bg/ai` then `codex plugin add connectix@connectix`; skill only: `npx skills add connectix-bg/ai` | `codex mcp add connectix-docs --url https://mcp.connectix.bg/mcp` |
| Cursor | `npx skills add connectix-bg/ai`, or `/add-plugin connectix` | `.cursor/mcp.json`: `{"mcpServers":{"connectix-docs":{"url":"https://mcp.connectix.bg/mcp"}}}` |
| GitHub Copilot (VS Code) | `chat.plugins.marketplaces: ["connectix-bg/ai"]` | `.vscode/mcp.json`: `{"servers":{"connectix-docs":{"type":"http","url":"https://mcp.connectix.bg/mcp"}}}` |
| Copilot CLI / coding agent | copy `skills/` into `.github/skills/` | `copilot mcp add --transport http connectix-docs https://mcp.connectix.bg/mcp` |
| Gemini CLI | copy `templates/GEMINI.md` into your repo | `.gemini/settings.json`: `{"mcpServers":{"connectix-docs":{"httpUrl":"https://mcp.connectix.bg/mcp"}}}` |
| Claude.ai / Desktop | - | Customize > Connectors > Add custom connector > `https://mcp.connectix.bg/mcp` (no sign-in) |
| Any agent | `npx skills add connectix-bg/ai` | `npx add-mcp https://mcp.connectix.bg/mcp` |
| Postman | - | Import `https://mcp.connectix.bg/connectix.postman_collection.json` (sandbox only; `?lang=bg` for Bulgarian) |

## Layout

```
plugin.json              Agent Plugins 1.0 manifest (root; Codex, Cursor, Copilot, Kiro, VS Code)
mcp.json                 Agent Plugins MCP config (streamable-http)
skills/connectix/        The skill: SKILL.md + references/
.claude-plugin/          Claude Code plugin.json + marketplace.json
.mcp.json                Claude Code MCP config
.codex-plugin/           Codex compatibility manifest
.cursor-plugin/          Cursor manifest
templates/               AGENTS.md, CLAUDE.md, GEMINI.md snippets for your repository
docs/                    Guides and one Markdown page per endpoint, generated from the OpenAPI spec
openapi.json             The public OpenAPI 3.0.3 contract (English)
llms.txt, llms-full.txt  Index and full text for LLM ingestion
CHANGELOG.md, VERSION    Contract version history
```

## Links

- Documentation: https://docs.connectix.bg
- OpenAPI: https://mcp.connectix.bg/openapi.json (Bulgarian: `openapi.bg.json`)
- MCP server: https://mcp.connectix.bg/mcp
- Issues: https://github.com/connectix-bg/ai/issues
