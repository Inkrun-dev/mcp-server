# Security policy

## Reporting a vulnerability

If you believe you have found a security vulnerability in the Inkrun MCP server, the REST API, or the Inkrun service itself, please report it privately — **do not open a public issue**.

- Email **security@inkrun.dev** with a description of the issue, steps to reproduce, and the potential impact.
- You will receive an acknowledgement within 5 business days.
- Please give us a reasonable window to investigate and remediate before any public disclosure.

## Scope

In scope:

- The MCP endpoints (`https://inkrun.dev/api/mcp`, `https://mcp.inkrun.dev/mcp`) and their OAuth / API-key authentication flows
- The REST API (`https://inkrun.dev/api/v1/*`)
- The PDF render pipeline (e.g. content injection into rendered documents)
- Account, quota, or template access-control issues

Out of scope:

- Denial-of-service / volumetric attacks and rate-limit probing
- Issues in third-party MCP clients (Claude, ChatGPT, Cursor, …) — report those to their vendors
- Social engineering or physical attacks

## Good faith

We will not pursue action against researchers who act in good faith: access only your own account's data, avoid privacy violations and service disruption, and report findings promptly.
