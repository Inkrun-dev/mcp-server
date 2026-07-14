# Inkrun MCP Server

**The official Model Context Protocol (MCP) server for [Inkrun](https://inkrun.dev): give Claude, ChatGPT, Cursor, and any MCP-compatible client the ability to turn Markdown into consistently themed PDFs — and to pull structured data back out of PDFs, with grounded, never-guessed extraction.**

[![MCP](https://img.shields.io/badge/Model_Context_Protocol-compatible-000000?logo=modelcontextprotocol&logoColor=white)](https://modelcontextprotocol.io)
![Auth: OAuth 2.1 or API key](https://img.shields.io/badge/Auth-OAuth_2.1_%7C_API_key-2EBC4F)
![Hosting: Inkrun Cloud](https://img.shields.io/badge/Hosting-Inkrun_Cloud-6C5CE7)

[**Getting started**](https://inkrun.dev/docs/mcp/claude-desktop) ·
[**All clients**](https://inkrun.dev/docs/mcp/clients) ·
[**REST API**](https://inkrun.dev/docs/api/render) ·
[**Dashboard**](https://inkrun.dev)

---

Inkrun is a hosted document service that works in both directions. **Documents out:** your agent sends Markdown, Inkrun renders it through a themed pipeline (typography, cover pages, charts, headers/footers) and returns a download link. **Data in:** your agent sends a PDF, Inkrun returns structured data — every value grounded at a concrete place in the document, with per-field confidence and provenance, never a fabricated answer. The MCP server is **cloud-hosted** — there is nothing to install or run locally. This repository is the public home for connection configs, the MCP Registry entry, and issue reports.

With Inkrun connected, you can ask your AI tool to:

- **Render any Markdown as a polished PDF** — "make a PDF of this report with the sales-proposal template."
- **Extract structured data from a PDF** — "pull the vendor, line items and totals from this invoice" — using a built-in preset or your own JSON Schema, with per-field confidence and page provenance.
- **Extract every table from a PDF** as rows + CSV, tagged with page provenance.
- **Round-trip a document in one call** — "take this supplier invoice and produce a payment advice on our letterhead": extraction fills a `{{field}}` Markdown template and the result renders as a themed PDF.
- **Fill a fillable (AcroForm) PDF from JSON** — government forms, insurance claims, onboarding packets.
- **Inspect a document for free** before spending anything — page count, digital-vs-scanned classification, fillable-form field inventory, and exactly what each extraction tier would cost.
- **Reuse your saved templates and themes**, or create new ones from chat.
- **Embed charts** with a fenced ` ```chart ` block.
- **Email the PDF to someone** as a branded download link valid for 3 days.

## Contents

- [Endpoints](#endpoints)
- [Supported clients](#supported-clients)
- [Available tools](#available-tools)
- [Quick setup](#quick-setup)
- [Agent skill](#agent-skill)
- [Credits and rate limits](#credits-and-rate-limits)
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

Both endpoints expose the same ten tools.

**Markdown → PDF:**

| Tool | What it does |
| --- | --- |
| `create_pdf` | Render Markdown → themed PDF, store it, return a download URL. Accepts `template`, `theme`, `format`, `title` — plus `sendEmailTo` + `senderName` (and optional `emailMessage`) to email the PDF as a branded download link valid for 3 days. |
| `list_templates` | List your saved templates plus the built-ins. |
| `create_template` | Create a reusable style option (theme + accent + fonts + format). |
| `get_styles` | Return a theme's CSS + page config. |
| `get_chart_syntax` | Explain the `chart` fenced-block syntax for embedding charts. |

**PDF → data:**

| Tool | What it does |
| --- | --- |
| `inspect_document` | **Free.** Page count, per-page digital-vs-scanned classification, detected languages, fillable-form field inventory, and what each extraction tier would cost for *this* document. The "inspect first" step of every extraction workflow. |
| `extract` | Extract structured data using a built-in preset (`invoice`, `receipt`, `contract_terms`, `table_dump`, `full_text`) or a custom JSON Schema. Every value is grounded in the document's positioned text; ungrounded fields come back `null` with a warning — never a guess. Returns per-field confidence, method, and page provenance. Opt-in OCR (`ocr_fallback: true`) for scanned pages. |
| `extract_tables` | Extract every recoverable table as rows + CSV with `{page, index}` provenance. Deterministic by default; opt-in OCR for scanned pages. |

**Round trip & forms:**

| Tool | What it does |
| --- | --- |
| `extract_and_generate` | Extract + render in one call: extraction fills a `{{field}}` Markdown template and the result renders as a themed PDF. Extraction and render credits reserve atomically together — a round trip is never half-billed. Substituted values are inert text; a missing field renders as a visible blank plus a warning, never an invented value. |
| `fill_form` | Fill a fillable (AcroForm) PDF from a JSON map of field names → values. Use `inspect_document` first (free) to list the exact field names. Unknown fields and invalid options come back as warnings, never silent drops. The filled PDF is stored and returned as a download URL. |

Documents are sent as `file_url` (preferred — the server downloads it), `file_base64`, or `file_path` (local stdio server only).

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

[`skills/inkrun/`](skills/inkrun/) ships an [Agent Skill](https://code.claude.com/docs/en/skills) that teaches Claude how to use Inkrun well — when to reach for which tool, the inspect-first extraction workflow, preset vs schema, OCR opt-in, the round-trip `{{field}}` template contract, form filling, cover-page frontmatter, chart syntax, font preset keys, email delivery, and the REST fallback.

Install it for Claude Code (all projects):

```bash
mkdir -p ~/.claude/skills && cp -R skills/inkrun ~/.claude/skills/inkrun
```

Or for a single project, copy it to `.claude/skills/inkrun/` in that repo. Once installed, Claude applies it automatically whenever a task involves turning Markdown into a PDF or pulling data out of one.

## Credits and rate limits

Every metered operation costs **credits** against a monthly allowance (calendar month, UTC), and API keys have a steady-rate burst guard. Both scale with your plan:

| Plan | Monthly credits | API rate limit | Document upload cap |
| --- | --- | --- | --- |
| Free | 20 | 1 request / sec | 10 MB |
| Pro | 500, then metered overage up to 2,000 more | 5 requests / sec | 25 MB |
| Team | 5,000, then metered overage up to 20,000 more | 20 requests / sec | 50 MB |

What operations cost:

| Operation | Credits |
| --- | --- |
| `create_pdf` render | 1 |
| `inspect_document` | **Free** (rate-limited, never billed) |
| `extract` / `extract_tables`, deterministic tier | 1 per 20 pages |
| `extract`, model tier | 3 up to 5 pages, +1 per additional 5 |
| `extract`, OCR tier (opt-in only) | 10 up to 5 pages, +2 per additional 5 |
| `extract_and_generate` | The extraction tier actually used + 1 render credit, reserved atomically together |
| `fill_form` | 1 per filled PDF |

You are only ever charged for the extraction tier actually used, an over-quota request fails before any work is done, and failed operations are refunded. `inspect_document` reports the exact cost of each tier for your specific document before you commit — inspect liberally.

See the [render API reference](https://inkrun.dev/docs/api/render) and the [extract API reference](https://inkrun.dev/docs/api/extract) for the error responses when you hit a limit.

## Data and security

- All traffic is encrypted in transit over HTTPS.
- **OAuth 2.1** tokens are issued per client through Inkrun's consent flow; **API keys** are scoped to your account and revocable from the dashboard at any time.
- Renders are attributed to your account and appear in your [History](https://inkrun.dev/history); download links are presigned and expire.
- **Source documents you send for inspection, extraction, round trips, or form filling are processed in memory and never stored** — page images included. Only the structured extraction *result*, and any PDF Inkrun *produces* for you (renders, round-trip outputs, filled forms), are retained with your account history. See [data handling](https://inkrun.dev/docs/data-handling).
- Extraction never fabricates: a field that cannot be grounded in the document comes back `null` with a warning, and OCR-read values are labelled as such with capped confidence.
- Markdown input is rendered with raw HTML disabled, so untrusted document content cannot inject markup into the PDF pipeline.

MCP lets AI agents act with your account's permissions, which is powerful but carries structural risks — LLMs are vulnerable to [prompt injection](https://owasp.org/www-community/attacks/PromptInjection) and related attacks. Only enable MCP servers you trust, use least privilege (revoke unused keys), and review what an agent is about to send before rendering or extracting sensitive content.

To report a vulnerability, see [SECURITY.md](SECURITY.md).

## Support and feedback

- **Bugs and feature requests** — [open an issue](../../issues) in this repository.
- **Documentation** — [inkrun.dev/docs](https://inkrun.dev/docs) is the canonical reference; this README is a summary.
- **Account or billing questions** — via the [dashboard](https://inkrun.dev).

## Disclaimer

MCP clients can create renders, templates, extractions, and emails on your behalf with your existing Inkrun permissions and credits. Review high-impact actions (like emailing a PDF to a third party or sending a sensitive document for extraction) before confirming.
