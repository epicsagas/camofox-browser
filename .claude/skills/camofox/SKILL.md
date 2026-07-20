---
name: camofox
description: Anti-detection browser automation via the local camofox MCP server. Use when the user wants to browse, search, scrape, fill forms, or interact with sites that block bots (Google, Amazon, LinkedIn, Reddit, Cloudflare-protected pages). Routes through the camofox-browser MCP tools instead of Playwright/Chrome.
---

# camofox — Anti-Detection Browsing via MCP

Reaches the **camofox-browser** REST server (`localhost:9377`) through its MCP
tools (`mcp__camofox-browser__*`). Use this instead of Playwright/built-in
browser tools — Camoufox spoils fingerprints at the C++ level, so it bypasses
bot detection on Google, Cloudflare, LinkedIn, etc.

## Prerequisites

1. **camofox REST server running** — in the `camofox-browser` repo:
   ```bash
   npm install          # downloads Camoufox binary (~300MB) on first run
   npm start            # → http://localhost:9377
   ```
2. **MCP server registered in Claude Code** — see `mcp-setup.md` in this skill
   folder. Verify with `/mcp` — you should see `camofox-browser` connected.
3. If both are up, the `mcp__camofox-browser__*` tools appear automatically.
4. **Only for `camofox_import_cookies`** — set the same `CAMOFOX_API_KEY` on
   both the REST server (`CAMOFOX_API_KEY=xxx npm start`) and the MCP server
   (`--env CAMOFOX_API_KEY=xxx`). This key is **not** an external service key —
   it's a secret you choose to gate the cookie-import route. On localhost +
   non-production it's optional (loopback is allowed without it), but for any
   remote/production exposure both sides must match. Other tools never need it.

## Core workflow (almost every task)

Every camofox session follows the same shape — **snapshot before you act**:

```mermaid
graph LR
    A[create_tab<br/>with URL] --> B[snapshot<br/>get element refs e1,e2...]
    B --> C{interact}
    C -->|click/type| D[click or type<br/>by ref e1, e2]
    D --> B
    C -->|read result| E[snapshot again<br/>see new state]
    E --> F[close_tab<br/>when done]
```

**Critical rule**: never guess selectors. Always `snapshot` first to get the
`e1`, `e2` element refs, then pass that ref to `click`/`type`. Refs are stable
within a tab until the page navigation changes them — re-snapshot after any
`navigate`.

## Available tools (`mcp__camofox-browser__*`)

| Tool | When |
|------|------|
| `camofox_create_tab` | Open a URL → returns `tabId` |
| `camofox_snapshot` | Read page state + element refs (+ screenshot). Paginate with `offset` when `hasMore=true` |
| `camofox_navigate` | Go to URL **or** use search macro (`@google_search`, `@reddit_search`, ...) |
| `camofox_click` | Click by ref (`e1`) or CSS selector |
| `camofox_type` | Type text into a ref/selector, optional `pressEnter` |
| `camofox_scroll` | up/down/left/right by pixels — **unreliable, prefer `evaluate`** (see Recipes) |
| `camofox_screenshot` | Standalone screenshot (snapshot already includes one) |
| `camofox_evaluate` | Run JS in page context — extract data, call page APIs |
| `camofox_list_tabs` | See what's open in this session |
| `camofox_close_tab` | Free resources when done |
| `camofox_import_cookies` | Authenticate via Netscape cookie file (needs `CAMOFOX_API_KEY`) |

## Recipes

### Web search (one call via macro)
```
create_tab({ url: "about:blank" })          → tabId
navigate({ tabId, macro: "@google_search", query: "rust async runtime" })
snapshot({ tabId })                          → read result refs
```

### Extract structured data from a page
```
create_tab({ url: "https://example.com/products" })
snapshot({ tabId })                          → locate items
evaluate({ tabId, expression: `
  Array.from(document.querySelectorAll('.product')).map(p => ({
    name: p.querySelector('.name')?.textContent,
    price: p.querySelector('.price')?.textContent,
  }))
` })
```

### Login via cookies (LinkedIn etc.)
```
import_cookies({ cookiesPath: "/path/to/cookies.txt", domainSuffix: "linkedin.com" })
create_tab({ url: "https://linkedin.com/feed" })
snapshot({ tabId })                          → authenticated view
```

### Paginate a huge page
```
snapshot({ tabId })           → hasMore:true, nextOffset:12345
snapshot({ tabId, offset: 12345 })   → next chunk
```

### Scroll a page (use evaluate, not camofox_scroll)
`camofox_scroll` fires `mouse.wheel` from the viewport top — it silently no-ops on
lazy-load / virtual-scroll sites (Wikipedia, Twitter, Reddit) and on
`overflow:scroll` containers. Use `camofox_evaluate` instead — it's reliable:
```
# absolute position
evaluate({ tabId, expression: "window.scrollTo(0, 1500)" })
# relative
evaluate({ tabId, expression: "window.scrollBy(0, 800)" })
# to bottom (triggers lazy load)
evaluate({ tabId, expression: "window.scrollTo(0, document.body.scrollHeight)" })
# check position
evaluate({ tabId, expression: "String(window.scrollY)" })
```

## When NOT to use this skill

- **Writing/analysis tasks** that only need fetched text → use `agent-reach`
  or `WebFetch` (lighter, no browser).
- **Internal/test pages with no bot detection** → plain Playwright is faster.
- **Posting/commenting/liking** → camofox is read + interact; don't automate
  spammy write actions.

## Troubleshooting

- **`MCP tool not found`** → server not registered; run `/mcp` and check
  `mcp-setup.md`.
- **`503 session_expired` / `tab create timed out`** → the REST server's
  browser session died. Restart `npm start`, or check `BROWSER_IDLE_TIMEOUT_MS`
  isn't too aggressive.
- **`400 userId and sessionKey required`** → MCP wrapper bug (should send both).
  Confirm you're on a build that includes the `sessionKey` fix.
- **`camofox_scroll` returns `{ok:true}` but page doesn't move** → expected on
  lazy-load / virtual-scroll pages; the server's `mouse.wheel` no-ops there. Use
  `evaluate` with `window.scrollTo` / `scrollBy` (see Recipes).
- **`422 strict mode violation ... resolved to N elements`** on `click` → your
  CSS selector matched multiple elements. Re-`snapshot` and click by element
  ref (`e1`, `e2`) instead — refs are unambiguous.
- **`403 Forbidden` on `camofox_import_cookies`** → either the server has
  `CAMOFOX_API_KEY` set and your MCP env doesn't match it, or you're hitting a
  remote/production server without the key. Set the same key on both sides.
