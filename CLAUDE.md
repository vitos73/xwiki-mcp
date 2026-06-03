# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MCP (Model Context Protocol) server that bridges AI agents (Claude Code, OpenClaw, Gemini CLI, etc.) with a corporate XWiki instance via REST API. TypeScript, stdio transport, built on `@modelcontextprotocol/sdk`.

Published as `xwiki-mcp` on npm. Current version: **0.2.0**.

## Commands

```bash
npm install            # Install dependencies
npm run build          # Compile TypeScript to dist/
npm run dev            # Run with tsx (no build step)
npm test               # vitest (15 tests)
npm publish            # Publishes to npm — prepublishOnly runs build + tests
```

Stdio MCP server — reads JSON-RPC from stdin, writes to stdout. Logs go to stderr (so they don't pollute the protocol stream).

## Architecture

```
src/
├── index.ts           # MCP server init, tool registration (9 tools)
├── config.ts          # Env var parsing/validation
├── client.ts          # XWiki REST client — Solr + legacy search, read + write
├── client.test.ts     # 15 unit tests (mocked fetch)
├── types.ts           # Shared TypeScript interfaces
└── tools/
    ├── list-spaces.ts        ┐
    ├── list-pages.ts         │ READ
    ├── get-page.ts           │
    ├── get-page-children.ts  │
    ├── get-attachments.ts    │
    ├── search.ts             ┘  Solr by default, engine="legacy" as fallback
    ├── create-page.ts        ┐
    ├── delete-page.ts        │ WRITE (v0.2+)
    └── add-comment.ts        ┘
```

## Configuration (Environment Variables)

```
XWIKI_BASE_URL      # Required. Base URL without /rest (e.g. https://wiki.example.com)
XWIKI_AUTH_TYPE     # basic | token | none  (default: basic)
XWIKI_USERNAME      # For basic auth
XWIKI_PASSWORD      # For basic auth
XWIKI_TOKEN         # For token auth (Bearer)
XWIKI_WIKI_NAME     # Wiki name (default: xwiki)
XWIKI_REST_PATH     # REST path (default: /rest)
XWIKI_PAGE_LIMIT    # Default page size (default: 50)
```

## XWiki REST API Specifics

- **JSON mode:** Use `?media=json` or `Accept: application/json`
- **Pagination:** params `start` + `number` (NOT `limit`). The tool surface exposes `limit` and maps it to `number`.
- **Nested spaces:** dot notation in the tool API (`Foo.Bar`) → REST path `/spaces/Foo/spaces/Bar`
- **Default page size:** 50 (XWiki max is 1000; keep low to save tokens)

### Search endpoints — important

XWiki exposes two different search endpoints:

| Endpoint | Backend | Searches | Default in v0.2+ |
|----------|---------|----------|------------------|
| `/rest/wikis/{wiki}/search` | HQL over DB | page **name** mostly — content matches are unreliable | `engine: "legacy"` |
| `/rest/wikis/query?type=solr` | Solr index | full-text over title + content, ranked by score | `engine: "solr"` (default) |

`searchSolr` builds `q` as bare terms (or `title:(...)` / `name:(...)` when scope is set), adds `space:"..."` filter when a space is given, calls `/rest/wikis/query?type=solr&wikis={wiki}`.

## Tool design principles

Tool descriptions are written for **less-capable agents** (Gemini, smaller LLMs). Each description has:

1. **First line:** what it does + trigger verbs the user might say (incl. Russian: "запиши в вики", "видимость", "документация").
2. **"Use this when…"** block with concrete scenarios.
3. **"Next step"** hint pointing to the natural follow-up tool.
4. **Param examples:** how to format `space` (dot notation), how to read `page` from a search result id.
5. **Negative guidance** on destructive tools — `delete_page` explicitly says "do NOT call this for cleanup/refactor; for overwrite use `create_page`".
6. **XWiki markup cheatsheet** inline on `create_page` (`= H1 =`, `**bold**`, `[[label>>page]]`, `{{code}}`).

The `search` description is the most important — it's marked as the **PRIMARY ENTRY POINT** with an explicit "DO NOT guess page paths" rule. This is the failure mode we observed with the previous implementation: agents would skip search entirely and start brute-forcing `Main.WebHome`, `Documentation.WebHome` etc.

## Tool Response Format

- Compact JSON with only useful fields (XWiki XML wrappers stripped).
- Paginated results wrap in `{ ..., _pagination: { total, start, limit, has_more } }`.
- Page URLs are full web URLs (`/bin/view/...`), not REST endpoints. Solr responses without a `hierarchy` block get a synthesised URL from `pageFullName`.
- Search responses include the actual `engine` used so callers can verify which backend answered.

## Error Handling Convention

| HTTP Status | Message |
|---|---|
| 404 | `"Not found: {path}"` (then `get_page` rewrites to `"Page not found: {space}/{page}"`) |
| 401/403 | `"Authentication failed. Check XWIKI_AUTH_TYPE and credentials."` |
| Other 5xx | `"XWiki server error: {status}. URL: {url}. Body: {first 200 chars}"` |
| Network | `"Cannot connect to XWiki at {baseUrl}. Check XWIKI_BASE_URL."` |

Errors are surfaced via `{ content: [{ type: 'text', text: msg }], isError: true }` — the MCP server itself never crashes.

## Write operations

All write tools use XWiki REST PUT/POST with XML bodies. The `escapeXml` helper in `client.ts` escapes `& < > " '` so user-supplied titles/content with special chars don't break the request — this was a bug in the reference implementation we replaced.

`create_page` is **upsert** — same call for create and update (XWiki semantics, not a quirk of this server).

## References

- XWiki REST API: https://www.xwiki.org/xwiki/bin/view/Documentation/UserGuide/Features/XWikiRESTfulAPI
- MCP TypeScript SDK: https://github.com/modelcontextprotocol/typescript-sdk
- npm: https://www.npmjs.com/package/xwiki-mcp
- GitHub: https://github.com/vitos73/xwiki-mcp
