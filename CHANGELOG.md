# Changelog

## 1.0.1 (2026-10-04)

- Listing text: a clearer short description and README introduction.

## 1.0.0 (2026-10-04)

- The skill now covers all 12 tools on the hosted MCP server, including
  `open_markets_board`, the interactive markets board with charts and search,
  and `search_symbols`, which powers the board's search box.
- Perpetual futures: the `perpetuals` asset class, USDT-quoted contract codes,
  24/7 trading, `mark_price`, and the plan they need.
- More accurate guidance: history span limits per request, paging and sort
  order, market calendar times and local dates, what the session and holiday
  tools need, the 400, 403, 429 and 503 cases, ETF codes, `UK100` for the UK
  index, and `UK` as an alias of `GB` in the calendar tools.
- Claude directory listing details: display name, icon, documentation,
  support, privacy policy and terms of use links, and keywords.
- README rewritten as the listing description: install steps for Claude,
  including connecting the bundled connector, and what the plugin sends and
  where.

## 0.1.0 (2026-08-07)

Initial release.

- Agent Plugins 1.0.0 manifest (`plugin.json`) and MCP configuration
  (`mcp.json`) for the hosted TickerLayer MCP server
  (`https://mcp.tickerlayer.com/mcp`, Streamable HTTP, OAuth or API key).
- `market-data` skill: tool selection, symbol conventions, discovery-first
  workflow, market-calendar usage, error handling, timestamp semantics, and
  data-positioning guidance, with a deeper parameter reference in
  `skills/market-data/references/api-notes.md`.
- Claude Code compatibility files (`.claude-plugin/plugin.json`,
  `.claude-plugin/marketplace.json`, `.mcp.json`) and Codex marketplace file
  (`.agents/plugins/marketplace.json`).
