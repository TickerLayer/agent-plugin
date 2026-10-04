# TickerLayer

Market data inside Claude. Ask how Bitcoin, gold or the US 10-year yield moved
today, pull a month of daily bars, or check whether Tokyo is open, and Claude
answers with data from one consistent API across crypto, forex, stocks,
indices, ETFs, commodities, perpetual futures and government bond yields.

The plugin connects Claude to the hosted TickerLayer MCP server and adds a
skill that teaches Claude to use it well: which tool fits a question, how
symbols are written, and why a quiet market is usually closed rather than
broken.

Ask in plain language:

- "How are Bitcoin and gold doing today?"
- "Pull daily EURUSD bars for June and summarize the trend."
- "Is the US stock market open right now? When does Tokyo open next?"
- "What is the US 10-year Treasury yield, and how does Germany compare?"
- "Open the TickerLayer markets board."

## What you get

**Coverage:** crypto, forex, US and international stocks, indices, ETFs,
commodities, perpetual futures and government bond yields, all through one
consistent schema.

**Tools.** Every tool only reads data.

| Tool | What it does |
| --- | --- |
| `get_snapshot` | Last price, bid and ask, previous close and change in one call |
| `get_quote` | Latest bid and ask |
| `get_last_trade` | Most recent trade: price, size and time |
| `get_previous_close` | Previous daily bar, or the latest settled intraday bar |
| `get_history` | OHLCV bars from 1 minute to 1 day between two dates |
| `list_symbols` | The symbols available for an asset class, with search |
| `get_market_status` | Whether a market is open, closed, or in pre-market or post-market |
| `get_market_sessions` | Trading session times for a market on a given date |
| `get_market_holidays` | Market holidays and early closes |
| `get_bond_yield` | Latest government bond yield, such as US:10Y |
| `open_markets_board` | An interactive board of ten widely followed markets, with charts and search |
| `search_symbols` | Powers the search box inside the markets board |

**Skill:** `market-data` covers tool selection, symbol conventions (stocks
always use `COUNTRY:TICKER`, for example `US:KO`), market hours and holidays,
error handling, and a parameter reference.

## Install

**Claude on the web, desktop and mobile, and Cowork:**

1. Add TickerLayer from the directory in **Customize > Plugins**.
2. Open the plugin's **Connectors** tab, add or connect TickerLayer, and sign
   in with your TickerLayer account. On Team and Enterprise plans, an Owner
   adds the connector for the organization first.

The skill loads either way, but the tools work only once the connector shows
**Connected**.

**Claude Code:**

```
/plugin marketplace add tickerlayer/agent-plugin
/plugin install tickerlayer@tickerlayer
```

Then run `/mcp`, select `plugin:tickerlayer:tickerlayer` and choose
**Authenticate**.

**Other agent clients:** this repository is also an
[Agent Plugins 1.0.0](https://agent-plugins.org) package for Codex CLI, Cursor
and VS Code. Install steps for each client are at
https://tickerlayer.com/ai/agent-plugins.

## Account and sign-in

You need a TickerLayer account, and a free plan is available. Create one at
https://tickerlayer.com/signup and compare plans at
https://tickerlayer.com/pricing.

Sign-in uses OAuth. The TickerLayer sign-in page opens, you approve the
connection, and no key is pasted into Claude or stored in this package.
Clients that you configure by hand can send an API key in the `x-api-key`
header instead. Keep keys in your client's private settings and never commit
them.

Data requests count against your plan's monthly quota. Opening the markets
board costs ten requests, and each chart, search or refresh inside it costs
more.

## What this plugin runs and sends

The plugin contains no executable code: a manifest, a reference to the hosted
MCP server, a skill written in Markdown, and an icon. Nothing runs on your
machine.

Claude connects only to TickerLayer: `https://mcp.tickerlayer.com/mcp` for tool
calls and api.tickerlayer.com for sign-in tokens. A tool call carries your
sign-in token and the tool's parameters, such as the asset class, symbol,
dates or market code. The server reads TickerLayer's own API and returns the
result. The markets board is drawn inside Claude and loads nothing from other
sites.

You sign in through your browser on tickerlayer.com, which uses our sign-in
service and also offers Google or GitHub sign-in. The website loads analytics
and support chat only with your cookie consent. The privacy policy covers all
of this.

Privacy policy: https://tickerlayer.com/privacy-policy

Terms of use: https://tickerlayer.com/terms-of-use

## About the data

TickerLayer data is derived and indicative. It is aggregated from multiple
sources, it is not an official exchange print, and prices can differ from
official exchange values. It is provided for information only and is not
investment advice.

## Support

Documentation: https://tickerlayer.com/docs/mcp

Contact: https://tickerlayer.com/consultation or info@tickerlayer.com

## License

[MIT](LICENSE). © 2026 TickerLayer.
