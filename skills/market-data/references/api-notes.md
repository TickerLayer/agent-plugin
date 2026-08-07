# TickerLayer MCP — API notes

Condensed reference for the TickerLayer MCP server at
`https://mcp.tickerlayer.com/mcp`. The tool input schemas served by the MCP server
are authoritative; these notes cover the conventions and the long tail. Full
agent-facing documentation: https://tickerlayer.com/llms-full.txt

## Tools and parameters

The five data tools (`get_quote`, `get_last_trade`, `get_snapshot`,
`get_previous_close`, `get_history`) all take **`asset_class` (required)** —
one of `crypto`, `forex`, `stocks`, `indices`, `etfs`, `commodities` — alongside
`symbol`. None of them accepts a bare symbol, so pick the asset class deliberately:
`XAUUSD` (commodities) and `EURUSD` (forex) look alike but resolve differently.
(`get_bond_yield` is the one tool that takes `symbol` alone.)

### get_quote

- `asset_class` (required) — `crypto`, `forex`, `stocks`, `indices`, `etfs`,
  `commodities`.
- `symbol` (required) — any TickerLayer symbol except bonds.
- Returns the latest **bid/ask** only: `bid`, `ask`, `bid_size`, `ask_size`,
  `timestamp` (Unix ms). It does **not** return a last price, change, or percent
  change — those come from `get_snapshot`.

### get_last_trade

- `asset_class` (required), `symbol` (required).
- Returns the most recent trade: `price`, `size`, `timestamp` (Unix ms). Prefer
  `get_snapshot` unless the user specifically wants trade-level detail.

### get_snapshot

- `asset_class` (required), `symbol` (required).
- One call for: `bid`, `ask`, `bid_size`, `ask_size`, `last_price`, `last_size`,
  `last_timestamp`, `prev_close`, `change`, `change_percent`. The default tool for
  "how is X doing". Note it carries the previous close, not intraday
  open/high/low — for the day's range use `get_history`.
- Individual fields are null when that component is unavailable (e.g. no recent
  trade, or no settled previous close); report what is present rather than
  inferring a missing value.

### get_previous_close

- `asset_class` (required), `symbol` (required).
- `interval` (optional) — exactly `1m`, `5m`, `15m`, `1h`, `4h`, `1d`. Omit it for
  the previous completed **daily** bar. With an interval, returns the **latest
  fully settled bar** of that interval instead, along with its `bar_start` /
  `bar_end` window. Settled means the bar's window has closed; a live,
  still-forming bar is never returned.

### get_history

- `asset_class` (required), `symbol` (required).
- **There is no `interval` parameter here.** The bar size is `multiplier` +
  `timespan`:
  - `multiplier` (required, integer) — `1`, `5`, `15` with `minute`; `1`, `4` with
    `hour`; `1` with `day`. No other pair is valid.
  - `timespan` (required) — `minute`, `hour`, or `day`.
- `from`, `to` (both required) — UTC dates, `YYYY-MM-DD`, inclusive, `to >= from`.
  The range is inclusive of the final requested session for daily bars.
- `limit` (optional) — 1–5000 bars, default 500.
- `sort` (optional) — `asc` (oldest first) or `desc` (newest first), default
  `desc`.
- Returns OHLCV bars; bar timestamps are Unix ms marking the bar open.
- **Plan depth caveat:** how far back history reaches is plan-limited and varies by
  bar size (intraday has shorter depth than daily). A 403 or a range truncated at
  the plan boundary is a plan limit, not an error to work around.

### list_symbols

- `asset_class` (required) — one of `crypto`, `forex`, `stocks`, `indices`,
  `etfs`, `commodities`.
- `search` (optional) — case-insensitive substring match on the symbol **or** the
  name; resolves fuzzy names.
- `limit` (optional) — 1–500 entries, default 100.
- Returns `{ matched, returned, data }`, where `matched` is the pre-limit count.
- **There is no `market` / country parameter** and no pagination beyond `limit`.
  To approximate a country filter for stocks, search the symbol prefix:
  `asset_class="stocks", search="DE:"`. If `matched` exceeds `returned`, narrow the
  search string rather than expecting a next page.

### get_market_status

The only market tool with the two include flags.

- `asset_class` (optional) — `crypto`, `forex`, `stocks`, `indices`, `etfs`,
  `commodities`, or `cfds`. (`asset` is a legacy alias for the same thing; prefer
  `asset_class`.)
- `symbol` (optional) — resolve status for one symbol.
- `market` (optional) — region/market code: `US`, `GB`, `DE`, `JP`, `SA`, ...
- `include_sessions` (optional boolean) — include session phases in the response.
- `include_holiday` (optional boolean) — include holiday context in the response.
- Call with **no arguments** for a global overview (crypto, forex, stock regions).
  Returns open/closed/pre-market/post-market state per region.

### get_market_sessions

- `asset_class` (optional, alias `asset`), `symbol` (optional), `market`
  (optional) — all three are optional; `market` is *not* required.
- `date` (optional) — UTC `YYYY-MM-DD`, default today.
- Returns pre-market / primary / post-market windows where the market has them.
- No `include_holiday` flag here — that lives on `get_market_status`.

### get_market_holidays

- `asset_class` (optional, alias `asset`), `symbol` (optional), `market`
  (optional) — `market` is *not* required.
- `year` (optional) — four digits, e.g. `2026`.
- `start`, `end` (optional) — range bounds, `YYYY-MM-DD`.
- Returns full closures and early closes.
- No `include_sessions` flag here — that lives on `get_market_status`.

### get_bond_yield

- `symbol` (required) — `COUNTRY:TENOR`, e.g. `US:10Y`, `DE:10Y`, `US:3M`. This is
  the only tool that takes no `asset_class`.
- Returns the latest yield: `rate`, `unit` (`percent`), `date`, and a Unix-ms
  `timestamp`. Yields are rates, not prices; do not feed bond symbols to the
  quote/snapshot tools.

## Bond tenor coverage

Countries are exactly `DE`, `ES`, `FR`, `IT`, `UK`, `US`.

- **US:** `1M`, `3M`, `6M`, `1Y`, `2Y`, `3Y`, `5Y`, `7Y`, `10Y`, `20Y`, `30Y` —
  money-market tenors plus the full coupon curve.
- **DE, ES, FR, IT, UK:** `1Y`, `2Y`, `3Y`, `5Y`, `10Y`, `30Y` only. No sub-year
  tenors, and no `7Y` or `20Y` — those are US-only.
- **UK gilts use `UK`, not `GB`.** `GB:10Y` returns 404. (`GB` is still the correct
  region code for the UK *stock* market in `list_symbols` symbols and the market
  calendar tools — the `UK` spelling is bonds-only.)
- If a tenor 404s, list what is available rather than interpolating a yield.

## Indices and ETF code style

Indices use compact region-plus-number codes: `US500`, `US100`, `US30`, `DE40`,
`GB100`, `JP225`, `EU50`. The ETF tracking an index is the index code with an
`ETF` suffix: `US500ETF`. Index values are levels (no volume in the usual sense);
the ETF is the tradable instrument with real volume. Match the user's intent:
benchmark questions → index, fund questions → ETF.

## Symbols, timestamps, limits — quick recap

- Stocks are always `COUNTRY:TICKER` (`US:AAPL`, `DE:SAP`, `SA:2222`). Crypto,
  forex, and commodities are plain pairs (`BTCUSD`, `EURUSD`, `XAUUSD`).
- Timestamps are Unix **milliseconds** unless the field says otherwise;
  `get_history` `from`/`to` are UTC `YYYY-MM-DD` dates.
- 401 = connect account / configure key. 403 = plan does not include the data —
  stop, report, never fabricate endpoints. 429 = back off.
- The MCP tools are the complete surface. There are no hidden REST routes to
  discover, and attempting them will not produce data.

## Positioning (applies to every answer built on this data)

TickerLayer data is derived and indicative, not exchange-official. Never name
upstream providers or exchanges as the source, and never turn the data into
investment advice.
