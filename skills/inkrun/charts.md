# Inkrun chart fence — full syntax

Add a fenced code block tagged `chart` to the Markdown passed to `create_pdf`
(or the REST `markdown` field). It renders as a static inline SVG chart sized
to the page.

````markdown
```chart
type: bar
x: Quarter
y: Revenue, Costs
title: 2026 Performance
stacked: false
---
| Quarter | Revenue | Costs |
| ------- | ------- | ----- |
| Q1      | 120     | 80    |
| Q2      | 180     | 95    |
```
````

## Structure

1. A settings block of `key: value` lines (keys case-insensitive).
2. A `---` separator line.
3. A GitHub-style Markdown table: first row = headers, rest = data rows.

## Settings

| Key | Value |
| --- | --- |
| `type` | `bar` \| `line` \| `area` \| `pie` (default: `bar`) |
| `x` | category column header — **required**; must match a table header (x axis for bar/line/area, slice label for pie) |
| `y` | one or more value column headers, comma-separated — **required**; each must match a header. Pie uses only the first column. |
| `title` | optional chart title |
| `xLabel` | optional x-axis title |
| `yLabel` | optional y-axis title |
| `height` | plot height in px (clamped 160–760, default 340) |
| `stacked` | `true` \| `yes` \| `1` to stack (bar/area only; ignored for line/pie) |

## Rules

- Every cell under a `y` column must be numeric. Thousands separators and
  trailing units like `%` or `$` are tolerated; any non-numeric y cell makes
  the **whole fence fall back to a plain code block**.
- At least one data row is required.
- Charts are static SVG at a fixed width that fits the page — offline,
  deterministic, no interactivity.
- Raw HTML/SVG in Markdown is not supported — the `chart` fence is the only
  way to add a chart.
