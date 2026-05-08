# 0xArchive Skill

Agent-ready 0xArchive market data prompts with no runtime dependency beyond `curl` and `jq`.

0xArchive is granular market data infrastructure for Hyperliquid and Lighter.xyz. HIP-3 builder perps, HIP-4 outcome markets (binary prediction markets), and Hyperliquid Spot all live under the Hyperliquid namespace. This repository ships the skill package for coding-agent clients that support local skills, including OpenClaw, Claude Code, Codex, and other SKILL.md-compatible agent environments.

Use this repo when your coding-agent surface supports a local skill package and you want the fastest route from `X-API-Key` to one live market-data answer. Use [AI Clients](https://www.0xarchive.io/docs/ai-clients) to choose between skills, MCP, CLI, markdown docs, `llms.txt`, OpenAPI, and other Claude Code or GPT Codex routes.

**ClawHub:** [0xarchive on ClawHub](https://clawhub.ai/0xFantomMenace/0xarchive)

## First Answer

### OpenClaw install

```bash
openclaw install 0xarchive
export OXARCHIVE_API_KEY="0xa_your_api_key"
```

Then ask:

```text
/0xarchive:query BTC funding rate
```

Expected result: current or recent 0xArchive market data, or a direct auth/configuration error.

## Agent Integration Requirements

| Agent surface | Requirement | Setup mechanic |
| --- | --- | --- |
| Claude Code | Local skills enabled | Copy `skills/query/SKILL.md` to `.claude/skills/0xarchive/SKILL.md` or install through OpenClaw |
| GPT Codex | Codex app, CLI, or IDE extension with skills enabled | Copy `skills/query/SKILL.md` to `.agents/skills/0xarchive/SKILL.md` in the target repo or use the skill path your Codex client documents |
| Both coding agents | `OXARCHIVE_API_KEY`, `curl`, and `jq` | Ask one `/0xarchive` query first, then expand into CLI, MCP, SDKs, or Data Catalog exports |
| No local skill support | Shell, MCP, or docs access | Use the CLI, MCP where supported, SDKs, `llms.txt`, OpenAPI, or markdown docs |

### Standalone install

```bash
git clone https://github.com/0xArchiveIO/0xarchive-skill.git

# Claude Code
mkdir -p .claude/skills/0xarchive
cp 0xarchive-skill/skills/query/SKILL.md .claude/skills/0xarchive/SKILL.md

# Codex repo-scope skill
mkdir -p .agents/skills/0xarchive
cp 0xarchive-skill/skills/query/SKILL.md .agents/skills/0xarchive/SKILL.md

export OXARCHIVE_API_KEY="0xa_your_api_key"
```

Then ask:

```text
/0xarchive:query BTC orderbook
```

## Claude Code And GPT Codex

Claude Code and GPT Codex should both start from the same 0xArchive product truth: one authenticated request first, then expand into deeper tooling only after the result is concrete. Use this SKILL.md package directly in skill-capable clients. When a client does not support local skills, use the same API key through the CLI, SDKs, MCP where supported, [llms.txt](https://www.0xarchive.io/llms.txt), [OpenAPI](https://www.0xarchive.io/openapi.json), and markdown docs.

## Usage Examples

| Command | What you get |
|---------|-------------|
| `/0xarchive:query BTC funding rate` | Current funding rate for BTC |
| `/0xarchive:query ETH 4h candles last week` | Historical OHLCV candles |
| `/0xarchive:query SOL liquidations last 24h` | Recent liquidation events |
| `/0xarchive:query km:US500 trades last hour` | Hyperliquid HIP-3 trades |
| `/0xarchive:query HIP-4 outcome 0 orderbook` | Hyperliquid HIP-4 outcome market orderbook (coin `0`, bare numeric form) |
| `/0xarchive:query HYPE-USDC spot trades last hour` | Hyperliquid Spot trades (dashed canonical symbol) |
| `/0xarchive:query system health` | Data quality status across venue APIs |
| `/0xarchive:query compare BTC funding Hyperliquid vs Lighter` | Venue funding comparison |
| `/0xarchive:query sign up with wallet 0x...` | Web3 onboarding for a free API key |

## Data Available

- **Orderbooks** -- L2 snapshots with configurable depth and granularity
- **L4 Orderbooks** -- User-attributed orderbook reconstruction, diffs, and checkpoints (Hyperliquid + HIP-3)
- **L3 Orderbooks** -- Order-level orderbook snapshots with account filtering (Lighter)
- **Orders** -- Order history with user attribution, order flow aggregation, TP/SL history (Hyperliquid + HIP-3)
- **Trades** -- Individual fills with maker/taker details
- **Candles** -- OHLCV aggregations (1m to 1w intervals)
- **Funding Rates** -- Historical and current, with aggregation intervals
- **Open Interest** -- Historical and current, with aggregation intervals
- **Liquidations** -- By symbol, by user address, aggregated volume (Hyperliquid + HIP-3; HIP-4 has no liquidations). Available historically and over the realtime `liquidations` / `hip3_liquidations` WebSocket channels (each event is a fill row with `is_liquidation: true`).
- **Outcome Markets** -- HIP-4 binary prediction markets with implied-probability pricing (0..1), `/outcomes` and `/outcomes/{id}` discovery endpoints, bare-numeric coin format `<10*outcome_id + side>` (legacy `#0` / `%230` also accepted), and the realtime `outcome_settled` WebSocket event when an outcome resolves
- **Hyperliquid Spot** -- 294 spot pairs (HYPE-USDC, PURR-USDC, AAPL-USDC, ...) under `/v1/hyperliquid/spot`. Trades backfilled to 2025-03-22 (~284M rows); orderbook, L4, and TWAP are live-forward from 2026-05-05. No funding, no open interest, no liquidations, no candles (perp-only constructs). Realtime channels `spot_orderbook`, `spot_trades`, `spot_l4_diffs`, `spot_l4_orders`, `spot_twap`.
- **Price History** -- Mark, oracle, and mid price over time
- **Freshness** -- Per-data-type lag and last-updated timestamps
- **Market Summary** -- Price, funding, OI, volume, and liquidations in one call
- **Data Quality** -- Coverage, latency, SLA, incidents

## Install

For Claude Code or Codex with skills enabled, copy the skill into the local skill directory your client reads:

```bash
git clone https://github.com/0xArchiveIO/0xarchive-skill.git

# Claude Code
mkdir -p .claude/skills/0xarchive
cp 0xarchive-skill/skills/query/SKILL.md .claude/skills/0xarchive/SKILL.md

# Codex repo-scope skill
mkdir -p .agents/skills/0xarchive
cp 0xarchive-skill/skills/query/SKILL.md .agents/skills/0xarchive/SKILL.md
```

Set your API key:

```bash
export OXARCHIVE_API_KEY="0xa_your_api_key"
```

Then use `/0xarchive` in that skill-capable client, e.g.:

```
/0xarchive BTC funding rate
/0xarchive ETH 4h candles last week
/0xarchive system health status
```

## Data Catalog

For file-based historical pulls, use the [Data Catalog](https://www.0xarchive.io/data). It lets you search markets, choose datasets and dates, see a live quote, and export zstd-compressed Parquet.

## Requirements

- An 0xArchive API key from [signup](https://www.0xarchive.io/signup)
- `curl` and `jq` available in your shell

## Next Paths

- First authenticated route: [Quick Start](https://www.0xarchive.io/docs/quick-start)
- Skill/MCP/agent choice: [AI Clients](https://www.0xarchive.io/docs/ai-clients)
- CLI for Claude Code, GPT Codex, and other shell-first agent work: [CLI docs](https://www.0xarchive.io/docs/cli)
- Plans and limits: [Pricing](https://www.0xarchive.io/pricing)
- Status and changes: [Status](https://www.0xarchive.io/status), [Changelog](https://www.0xarchive.io/changelog)
