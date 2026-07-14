---
name: inkrun
description: Create themed PDFs from Markdown, extract structured data from PDFs, round-trip documents onto your own letterhead, and fill PDF forms with Inkrun. Use when the user wants to render, export, share, or email a document — report, proposal, invoice, release notes — as a polished PDF; wants data pulled out of a PDF — invoice fields, receipt totals, contract terms, tables, full text; wants a document restyled/regenerated from an extracted one; or wants a fillable (AcroForm) PDF form filled from data. Covers the Inkrun MCP tools (create_pdf, list_templates, create_template, get_styles, get_chart_syntax, inspect_document, extract, extract_tables, extract_and_generate, fill_form), the REST API fallback, templates and themes, cover pages via YAML frontmatter, embedded charts, email delivery, extraction presets vs custom JSON Schema, OCR opt-in, and credit costs.
---

# Inkrun — Markdown → themed PDF, and PDF → structured data

Inkrun is a hosted document service that works in both directions:

- **Documents out:** render Markdown through a themed pipeline (typography,
  cover pages, charts, headers/footers) and get a presigned download URL.
- **Data in:** send a PDF and get structured data back — every value grounded
  at a concrete place in the document with per-field confidence and page
  provenance; a field that can't be grounded comes back `null` with a
  warning, never a guess.
- **Both at once:** round-trip a document (extract → fill a `{{field}}`
  template → render on your letterhead) or fill a PDF form from JSON.

Nothing runs locally — you call the cloud MCP server or the REST API.

## Prefer the MCP tools

If the Inkrun MCP server is connected (tools named `create_pdf`,
`list_templates`, `create_template`, `get_styles`, `get_chart_syntax`,
`inspect_document`, `extract`, `extract_tables`, `extract_and_generate`,
`fill_form` — the server registers as `inkrun`), use it. If not connected,
either register it (see "Connecting" below) or fall back to the REST API.

## Rendering: Markdown → PDF

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
before sending — email is outward-facing. Requires a paid plan (free-tier
renders still succeed; the email is skipped with a notice).

### Creating a reusable template

`create_template` saves a named style option = base theme + accent + fonts +
page format:

- `name` (required), `description`
- `themeName` — built-in name or one of the user's custom-theme slugs
- `format` — `A4` (default), `Letter`, `Legal`
- `accent` — hex colour, e.g. `#1F6FEB`
- `fontBody` / `fontHeading` / `fontMono` — preset keys only (see below)

Then render with `create_pdf({ template: "<slug>", … })`.

## Extraction: PDF → structured data

### Always inspect first — it's free

`inspect_document` costs **zero credits** (rate-limited, never billed) and
returns: page count, per-page digital-vs-scanned classification, detected
languages, an inventory of fillable AcroForm fields (names and types, never
values), and **exactly what each extraction tier would cost for this
document** — including whether OCR is needed at all and for which pages.
Inspect liberally before choosing how to extract.

### Sending the document

All document tools take exactly ONE of:

- `file_url` — **preferred whenever the PDF is reachable at a URL.** The
  server downloads it, so you emit no base64 tokens. Must be publicly
  reachable (private/loopback addresses are refused).
- `file_base64` — works on every transport, but a 1 MB PDF is ~1.4 MB of
  base64 you must emit token by token. Use only when there is no URL.
- `file_path` — local filesystem path; **only works over the local stdio
  MCP server**, not remote connectors.

Only `application/pdf` is accepted, subject to the plan's document size cap.

### `extract` — structured fields

Send the document plus **exactly one** of:

- `preset` — a built-in extraction contract:

  | Preset | Extracts |
  | --- | --- |
  | `invoice` | Vendor, invoice number, dates, line items, subtotal/tax/total, currency. |
  | `receipt` | Merchant, purchase date, total, currency, payment method, item lines. |
  | `contract_terms` | Parties, effective date, initial term, auto-renewal, termination notice, governing law. |
  | `table_dump` | Every deterministically recoverable table as rows + CSV with provenance. |
  | `full_text` | The document's extractable text in reading order, whole-document and per-page. |

- `schema` — a JSON Schema **object** with `properties` (plain schema only:
  `$ref`, `allOf`/`anyOf`/`oneOf` are rejected; size/depth capped). Field
  NAMES double as label anchors in the document, so name them the way the
  document labels them: `invoice_number`, `due_date`, `total`. String fields
  with `format: "date"` parse as dates; number fields named
  total/amount/price parse as money.

Sending both, neither, or an unknown preset is an `invalid_request` error.

The response envelope: `data` (schema-shaped; ungrounded fields are `null`),
`fields` (per-leaf `{confidence, method, page}` provenance — `method` is
`"deterministic"`, `"model"`, or `"ocr"`), `warnings` (every null field;
scanned pages and what OCR would cost), `extraction_method`,
`credits_charged`, `pages`, `id`. **Relay warnings and null fields honestly**
— Inkrun deliberately returns nulls instead of invented values.

### `extract_tables` — every table as data

Returns each recoverable table as `rows` (arrays of cell strings) plus a
`csv` rendering, tagged `{page, index}`. Purely deterministic by default; a
table whose grid can't be recovered produces a warning naming its page
instead of a half-guessed grid. Takes no preset/schema.

### OCR is opt-in, never automatic

Scanned pages (no text layer) yield nothing at the cheap tiers. `extract`
and `extract_tables` accept `ocr_fallback: true` — but default false is
safe: without the flag you get cheap partial results plus a warning saying
what OCR would recover and cost. Set it only after `inspect_document` shows
scanned pages. OCR values are labelled `method: "ocr"` with confidence
capped at 0.8 (pixels can't be verified against text), and OCR refuses more
than 15 scanned pages per call.

## Round trip: `extract_and_generate`

The one-call compose of extract + create_pdf — an ugly supplier invoice goes
in, your letterhead comes out. Use it when the user wants a document's data
*re-presented* rather than just extracted. Send the document (same contract
as `extract`), exactly one of `preset` | `schema`, and `template_markdown` —
a Markdown template with `{{field}}` placeholders. Styling (`template`,
`theme`, `format`, `title`) works exactly as in `create_pdf`.

Placeholder rules:

- `{{field_name}}` substitutes the extracted value; **nested fields flatten
  with underscores** (`vendor.name` → `{{vendor_name}}`). Write `\{{` for a
  literal `{{`.
- Substituted values are **inert text** — markdown/HTML in extracted data
  can never restyle or inject into the template.
- Extracted values also land in the frontmatter value space, so cover-page
  bindings can read them (e.g. `title: "Invoice {{invoice_number}}"`).
- A placeholder whose field was not extracted renders as a visible blank
  `____` plus an entry in `substitution.missing` — never an invented value.
  **Check `substitution.missing` and tell the user what came up blank.**

Billing is atomic: extraction credits + 1 render credit reserve together and
refund together — a round trip is never half-billed; you're charged for the
extraction tier actually used + 1. Tip: while designing a template, run
`extract` alone first to see which field names the document actually yields.

## Form filling: `fill_form`

Fill a FILLABLE PDF (AcroForm) from JSON — government forms, insurance
claims, onboarding packets. Workflow:

1. `inspect_document` (free) — its `form.fields` inventory lists every
   field's exact name and type (names are case-sensitive and often dotted).
2. `fill_form` with the document and `fields`: a map of field name → value,
   e.g. `{"applicant.name": "Ada Lovelace", "agree": true, "country": "PT"}`.

Field types: text (string/number), checkbox (`true`/`false` or yes/no),
radio/dropdown/optionlist (value must be one of the field's options — an
invalid option returns a warning listing the valid ones). Signature/button
fields can't be filled. Unknown field names produce **warnings, never silent
drops** — check `warnings` to verify everything landed. XFA (legacy Adobe
XML) forms are rejected as unsupported. The filled output is stored and
returned as a presigned `url` like a render; the source form bytes are never
stored. Costs 1 credit. Caveat: some producers' forms show stale field
appearances in strict viewers — values are always set, and NeedAppearances
is flagged so conforming viewers redraw them.

## Credit costs

| Operation | Credits |
| --- | --- |
| Render (`create_pdf`) | 1 |
| `inspect_document` | **Free** |
| Deterministic extraction | 1 per 20 pages |
| Model-tier extraction | 3 up to 5 pages, +1 per additional 5 |
| OCR tier (opt-in) | 10 up to 5 pages, +2 per additional 5 |
| `extract_and_generate` | extraction tier used + 1, reserved atomically |
| `fill_form` | 1 per filled PDF |

You are only charged for the tier actually used; over-quota fails before
any work; failed operations are refunded. Monthly credits: Free 20, Pro 500
(+ metered overage), Team 5,000 (+ overage).

## Privacy

Source documents sent for inspect/extract/generate/fill are **processed in
memory and never stored** — page images included. Only the structured
extraction result, and any PDF Inkrun produces (renders, round-trip outputs,
filled forms), are retained in the account history.

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
`INKRUN_API_KEY`). Keep the key server-side; the API rejects cross-origin
browser reads.

### Render: `POST https://inkrun.dev/api/v1/render`

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

### Extraction endpoints

Same auth; bodies mirror the MCP tools (`file_url` | `file_base64`, plus
tool-specific fields):

```bash
# Free inspection:
curl "https://inkrun.dev/api/v1/inspect" \
  -H "Authorization: Bearer $INKRUN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "file_url": "https://example.com/invoice.pdf" }'

# Structured extraction (preset XOR schema):
curl "https://inkrun.dev/api/v1/extract" \
  -H "Authorization: Bearer $INKRUN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "file_url": "https://example.com/invoice.pdf", "preset": "invoice" }'

# All tables:
curl "https://inkrun.dev/api/v1/extract/tables" \
  -H "Authorization: Bearer $INKRUN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "file_url": "https://example.com/report.pdf" }'

# Round trip (extract → {{field}} template → themed PDF):
curl "https://inkrun.dev/api/v1/generate" \
  -H "Authorization: Bearer $INKRUN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "file_url": "https://example.com/invoice.pdf", "preset": "invoice",
        "template_markdown": "# Payment advice\n\nVendor: {{vendor}}\nTotal: {{total}}" }'

# Fill an AcroForm PDF:
curl "https://inkrun.dev/api/v1/fill" \
  -H "Authorization: Bearer $INKRUN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "file_url": "https://example.com/form.pdf",
        "fields": { "applicant.name": "Ada Lovelace", "agree": true } }'
```

Both `/generate` and `/fill` accept an `Idempotency-Key` header and honour
`Accept: application/pdf` for raw bytes, like `/render`.

Reference: https://inkrun.dev/docs/api/extract

## Connecting the MCP server

- **OAuth clients** (claude.ai, Claude Desktop, ChatGPT): add a custom
  connector with `https://inkrun.dev/api/mcp` and sign in once.
- **Header-based clients** (Claude Code, Cursor, VS Code, scripts):

  ```bash
  claude mcp add --transport http inkrun https://mcp.inkrun.dev/mcp \
    --header "Authorization: Bearer sk_..."
  ```

## Good to know

- Every metered operation counts credits against the monthly allowance
  (Free: 20, Pro: 500 + overage); a failed operation is refunded and
  inspection is always free.
- `create_pdf` requires an account (API key or OAuth) — renders are
  attributed and stored; anonymous rendering is not available over MCP.
- Charts render as static inline SVG — deterministic, no interactivity.
- Frontmatter `title:` doubles as the stored document title when no explicit
  `title` is passed.
- Extraction results are grounded, not generated: prefer relaying a `null`
  plus its warning over filling gaps yourself — if you do infer a missing
  value from context, say so explicitly.
