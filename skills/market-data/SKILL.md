---
name: market-data
description: Query latest and historical market data through the TickerLayer MCP server, including quotes, snapshots, OHLCV history, symbol discovery, market hours and holidays, government bond yields and an interactive markets board, across crypto, forex, stocks, indices, ETFs, commodities and perpetual futures. Use whenever the user asks about a price, a market move, an exchange rate, candle or chart history, whether a market is open, a bond yield, or a market overview, and the TickerLayer tools (get_snapshot, get_history, list_symbols, open_markets_board, and the rest) are available.
---

# TickerLayer Market Data

Answer market questions with the TickerLayer MCP tools. The tools are the
complete surface. Never construct or guess REST URLs. If a call fails, report
what failed instead of filling the gap with remembered or estimated prices.

## Picking the right tool

| Question shape | Tool |
| --- | --- |
| "How is X doing?" Last price and change against the previous close in one call | `get_snapshot` |
| Current bid/ask spread | `get_quote` |
| Most recent trade (price, size, time) | `get_last_trade` |
| A confirmed close: yesterday's daily bar, or the latest settled intraday bar | `get_previous_close` |
| Chart data, ranges, "over the last month" | `get_history` |
| "What symbols do you have?" or resolving a fuzzy name | `list_symbols` |
| "Is the market open?" | `get_market_status` |
| Trading hours and session times | `get_market_sessions` |
| Upcoming or past market holidays | `get_market_holidays` |
| Government bond yields (US:10Y, DE:10Y, ...) | `get_bond_yield` |
| A broad market overview, or "open TickerLayer" | `open_markets_board` |

`search_symbols` powers the search box inside the markets board. Use
`list_symbols` for your own lookups.

Rules of thumb:

- **Every data tool needs `asset_class` as well as `symbol`.** `get_quote`,
  `get_last_trade`, `get_snapshot`, `get_previous_close` and `get_history` all
  require `asset_class`, one of `crypto`, `forex`, `stocks`, `indices`, `etfs`,
  `commodities`, `perpetuals`. Choose it deliberately: `XAUUSD` is commodities,
  `EURUSD` is forex, `XAUUSDT` is perpetuals, and a wrong class resolves to the
  wrong instrument or a 404.
- **`get_snapshot` is the best single call** for "how is X doing". It bundles
  the last price, bid and ask, the previous close, and the change and percent
  change. Prefer it over stitching `get_quote` and `get_previous_close`
  together. `get_quote` returns only bid, ask and sizes, so it is not the tool
  for "what's the price".
- **`get_previous_close` with `interval`** returns the latest fully settled bar
  of that interval (for example the last completed 1-hour bar), not only
  yesterday's daily close. The values are exactly `1m`, `5m`, `15m`, `1h`,
  `4h`, `1d`. Omit it for the previous daily bar. Use it when the user wants a
  confirmed, non-live value.
- **`get_history` has no `interval` parameter.** The bar size is `multiplier`
  plus `timespan`: `1`, `5` or `15` with `minute`; `1` or `4` with `hour`; `1`
  with `day`. No other pair is valid. `from` and `to` are UTC dates in
  `YYYY-MM-DD`. `limit` (default 500, max 5000) and `sort` (`asc` or `desc`,
  default `desc`) are optional.
- **One `get_history` request covers a limited span**, and the span is shorter
  for smaller bars. A range that is too large returns a 400 that names the
  maximum, so split it into consecutive requests. A non-null `next_offset`
  means more bars matched than `limit` returned: raise `limit` or narrow the
  dates.
- **Bonds go through `get_bond_yield` only.** The value is a yield in percent,
  not a price, and it is a daily observation. Do not ask the quote tools for
  bond symbols.
- **`open_markets_board` costs ten requests**, one per row, on the user's plan,
  and each chart, search or refresh inside the board costs more. Use it for
  overviews or when the user asks for TickerLayer itself. For one instrument,
  call `get_snapshot`. Where the client cannot show the
  interactive board, the tool returns the same rows as a text summary: present
  them as a short table.

## Symbol conventions

| Asset class | Format | Examples |
| --- | --- | --- |
| Crypto | pair, no separator | `BTCUSD`, `ETHUSD` |
| Forex | pair, no separator | `EURUSD`, `USDJPY` |
| Commodities | pair-style code | `XAUUSD` (gold), `XAGUSD` (silver), `WTIUSD` (WTI oil) |
| Indices | region plus number | `US500`, `US30`, `DE40`, `UK100`, `JP225` |
| ETFs | catalog codes and listing tickers | `US500ETF`, `USGOLD`, `SPY` |
| Stocks | **always** `COUNTRY:TICKER` (ISO alpha-2) | `US:KO`, `DE:SAP`, `SA:2222` |
| Perpetual futures | USDT-quoted contract | `BTCUSDT`, `XAUUSDT`, `KOUSDT` |
| Bonds | `COUNTRY:TENOR` (`DE`, `ES`, `FR`, `IT`, `UK`, `US`) | `US:10Y`, `DE:10Y`, `US:3M`, `UK:2Y` |

The country prefix on stocks is mandatory. `KO` alone is not a valid
TickerLayer stock symbol; it is `US:KO`. When the user gives a bare ticker,
infer the market from context or confirm through discovery (below). Default to
US only when the ticker is unambiguously a US listing.

Index names in the catalog are neutral and descriptive, such as US 500,
Germany 40 or Japan 225. When a user names an index by a brand, map it to the
neutral code by region and size, or search with
`list_symbols(asset_class="indices", search="US")`, and answer with the
neutral code and name.

Bond country codes are their own set: UK gilts are `UK:10Y`, **not** `GB:10Y`
(which 404s), even though the market calendar tools use `GB` (or `UK`) for the
London market.

Perpetual futures trade around the clock, including weekends. Each contract
has one composite quote, and `get_snapshot` adds a `mark_price`. They need the
Perpetuals plan.

## Discovery first

Call `list_symbols` before guessing a symbol. Guessing wastes calls and
produces confident wrong answers when a guess happens to match a different
instrument.

- `asset_class` is required, so decide what kind of instrument the user means
  ("Coca-Cola stock" means stocks, "gold" means commodities, "the US 500"
  means indices). Narrow with the optional `search` substring, matched
  case-insensitively against symbol and name, and `limit` (max 500, default
  100). There is no `market` parameter: approximate a country filter with a
  prefix search such as `search="DE:"`.
- When the phrasing is ambiguous, list the candidates and pick the one that
  matches intent. "Gold" can be the commodity `XAUUSD`, the fund `USGOLD` or
  the perpetual `XAUUSDT`; "the US 500" can be the index `US500` or the fund
  `US500ETF`. Use the index for "how did the market do" and the fund only when
  the user asks about the tradable fund.
- For stocks, `list_symbols` lists the markets the user's plan includes, so an
  empty result can mean the market is not in the plan. Say so instead of
  substituting a look-alike from another market.

## Market calendar before "no data"

A quiet symbol is usually a closed market, not an error. Before telling the
user that data looks stale or missing:

1. Call `get_market_status` for the relevant market.
2. If it is closed, say so and present the last available data as the close,
   with its timestamp.
3. Use `get_market_sessions` for exact open and close times and
   `get_market_holidays` when "why is it closed on a weekday" comes up. Both
   need `market` or `symbol`.

Crypto and perpetual futures trade continuously; forex closes on weekends;
stock and index hours vary by country. Perpetual futures have no market
calendar, so the calendar tools do not take them. Never read a weekend forex
or holiday stock value as an outage.

## Timestamps

Data tools return timestamps as Unix **milliseconds**. In the market calendar
tools, session start and end, `next_open` and `next_close` are ISO 8601 UTC
strings, `local_time` uses the market's own offset, and dates (holidays, and
the `date` you pass to `get_market_sessions`) are the market's local dates.
`get_history` `from` and `to` are plain UTC dates (`YYYY-MM-DD`), and a bond's
`date` is its observation day. When you show a time to the user, convert it to
a readable time and state the timezone you chose.

## Error handling

- **400**: the request is invalid. The message says what to fix, such as an
  unsupported bar size, a range that is too large, or an unknown market code.
  Fix the parameter and try once more.
- **401**: no valid credentials. Ask the user to connect their TickerLayer
  account or configure an API key. Do not retry.
- **403**: the account cannot read this data. Usually the plan does not
  include it (an asset class, a market, or perpetual futures); it can also
  mean a package has expired or a key is suspended. Relay the message. Do
  **not** invent endpoints, try look-alike symbols to get around it, or retry
  repeatedly.
- **404 or empty**: check the symbol with `list_symbols` and the calendar with
  `get_market_status` before concluding the data is missing. A quiet perpetual
  contract can return 404 for its last trade while its quote is fine.
- **429**: either the per-second rate limit (slow down and reduce polling) or
  the monthly quota (`REST_QUOTA_EXCEEDED`), which slowing down does not fix:
  it resets in the next period or needs a larger plan.
- **503**: data is temporarily unavailable. Say so, and try again later rather
  than in a loop.

## How to present the data

Every substantive market answer reflects TickerLayer's positioning:

- The data is **derived and indicative**: aggregated, not official exchange
  prints. Present prices as TickerLayer data, never as official exchange
  prices.
- Add a brief note on figures a user might act on, for example "Prices are
  indicative and may differ from official exchange values."
- **No investment advice.** Report data, trends and facts. Do not recommend
  buying, selling or holding anything. If asked for advice, say you can
  provide data but not recommendations.

## Playbooks for common requests

**"What's the price of X?"**
1. Resolve the symbol (`list_symbols` if it is not obvious).
2. `get_snapshot`: quote `last_price` with its `last_timestamp` and the change
   against `prev_close`. If `last_price` is null, report the bid and ask.
3. If the market is closed, label the number as the last close, not
   "current".

**"How did X do this week or month?"**
1. `get_history` with `multiplier=1`, `timespan="day"`, `sort="asc"` and
   `from`/`to` covering the range. Without `sort="asc"` the newest bar comes
   first.
2. Compute the change from the first open to the last close yourself and state
   both dates so the user knows exactly what was measured.
3. A range that includes today ends with a bar that is still forming. Use
   `get_previous_close` for a confirmed close and `get_snapshot` for "today so
   far".

**"Compare X and Y."**
1. One `get_snapshot` per symbol.
2. Compare percent change, not absolute price. Mention when the two trade on
   different calendars (for example crypto against a stock on a market
   holiday).

**"Give me a confirmed close for X at interval N."**
1. `get_previous_close` with `interval` (`1m`, `5m`, `15m`, `1h`, `4h`, `1d`).
   Never derive a close from a live quote or a bar that is still forming.

**"Why is X flat or not updating?"**
1. `get_market_status` first. A closed market is expected: say when it reopens
   (`get_market_sessions`, `get_market_holidays`).
2. Only if the market is open and the data is stale, report a data issue.

**"What's the 10-year yield?"**
1. `get_bond_yield` with `COUNTRY:TENOR` (`US:10Y` unless another country is
   implied). Present it as a percentage yield, not a price.

**"How are the markets doing?" or "Open TickerLayer."**
1. `open_markets_board`. The user sees the values on the board, so summarize
   the notable moves instead of repeating every number.
2. For follow-ups about one instrument, use `get_snapshot` or `get_history`.

**"What is gold doing on the weekend?"**
1. `get_snapshot` with `asset_class="perpetuals"` and `symbol="XAUUSDT"`. The
   perpetual trades while the commodity market is closed; say which one you
   quoted.

Precision: quote prices at the precision the API returns. Do not round crypto
or forex to two decimals. Attach the data's timestamp whenever the user is
making time-sensitive comparisons.

## Going deeper

For the full parameter reference (bar sizes, limits, sort, market codes, the
`include_sessions` and `include_holiday` flags on `get_market_status`, bond
tenor coverage and perpetual futures), read
[references/api-notes.md](references/api-notes.md).
