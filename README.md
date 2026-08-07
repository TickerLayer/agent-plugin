# TickerLayer Agent Plugin

An [Agent Plugins 1.0.0](https://agent-plugins.org) package that gives AI agents
market data: it bundles the hosted **TickerLayer MCP server** with a
**market-data skill** that teaches agents how to use it well.

- **MCP server:** `https://mcp.tickerlayer.com/mcp` (Streamable HTTP) with tools
  for quotes, last trades, snapshots, previous/settled closes, OHLCV history,
  symbol discovery, market status/sessions/holidays, and government bond yields.
- **Coverage:** crypto, forex, stocks, indices, ETFs, commodities, and bonds.
- **Skill:** `skills/market-data/` — tool selection guidance, symbol conventions
  (stocks are always `COUNTRY:TICKER`, e.g. `US:AAPL`), discovery-first workflow,
  market-calendar awareness, and error handling.

Docs for agents: <https://tickerlayer.com/llms-full.txt> ·
Plugin page: <https://tickerlayer.com/ai/agent-plugins> ·
MCP details: <https://tickerlayer.com/mcp>

## Install

### Codex CLI

```
/plugin marketplace add tickerlayer/agent-plugin
/plugin install tickerlayer
```

### Cursor

Run `/add-plugin` and provide this repository
(`tickerlayer/agent-plugin`).

### VS Code (Copilot)

Run **Chat: Install Plugin From Source** and paste the git URL
(`https://github.com/tickerlayer/agent-plugin`), or add this repo to
the `chat.plugins.marketplaces` setting.

### Claude Code

```
/plugin marketplace add tickerlayer/agent-plugin
/plugin install tickerlayer
```

### ChatGPT

ChatGPT connects to the MCP server directly — no plugin package needed. In
ChatGPT on the web: **Settings → Apps & Connectors → Advanced settings →
Developer mode**, then **Create** a connector with the server URL
`https://mcp.tickerlayer.com/mcp` and **Authentication: OAuth**. Sign in with
your TickerLayer account when prompted, then enable it in a chat from the
**+** menu → **Developer mode**.

## Authentication

- **OAuth (default):** clients that implement MCP authorization discover the
  authorization server automatically on first connect — sign in with your
  TickerLayer account. No keys to paste, nothing stored in this package.
- **API key (manual configs):** clients configured by hand can send an
  `x-api-key` header instead. Get a key at
  <https://tickerlayer.com/dashboard>. Keep keys in your client's local,
  private configuration — never commit them.

This package intentionally ships no credentials and no headers; per the Agent
Plugins spec, auth is managed by the client.

## What's in the package

```
plugin.json                          Agent Plugins manifest
mcp.json                             MCP server config (Agent Plugins format)
skills/market-data/SKILL.md          The market-data skill
skills/market-data/references/       Deeper parameter reference
.claude-plugin/plugin.json           Claude Code plugin manifest
.claude-plugin/marketplace.json      Claude Code marketplace manifest
.mcp.json                            Claude Code MCP server config
.agents/plugins/marketplace.json     Codex marketplace file
```

## Data positioning

TickerLayer data is derived and indicative — aggregated from multiple sources,
not exchange-official prints. It is provided for informational purposes only and
is not investment advice. Prices may differ from official exchange values.

## License

[MIT](LICENSE) — © 2026 TickerLayer.
