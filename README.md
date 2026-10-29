# @pipeworx/poshmark-comps

Sold-price and active-listing search on the public Poshmark marketplace —
the best open sold-price comp data on Poshmark, with no login and no API key.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `poshmark_sold_search(query, department?, sort_by?, limit?)` — search SOLD
  listings, returns price, size, brand, condition, sold date, seller, and a
  listing URL per hit.
- `poshmark_active_search(query, department?, sort_by?, limit?)` — same shape
  for currently-listed (not yet sold) items, for comparing ask vs. sold.

## Auth

Keyless. No login, no API key, no anti-bot challenge encountered (verified
live 2026-09-05).

## Data sources

- <https://poshmark.com/search> — Poshmark's own public search page. There is
  no separate JSON API endpoint: the page is server-rendered by a Vue app and
  embeds the full result set in a `window.__INITIAL_STATE__` script-tag blob
  (checked live: an XHR/`Accept: application/json` request returns the same
  full HTML page, not JSON). This pack fetches that HTML and extracts the
  embedded JSON directly — cheaper and more robust than DOM parsing.

Traps for the next person:

- One page per call, up to ~48 results (Poshmark's own default page size).
  `next_max_id` in the response is a real pagination cursor but this pack
  doesn't walk it yet — `total_matches_reported_by_poshmark` in the tool
  output tells the caller how much more exists.
- `price` on a sold item is the **listing price** Poshmark shows on the sold
  post (what the seller was asking), not a separately verified receipt/hammer
  amount, and excludes Poshmark's fee and shipping. State this to callers —
  it's in `coverage_note` on every response.
- `sort_by` only reliably takes `best_match` (default), `price_desc`,
  `price_asc` — `newest` and `most_loved` returned empty result sets when
  probed live 2026-09-05; not exposed as options for that reason.
- `brand`/`size` query params on the site did not reliably filter when probed
  (results included non-matching brands); free-text `query` does the real
  filtering, so brand/size aren't exposed as separate parameters — put them in
  `query` instead (e.g. `"lululemon align size 6"`).
- Crawl politely: one page fetch per tool call, no pagination loop, no
  concurrent fan-out.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "poshmark-comps": {
      "url": "https://gateway.pipeworx.io/poshmark-comps/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/poshmark-comps/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1856+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/poshmark_sold_search \
  -H 'Content-Type: application/json' \
  -d '{"query":"lululemon align","sort_by":"relevance","limit":10}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/poshmark_sold_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-poshmark-comps` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/poshmark-comps/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "poshmark-comps": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-poshmark-comps"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Poshmark Comps data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
