---
name: market-data
description: Query live and historical market data through the TickerLayer MCP server — quotes, snapshots, OHLCV history, symbol discovery, market hours and holidays, and government bond yields across crypto, forex, stocks, indices, ETFs, and commodities. Use whenever the user asks about a price, a market move, an exchange rate, candle or chart history, whether a market is open, or a bond yield, and the TickerLayer tools (get_quote, get_snapshot, get_history, list_symbols, ...) are available.
---

# TickerLayer Market Data

Answer market questions with the TickerLayer MCP tools. The tools are the complete
surface — never construct or guess REST URLs, and never fall back to another data
source when a tool call fails.

## Picking the right tool

| Question shape | Tool |
| --- | --- |
| "How is X doing?" / last price + change vs. previous close in one shot | `get_snapshot` |
| Current bid/ask spread | `get_quote` |
| Most recent trade (price, size, time) | `get_last_trade` |
| Latest settled/closed bar, or yesterday's close | `get_previous_close` |
| Chart data, ranges, "over the last month" | `get_history` |
| "What symbols do you have?" / resolve a fuzzy name | `list_symbols` |
| "Is the market open?" | `get_market_status` |
| Trading hours / session times | `get_market_sessions` |
| Upcoming or past market holidays | `get_market_holidays` |
| Government bond yields (US:10Y, DE:10Y, ...) | `get_bond_yield` |

Rules of thumb:

- **Every data tool needs `asset_class` as well as `symbol`.** `get_quote`,
  `get_last_trade`, `get_snapshot`, `get_previous_close`, and `get_history` all
  require `asset_class` — one of `crypto`, `forex`, `stocks`, `indices`, `etfs`,
  `commodities`. Choose it deliberately: `XAUUSD` is commodities, `EURUSD` is
  forex, and a wrong class resolves to the wrong instrument or a 404.
- **`get_snapshot` is the best single call** for "how is X doing" — it bundles the
  last trade price and size, bid/ask, previous close, and change/percent change.
  Prefer it over stitching together `get_quote` + `get_previous_close` yourself.
  `get_quote` returns only bid/ask and sizes, so it is not the tool for "what's the
  price".
- **`get_previous_close` with `interval=`** returns the latest fully settled bar of
  that interval (e.g. the last completed 1-hour candle), not just yesterday's daily
  close. The interval values are exactly `1m`, `5m`, `15m`, `1h`, `4h`, `1d`; omit
  the parameter for the previous daily bar. Use it when the user wants a confirmed,
  non-live value.
- **`get_history` has no `interval` parameter.** The bar size is `multiplier` +
  `timespan`: `1`, `5`, or `15` with `minute`; `1` or `4` with `hour`; `1` with
  `day` — no other pair is valid. `from`/`to` are UTC dates in `YYYY-MM-DD`, and
  `limit` (default 500, max 5000) and `sort` (`asc`/`desc`, default `desc`) are
  optional. History depth is plan-limited; if a range comes back truncated or
  forbidden, report the limit — do not retry with tricks.
- **Bonds go through `get_bond_yield` only.** The value is a yield in percent, not a
  price. Do not ask the quote tools for bond symbols.

## Symbol conventions

| Asset class | Format | Examples |
| --- | --- | --- |
| Crypto | pair, no separator | `BTCUSD`, `ETHUSD` |
| Forex | pair, no separator | `EURUSD`, `USDJPY` |
| Commodities | pair-style code | `XAUUSD` (gold), `XAGUSD` (silver) |
| Indices | region + number style | `US500`, `DE40`, `JP225` |
| ETFs | index code + `ETF` suffix | `US500ETF` |
| Stocks | **always** `COUNTRY:TICKER` (ISO alpha-2) | `US:AAPL`, `DE:SAP`, `SA:2222` |
| Bonds | `COUNTRY:TENOR` (`DE`, `ES`, `FR`, `IT`, `UK`, `US`) | `US:10Y`, `DE:10Y`, `US:3M`, `UK:2Y` |

The country prefix on stocks is mandatory. `AAPL` alone is not a valid TickerLayer
stock symbol — it is `US:AAPL`. When the user gives a bare ticker, infer the market
from context or confirm via discovery (below); default US only when the ticker is
unambiguously a US listing.

Bond country codes are their own set: UK gilts are `UK:10Y`, **not** `GB:10Y`
(which 404s), even though `GB` is the right region code for UK *stocks* and for the
market-calendar tools.

## Discovery first

Call `list_symbols` before guessing a symbol. Guessing wastes calls and produces
confident wrong answers when a guess happens to collide with a different instrument.

- `asset_class` is required, so decide the kind of instrument the user means
  ("Apple stock" → stocks; "gold" → commodities; "the S&P" → indices), then narrow
  with the optional `search` substring (matched case-insensitively against symbol
  and name) and `limit` (max 500, default 100). There is no `market` parameter:
  approximate a country filter with a prefix search such as `search="DE:"`.
- When the user's phrasing is ambiguous ("Tesla" the stock vs. an ETF holding it,
  "S&P 500" the index vs. the ETF), list the candidates and pick the one that
  matches intent — index for "how did the S&P do", ETF only when they ask about the
  tradable fund.
- If discovery returns nothing for a plausible name, say the symbol is not covered.
  Do not substitute a look-alike from another market without saying so.

## Market calendar before "no data"

A quiet symbol is usually a closed market, not an error. Before telling the user
that data looks stale or missing:

1. Call `get_market_status` for the relevant market.
2. If closed, say so and present the last available data as the close, with its
   timestamp.
3. Use `get_market_sessions` for exact open/close times and `get_market_holidays`
   when "why is it closed on a weekday" comes up.

Crypto trades continuously; forex closes on weekends; stock and index hours vary by
country. Never interpret a weekend forex or holiday stock reading as an outage.

## Timestamps

All timestamps are **Unix milliseconds** unless a field's name or description says
otherwise. When you show a time to the user, convert to a human-readable time and
state the timezone you chose. `get_history` `from`/`to` are the exception: plain
UTC dates, `YYYY-MM-DD`.

## Error handling

- **401** — no valid credentials. Tell the user to connect their TickerLayer
  account (OAuth sign-in) or configure an API key in the client. Do not retry.
- **403** — the account's plan does not include that data (asset class, market, or
  history depth). Report exactly that. Do **not** invent REST endpoints, try
  alternate symbols to sneak around it, or hammer retries.
- **404 / empty** — check the symbol via `list_symbols` and the calendar via
  `get_market_status` before concluding data is missing.
- **429** — rate limited. Slow down, batch what you can, and reduce polling.

## How to present the data

Every substantive market answer must reflect TickerLayer's positioning:

- The data is **derived and indicative** — aggregated, not exchange-official. Do
  not describe prices as "official" exchange prints, and never name upstream
  providers or exchanges as the source.
- Add a brief note on figures a user might act on, e.g. "Prices are indicative and
  may differ from official exchange values."
- **No investment advice.** Report data, trends, and facts; do not recommend
  buying, selling, or holding anything. If asked for advice, say you can provide
  data but not recommendations.

## Playbooks for common requests

**"What's the price of X?"**
1. Resolve the symbol (`list_symbols` if not obviously known).
2. `get_snapshot` — quote `last_price` with its `last_timestamp` and the change
   against `prev_close`.
3. If the market is closed, label the number as the last close, not "current".

**"How did X do this week/month?"**
1. `get_history` with `multiplier=1`, `timespan="day"`, and `from`/`to` covering
   the range.
2. Compute the change from the first open to the last close yourself; state both
   endpoint dates so the user knows exactly what was measured.
3. For "today so far", use `get_snapshot` instead — history serves settled bars.

**"Compare X and Y."**
1. One `get_snapshot` per symbol.
2. Compare percent change, not absolute price. Mention if the two trade on
   different calendars (e.g. crypto vs. a stock on a market holiday).

**"Give me a confirmed close for X at interval N."**
1. `get_previous_close` with `interval=` (`1m`, `5m`, `15m`, `1h`, `4h`, `1d`) —
   never derive a "close" from a live quote or a still-forming history bar.

**"Why is X flat / not updating?"**
1. `get_market_status` first. Closed market → expected, say when it reopens
   (`get_market_sessions`, `get_market_holidays`).
2. Only if the market is open and data is genuinely stale, report a data issue.

**"What's the 10-year yield?"**
1. `get_bond_yield` with `COUNTRY:TENOR` (`US:10Y` unless another country is
   implied). Present as a percentage yield, not a price.

Precision: quote prices at the precision the API returns — do not round crypto
or forex to two decimals. Always attach the data's timestamp when the user is
making time-sensitive comparisons.

## Going deeper

For the full parameter reference (bar sizes, limits, sort, market codes, the
`include_sessions`/`include_holiday` flags on `get_market_status`, and bond tenor
coverage), read
[references/api-notes.md](references/api-notes.md). The complete agent-facing
documentation lives at https://tickerlayer.com/llms-full.txt.
