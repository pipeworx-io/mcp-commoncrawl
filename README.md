# @pipeworx/commoncrawl

Common Crawl's public web archive — find every time a URL was crawled since 2008
and read back the exact page bytes that were captured.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1721+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `commoncrawl_crawls(limit?, contains_date?)` — the monthly crawl collections
  (`CC-MAIN-2026-34` and friends) with the capture window each one covers.
  `contains_date` answers "which snapshot would have seen my page on that day".
- `commoncrawl_index_search(url, crawl?, match_type?, limit?, page?, filter?, from?, to?)` —
  CDX index lookup for a URL, host or whole domain. Returns capture timestamp,
  HTTP status, MIME type, detected language, content digest, and the WARC
  `filename` + `offset` + `length` that the next tool needs.
- `commoncrawl_fetch_record(filename, offset, length, max_body_bytes?)` — reads
  one archived record back by byte range: WARC headers, the captured HTTP
  response headers, and the page body as crawled.

## Auth

Keyless.

## Data sources

- <https://index.commoncrawl.org/collinfo.json> — the crawl collection list.
- <https://index.commoncrawl.org/{crawl}-index> — the CDX index, JSON-lines.
- <https://data.commoncrawl.org/{warc path}> — WARC records, HTTP range reads.

## Things the next person would otherwise rediscover

- **The CDX host 502s sporadically under load.** Measured 2026-09-17: the same
  `url=example.com` query alternated between 200 and nginx 502 inside a minute,
  so it is load, not the query. `ccFetch` retries a 5xx twice with a short
  backoff. Without that a caller reads transient nginx noise as "Common Crawl
  has no record of this URL", which is a different and wrong answer.
- **"No captures" is a 404 with an English sentence, not JSON.** Handled as an
  empty result set with an explicit `note`, so an empty `captures` array is
  never silently indistinguishable from an upstream failure.
- **Each indexed record is its own gzip member**, so a byte-range read of
  `offset`..`offset+length-1` decompresses standalone — no need to stream the
  whole 1 GB WARC. `DecompressionStream('gzip')` handles it in the Workers
  runtime.
- **`new TextDecoder('utf-8', { fatal: false })` does not typecheck** against
  `@cloudflare/workers-types`: its `TextDecoderConstructorOptions` requires
  `ignoreBOM` too. Pass no options; non-fatal is the default anyway.
- A broad `match_type: "domain"` query over a large domain is expensive upstream
  and is the first thing to 502. Narrow with `match_type: "host"` plus a
  `filter` such as `=status:200`.

## Related packs

`crawlgraph` is our own link graph over sites we crawl. This pack reads the
Common Crawl Foundation's corpus — different data, different questions.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "commoncrawl": {
      "url": "https://gateway.pipeworx.io/commoncrawl/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/commoncrawl/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1721+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/commoncrawl_crawls \
  -H 'Content-Type: application/json' \
  -d '{"limit":3}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/commoncrawl_crawls`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "commoncrawl": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-commoncrawl"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-commoncrawl
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Commoncrawl data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
