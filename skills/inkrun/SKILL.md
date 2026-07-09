---
name: inkrun
description: Create themed PDFs from Markdown with Inkrun. Use when the user wants to render, export, share, or email a document — report, proposal, invoice, release notes — as a polished PDF. Covers the Inkrun MCP tools (create_pdf, list_templates, create_template, get_styles, get_chart_syntax), the REST API fallback, templates and themes, cover pages via YAML frontmatter, embedded charts, and email delivery.
---

# Inkrun — Markdown → themed PDF

Inkrun is a hosted service that renders Markdown through a themed pipeline
(typography, cover pages, charts, headers/footers) and returns a presigned
download URL. Nothing runs locally — you call the cloud MCP server or the REST
API.

## Prefer the MCP tools

If the Inkrun MCP server is connected (tools named `create_pdf`,
`list_templates`, `create_template`, `get_styles`, `get_chart_syntax` — the
server registers as `inkrun`), use it. If not connected, either register it
(see "Connecting" below) or fall back to the REST API.

### Typical workflow

1. **Pick a style.** Call `list_templates` to see the user's saved templates
   plus the built-in themes. If the user names a brand/template ("use our
   sales proposal look"), match it by name or slug.
2. **Write the Markdown.** Standard Markdown; raw HTML is **not** supported
   (it renders as escaped text). Use a `chart` fenced block for charts
   (see [charts.md](charts.md) or call `get_chart_syntax`).
3. **Fill the cover (if any).** Templates/themes with a designed cover page
   are flagged `hasCover: true` with `coverBindings` (e.g. `title`,
   `subtitle`). Start the Markdown with a YAML frontmatter block supplying
   those keys:

   ```markdown
   ---
   title: Q2 Revenue Report
   subtitle: Prepared for Acme Corp
   ---

   # Executive summary
   …
   ```

   Missing keys fall back to the cover's static placeholder text.
4. **Render.** Call `create_pdf` with the `markdown` and either `template`
   (a saved style option's slug/id — preferred for on-brand output) or
   `theme` (a raw base theme). Optional: `format` (`A4` | `Letter` |
   `Legal`), `title`.
5. **Return the download URL** from the tool result to the user. It is a
   presigned S3 link — no local file is written.

### Email delivery

To put the PDF straight in someone's inbox, add to `create_pdf`:

- `sendEmailTo` — recipient address
- `senderName` — **required with sendEmailTo**; the human the email is from
  ("Ada sent you a PDF")
- `emailMessage` — optional personal note quoted in the email

The recipient gets a branded email with a download link **valid for 3 days**
(a link, not an attachment). Confirm the recipient address with the user
before sending — email is outward-facing.

### Creating a reusable template

`create_template` saves a named style option = base theme + accent + fonts +
page format:

- `name` (required), `description`
- `themeName` — built-in name or one of the user's custom-theme slugs
- `format` — `A4` (default), `Letter`, `Legal`
- `accent` — hex colour, e.g. `#1F6FEB`
- `fontBody` / `fontHeading` / `fontMono` — preset keys only (see below)

Then render with `create_pdf({ template: "<slug>", … })`.

## Built-in themes

| Theme | Look |
| --- | --- |
| `default` | Clean, versatile document style — the fallback. |
| `editorial` | Long-form magazine feel — display serifs, generous measure. |
| `minimal` | Airy Swiss product-doc look — lots of whitespace. |
| `technical` | Dense developer-docs look — grid tables, monospace headings. |
| `bold` | High-impact pitch/report look — heavy headings, punchy accents. |

`theme` also accepts the slug/id of a custom theme the user designed in the
dashboard (those may carry a cover page).

## Font preset keys

Fonts are preset keys, never arbitrary names. System stacks: `serif`, `sans`,
`slab`, `mono`. Self-hosted Google fonts (embedded into the PDF): `inter`,
`roboto`, `opensans`, `lato`, `montserrat`, `poppins`, `merriweather`,
`lora`, `playfair`, `robotomono`.

## REST API fallback

Needs an API key (`sk_…`, created at https://inkrun.dev/api-mcp, expected in
`INKRUN_API_KEY`). Endpoint: `POST https://inkrun.dev/api/v1/render`.

```bash
# PDF bytes back directly:
curl "https://inkrun.dev/api/v1/render" \
  -H "Authorization: Bearer $INKRUN_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: application/pdf" \
  -d '{ "template": "quarterly-report", "markdown": "# Q2\n\nUp **12%** QoQ." }' \
  --output report.pdf
```

Drop the `Accept` header to get JSON `{ id, status, size, url, stored }`
where `url` is a presigned download link. Body fields mirror the MCP tool:
`markdown` (required), `template`, `theme`, `format`, `title`, and
`sendEmail: { to, senderName, message }`.

Keep the key server-side; the API rejects cross-origin browser reads.

## Connecting the MCP server

- **OAuth clients** (claude.ai, Claude Desktop, ChatGPT): add a custom
  connector with `https://inkrun.dev/api/mcp` and sign in once.
- **Header-based clients** (Claude Code, Cursor, VS Code, scripts):

  ```bash
  claude mcp add --transport http inkrun https://mcp.inkrun.dev/mcp \
    --header "Authorization: Bearer sk_..."
  ```

## Good to know

- Each successful render counts against the monthly quota (Free: 50,
  Pro: 500). A failed render is not charged.
- `create_pdf` requires an account (API key or OAuth) — renders are
  attributed and stored; anonymous rendering is not available over MCP.
- Charts render as static inline SVG — deterministic, no interactivity.
- Frontmatter `title:` doubles as the stored document title when no explicit
  `title` is passed.
