# Inkrun MCP Server

**The official Model Context Protocol (MCP) server for [Inkrun](https://inkrun.dev): give Claude, ChatGPT, Cursor, and any MCP-compatible client the ability to turn Markdown into consistently themed, downloadable PDFs.**

[![MCP](https://img.shields.io/badge/Model_Context_Protocol-compatible-000000?logo=modelcontextprotocol&logoColor=white)](https://modelcontextprotocol.io)
![Auth: OAuth 2.1 or API key](https://img.shields.io/badge/Auth-OAuth_2.1_%7C_API_key-2EBC4F)
![Hosting: Inkrun Cloud](https://img.shields.io/badge/Hosting-Inkrun_Cloud-6C5CE7)

[**Getting started**](https://inkrun.dev/docs/mcp/claude-desktop) ·
[**All clients**](https://inkrun.dev/docs/mcp/clients) ·
[**REST API**](https://inkrun.dev/docs/api/render) ·
[**Dashboard**](https://inkrun.dev)

---

Inkrun is a hosted Markdown → PDF service: your agent sends Markdown, Inkrun renders it through a themed pipeline (typography, cover pages, charts, headers/footers) and returns a download link. The MCP server is **cloud-hosted** — there is nothing to install or run locally. This repository is the public home for connection configs, the MCP Registry entry, and issue reports.

With Inkrun connected, you can ask your AI tool to:

- **Render any Markdown as a polished PDF** — "make a PDF of this report with the sales-proposal template."
- **Reuse your saved templates and themes**, or create new ones from chat.
- **Embed charts** with a fenced ` ```chart ` block.
- **Email the PDF to someone** as a branded download link valid for 3 days.

## Contents

- [Endpoints](#endpoints)
- [Supported clients](#supported-clients)
- [Available tools](#available-tools)
- [Quick setup](#quick-setup)
- [Agent skill](#agent-skill)
- [Quotas](#quotas)
- [Data and security](#data-and-security)
- [Support and feedback](#support-and-feedback)

---

## Endpoints

Pick by how your client authenticates:

| Your client… | Endpoint | Auth |
| --- | --- | --- |
| supports **OAuth connectors** (claude.ai, Claude Desktop, ChatGPT) | `https://inkrun.dev/api/mcp` | OAuth 2.1 — sign in once, no key |
| can send **custom headers** (Claude Code, Cursor, VS Code, scripts) | `https://mcp.inkrun.dev/mcp` | API key — `Authorization: Bearer sk_...` |

Create and manage API keys in the dashboard under [**API & MCP**](https://inkrun.dev/api-mcp). Keys look like `sk_...` and are shown once.

## Supported clients

| Client | Setup guide |
| --- | --- |
| Claude Desktop | [Connect Claude Desktop](https://inkrun.dev/docs/mcp/claude-desktop) |
| claude.ai (web) | [Connect claude.ai](https://inkrun.dev/docs/mcp/claude-ai) |
| Claude Code | [Connect other clients](https://inkrun.dev/docs/mcp/clients) |
| ChatGPT | [Connect other clients](https://inkrun.dev/docs/mcp/clients) |
| Cursor, VS Code, agent frameworks | [Connect other clients](https://inkrun.dev/docs/mcp/clients) |

Any client that speaks MCP **streamable HTTP** works: use the OAuth endpoint if your client supports OAuth connectors, or the API-key endpoint if it can send an `Authorization` header.

## Available tools

Both endpoints expose the same five tools:

| Tool | What it does |
| --- | --- |
| `create_pdf` | Render Markdown → themed PDF, store it, return a download URL. Accepts `template`, `theme`, `format`, `title` — plus `sendEmailTo` + `senderName` (and optional `emailMessage`) to email the PDF as a branded download link valid for 3 days. |
| `list_templates` | List your saved templates plus the built-ins. |
| `create_template` | Create a reusable style option (theme + accent + fonts + format). |
| `get_styles` | Return a theme's CSS + page config. |
| `get_chart_syntax` | Explain the `chart` fenced-block syntax for embedding charts. |

## Quick setup

### Claude Desktop / claude.ai

**Settings → Connectors → Add custom connector**, name it **Inkrun**, and paste:

```
https://inkrun.dev/api/mcp
```

Sign in once when the browser opens — Claude receives its own OAuth token. Full walkthrough: [Claude Desktop](https://inkrun.dev/docs/mcp/claude-desktop) · [claude.ai](https://inkrun.dev/docs/mcp/claude-ai).

### Claude Code

```bash
claude mcp add --transport http inkrun https://mcp.inkrun.dev/mcp --header "Authorization: Bearer sk_..."
```

Add `--scope user` to make it available in every project. Alternatively, clone this repository — the included [`.mcp.json`](.mcp.json) registers the server automatically, reading your key from the `INKRUN_API_KEY` environment variable.

### ChatGPT

Requires a paid plan with **developer mode** enabled (**Settings → Apps & Connectors**). Create a connector with the OAuth endpoint `https://inkrun.dev/api/mcp`. Details: [Connect other clients](https://inkrun.dev/docs/mcp/clients).

### Any other MCP client

Clients with custom-header support connect with URL + header:

```json
{
  "mcpServers": {
    "inkrun": {
      "url": "https://mcp.inkrun.dev/mcp",
      "headers": { "Authorization": "Bearer sk_..." }
    }
  }
}
```

(Field names vary by client — e.g. VS Code uses `servers` with `"type": "http"` — but URL + header is all Inkrun needs.)

## Agent skill

[`skills/inkrun/`](skills/inkrun/) ships an [Agent Skill](https://code.claude.com/docs/en/skills) that teaches Claude how to use Inkrun well — when to reach for which tool, cover-page frontmatter, chart syntax, font preset keys, email delivery, and the REST fallback.

Install it for Claude Code (all projects):

```bash
mkdir -p ~/.claude/skills && cp -R skills/inkrun ~/.claude/skills/inkrun
```

Or for a single project, copy it to `.claude/skills/inkrun/` in that repo. Once installed, Claude applies it automatically whenever a task involves turning Markdown into a PDF.

## Quotas

Renders count against a monthly quota (calendar month, UTC), and API keys have a steady-rate burst guard. Both scale with your plan:

| Plan | Monthly renders | API rate limit |
| --- | --- | --- |
| Free | 50 | 1 request / sec |
| Pro | 500 | 5 requests / sec |

See the [render API reference](https://inkrun.dev/docs/api/render) for the error responses when you hit either limit.

## Data and security

- All traffic is encrypted in transit over HTTPS.
- **OAuth 2.1** tokens are issued per client through Inkrun's consent flow; **API keys** are scoped to your account and revocable from the dashboard at any time.
- Renders are attributed to your account and appear in your [History](https://inkrun.dev/history); download links are presigned and expire.
- Markdown input is rendered with raw HTML disabled, so untrusted document content cannot inject markup into the PDF pipeline.

MCP lets AI agents act with your account's permissions, which is powerful but carries structural risks — LLMs are vulnerable to [prompt injection](https://owasp.org/www-community/attacks/PromptInjection) and related attacks. Only enable MCP servers you trust, use least privilege (revoke unused keys), and review what an agent is about to send before rendering sensitive content.

To report a vulnerability, see [SECURITY.md](SECURITY.md).

## Support and feedback

- **Bugs and feature requests** — [open an issue](../../issues) in this repository.
- **Documentation** — [inkrun.dev/docs](https://inkrun.dev/docs) is the canonical reference; this README is a summary.
- **Account or billing questions** — via the [dashboard](https://inkrun.dev).

## Disclaimer

MCP clients can create renders, templates, and emails on your behalf with your existing Inkrun permissions and quota. Review high-impact actions (like emailing a PDF to a third party) before confirming.
