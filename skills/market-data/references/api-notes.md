# TickerLayer MCP: API notes

Condensed reference for the TickerLayer MCP server at
`https://mcp.tickerlayer.com/mcp`. The tool input schemas served by the MCP
server are authoritative; these notes cover the conventions and the long tail.

## Tools and parameters

The five data tools (`get_quote`, `get_last_trade`, `get_snapshot`,
`get_previous_close`, `get_history`) all take **`asset_class` (required)**, one
of `crypto`, `forex`, `stocks`, `indices`, `etfs`, `commodities`,
`perpetuals`, alongside `symbol`. None of them accepts a bare symbol, so pick
the asset class deliberately: `XAUUSD` (commodities), `XAUUSDT` (perpetuals)
and `EURUSD` (forex) look alike but resolve differently. `get_bond_yield` is
the one data tool that takes `symbol` alone.

### get_quote

- `asset_class` (required), `symbol` (required): any TickerLayer symbol except
  bonds.
- Returns the latest **bid/ask** only: `bid`, `ask`, `bid_size`, `ask_size`,
  `timestamp` (Unix ms). It does **not** return a last price, change, or
  percent change; those come from `get_snapshot`.

### get_last_trade

- `asset_class` (required), `symbol` (required).
- Returns the most recent trade: `price`, `size`, `timestamp` (Unix ms).
  `size` can be null for some classes. Prefer `get_snapshot` unless the user
  wants trade-level detail.

### get_snapshot

- `asset_class` (required), `symbol` (required).
- One call for `bid`, `ask`, `bid_size`, `ask_size`, `last_price`,
  `last_timestamp`, `prev_close`, `change` and `change_percent`, plus
  `last_size` when a trade size is known. Perpetual futures add `mark_price`
  and measure the change from the composite price rather than a last trade.
- It carries the previous close, not the intraday open, high and low; for the
  day's range use `get_history`.
- Individual fields are null when that component is unavailable (for example
  no recent trade, or no settled previous close). Report what is present
  rather than inferring a missing value.

### get_previous_close

- `asset_class` (required), `symbol` (required).
- `interval` (optional): exactly `1m`, `5m`, `15m`, `1h`, `4h`, `1d`. Omit it
  for the previous completed **daily** bar. With an interval, it returns the
  **latest fully settled bar** of that interval instead, with its `bar_start`
  and `bar_end` window. Settled means the bar's window has closed; a bar that
  is still forming is never returned.

### get_history

- `asset_class` (required), `symbol` (required).
- **There is no `interval` parameter here.** The bar size is `multiplier` plus
  `timespan`:
  - `multiplier` (required, integer): `1`, `5` or `15` with `minute`; `1` or
    `4` with `hour`; `1` with `day`. No other pair is valid.
  - `timespan` (required): `minute`, `hour` or `day`.
- `from`, `to` (both required): UTC dates, `YYYY-MM-DD`, inclusive, with
  `to` on or after `from`.
- `limit` (optional): 1 to 5000 bars, default 500.
- `sort` (optional): `asc` (oldest first) or `desc` (newest first), default
  `desc`.
- Returns OHLCV bars; each bar's `t` is the bar open in Unix ms, and `v` can be
  null where a class has no volume.
- **Span per request:** one request covers a limited date range, and the
  limit is tighter for smaller bars (minute bars cover weeks, daily bars cover
  many years). Perpetual futures allow shorter intraday ranges than the other
  classes. A range that is too large returns a 400 whose message names the
  maximum; split the range into consecutive requests.
- **Paging:** `next_offset` is non-null when more bars matched than `limit`
  returned. This tool has no offset parameter, so raise `limit` or narrow
  `from`/`to`. With the default `desc` sort, the newest bars come first.

### list_symbols

- `asset_class` (required): one of `crypto`, `forex`, `stocks`, `indices`,
  `etfs`, `commodities`, `perpetuals`.
- `search` (optional): case-insensitive substring match on the symbol **or**
  the name; resolves fuzzy names.
- `limit` (optional): 1 to 500 entries, default 100.
- Returns `{ matched, returned, data }`, where `matched` is the count before
  the limit. Rows carry extra fields by class, such as `market` for stocks.
- **There is no `market` or country parameter** and no pagination beyond
  `limit`. To approximate a country filter for stocks, search the symbol
  prefix: `asset_class="stocks", search="DE:"`. If `matched` exceeds
  `returned`, narrow the search string rather than expecting a next page.
- Stocks list only the markets the account's plan includes. An empty result
  for a stock can mean its market is not in the plan.

### get_market_status

The only market tool with the two include flags.

- `asset_class` (optional): `crypto`, `forex`, `stocks`, `indices`, `etfs`,
  `commodities` or `cfds`. `asset` is a legacy alias for the same thing;
  prefer `asset_class`. Perpetual futures have no market calendar.
- `symbol` (optional): resolve status for one symbol.
- `market` (optional): region or market code, such as `US`, `GB`, `DE`, `JP`
  or `SA`. `UK` is accepted as an alias of `GB`. An unknown code returns 400.
- `include_sessions` (optional boolean): include session phases in the
  response.
- `include_holiday` (optional boolean): holiday context is already included on
  narrowed calls, so this flag rarely matters.
- Call with **no arguments** for a global overview (crypto, forex and stock
  regions). Narrowed calls return the open, closed, pre-market or post-market
  state with `next_open` and `next_close`.
- `next_open`, `next_close` and session times are ISO 8601 UTC strings, not
  Unix ms; `local_time` uses the market's own offset.

### get_market_sessions

- `asset_class` (optional, alias `asset`), `symbol` (optional), `market`
  (optional). Pass `market` or `symbol`; `asset_class` alone works only for
  `crypto`, `forex` and `commodities`, and a call with none of them returns
  400.
- `date` (optional): the market's local date, `YYYY-MM-DD`, default today in
  that market's timezone. Late in the UTC day, Asian markets are already on
  the next date.
- Returns pre-market, primary and post-market windows where the market has
  them, with ISO 8601 UTC `start` and `end`.
- No `include_holiday` flag here; that lives on `get_market_status`.

### get_market_holidays

- `asset_class` (optional, alias `asset`), `symbol` (optional), `market`
  (optional). As with sessions, pass `market` or `symbol` unless the asset
  class is `crypto`, `forex` or `commodities`.
- `year` (optional): four digits, such as `2026`; default the current year.
- `start`, `end` (optional): `YYYY-MM-DD` bounds that filter within `year`.
  For a range that crosses New Year, query each year separately.
- Returns full closures and early closes, dated in the market's local
  calendar.
- No `include_sessions` flag here; that lives on `get_market_status`.

### get_bond_yield

- `symbol` (required): `COUNTRY:TENOR`, such as `US:10Y`, `DE:10Y` or `US:3M`.
  This is the only data tool that takes no `asset_class`.
- Returns the latest yield: `rate`, `unit` (`percent`), `date` (the daily
  observation day) and a Unix-ms `timestamp`. Yields are rates, not prices; do
  not feed bond symbols to the quote or snapshot tools.

### open_markets_board

- No parameters.
- Opens an interactive board with ten widely followed markets (crypto, forex,
  gold, oil, US indices, a US stock and the US 10-year yield), their price and
  change against the previous close, a chart for each row except the yield,
  and search across the catalog.
- Costs ten requests per open, one per row, on the user's plan; each chart,
  search or refresh inside the board costs more. For a single instrument, use
  `get_snapshot`.
- In clients that cannot show interactive views, the tool returns the same
  rows as a text summary.

### search_symbols

- Called by the markets board for its search box. Use `list_symbols` for your
  own symbol lookups.

## Perpetual futures

- `asset_class="perpetuals"` with USDT-quoted contract codes: `BTCUSDT`,
  `XAUUSDT`, `KOUSDT`. Resolve others with
  `list_symbols(asset_class="perpetuals", search=...)`.
- Contracts trade 24/7, including weekends, and each has one composite quote.
  `get_snapshot` adds `mark_price`.
- Prices need the Perpetuals plan; without it the data tools return 403.
  Listing the contracts works on any plan.
- `get_last_trade` can return 404 on a quiet contract while its quote is
  fine. History has tighter intraday ranges than the other classes.
- They have no market calendar, so the market tools do not take them.

## Bond tenor coverage

Countries are exactly `DE`, `ES`, `FR`, `IT`, `UK`, `US`.

- **US:** `1M`, `3M`, `6M`, `1Y`, `2Y`, `3Y`, `5Y`, `7Y`, `10Y`, `20Y`, `30Y`:
  money-market tenors plus the full coupon curve.
- **DE, ES, FR, IT, UK:** `1Y`, `2Y`, `3Y`, `5Y`, `10Y`, `30Y` only. No sub-year
  tenors, and no `7Y` or `20Y`; those are US-only.
- **UK gilts use `UK`, not `GB`.** `GB:10Y` returns 404. (The market
  calendar tools use `GB`, with `UK` as an alias, for the London market.)
- If a tenor returns 404, list what is available rather than interpolating a
  yield.

## Indices and ETF codes

Indices use neutral region-plus-number codes with descriptive names: `US500`
(US 500), `US100` (US 100), `US30` (US 30), `DE40` (Germany 40), `UK100`
(UK 100), `JP225` (Japan 225), `HK50` (Hong Kong 50), `EU50` (Europe 50).
Index values are levels, without volume in the usual sense.

ETFs use several forms: codes that follow the index they track (`US500ETF`,
`US100ETF`, `US30ETF`), sector, region or theme codes (`USGOLD`, `USTECH`),
plain US listing tickers (`SPY`), and `COUNTRY:CODE` for some Asian funds.
Resolve them with `list_symbols(asset_class="etfs")`.
Match the user's intent: benchmark questions mean the index, fund questions
mean the ETF.

## Symbols, timestamps, errors: quick recap

- Stocks are always `COUNTRY:TICKER` (`US:KO`, `DE:SAP`, `SA:2222`). Crypto,
  forex and commodities are plain pairs (`BTCUSD`, `EURUSD`, `XAUUSD`).
  Perpetual futures end in `USDT` (`BTCUSDT`).
- Data tools use Unix **milliseconds**; market calendar times are ISO 8601 UTC
  strings and their dates are the market's local dates; `get_history`
  `from`/`to` are UTC `YYYY-MM-DD` dates.
- 400 means fix the parameter the message names. 401 means connect the account
  or configure a key. 403 means the account cannot read the data, usually
  because of the plan, sometimes an expired package or a suspended key: stop,
  relay the message, never fabricate endpoints. 404 means check the symbol and
  the calendar. 429 means either slow down or, with `REST_QUOTA_EXCEEDED`, the
  monthly quota is used up. 503 means try again later.
- The MCP tools are the complete surface. There are no hidden REST routes to
  discover, and attempting them will not produce data.

## Positioning (applies to every answer built on this data)

TickerLayer data is derived and indicative, not official exchange prints.
Present it as TickerLayer data, never as official exchange prices, and never
turn it into investment advice.
