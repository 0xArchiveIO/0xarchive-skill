# 0xArchive Skill

A Claude Code plugin that teaches Claude how to query historical and real-time crypto market data across Hyperliquid, Lighter.xyz, and HIP-3 — no dependencies, just `curl`.

**ClawHub:** [0xarchive on ClawHub](https://clawhub.ai/0xFantomMenace/0xarchive)

## Quick Start

```bash
git clone https://github.com/0xArchiveIO/0xarchive-skill.git
mkdir -p .claude/skills/0xarchive
cp 0xarchive-skill/skills/query/SKILL.md .claude/skills/0xarchive/SKILL.md
export OXARCHIVE_API_KEY="0xa_your_api_key"
```

Then try: `/0xarchive BTC orderbook`

## Usage Examples

| Command | What you get |
|---------|-------------|
| `/0xarchive BTC funding rate` | Current funding rate for BTC |
| `/0xarchive ETH 4h candles last week` | Historical OHLCV candles |
| `/0xarchive SOL liquidations last 24h` | Recent liquidation events |
| `/0xarchive system health` | Data quality status across all exchanges |
| `/0xarchive compare BTC funding Hyperliquid vs Lighter` | Cross-exchange funding comparison |
| `/0xarchive sign up with wallet 0x...` | Web3 onboarding for a free API key |

## Data Available

- **Orderbooks** -- L2 snapshots with configurable depth and granularity
- **L4 Orderbooks** -- User-attributed orderbook reconstruction, diffs, and checkpoints (Hyperliquid + HIP-3)
- **L3 Orderbooks** -- Order-level orderbook snapshots with account filtering (Lighter)
- **Orders** -- Order history with user attribution, order flow aggregation, TP/SL history (Hyperliquid + HIP-3)
- **Trades** -- Individual fills with maker/taker details
- **Candles** -- OHLCV aggregations (1m to 1w intervals)
- **Funding Rates** -- Historical and current, with aggregation intervals
- **Open Interest** -- Historical and current, with aggregation intervals
- **Liquidations** -- By symbol, by user address, aggregated volume (Hyperliquid + HIP-3)
- **Price History** -- Mark, oracle, and mid price over time
- **Freshness** -- Per-data-type lag and last-updated timestamps
- **Market Summary** -- Price, funding, OI, volume, and liquidations in one call
- **Data Quality** -- Coverage, latency, SLA, incidents

## Install

Copy the skill into your project's `.claude/skills` directory:

```bash
git clone https://github.com/0xArchiveIO/0xarchive-skill.git
mkdir -p .claude/skills/0xarchive
cp 0xarchive-skill/skills/query/SKILL.md .claude/skills/0xarchive/SKILL.md
```

Set your API key:

```bash
export OXARCHIVE_API_KEY="0xa_your_api_key"
```

Then use `/0xarchive` in Claude Code, e.g.:

```
/0xarchive BTC funding rate
/0xarchive ETH 4h candles last week
/0xarchive system health status
```

## Bulk Data Downloads

For large-scale data exports (full order books, complete trade history, etc.), use the S3 Parquet bulk export available at [0xarchive.io/data](https://0xarchive.io/data). The Data Explorer lets you select time ranges, symbols, and data types, then download compressed Parquet files directly.

## Requirements

- An 0xArchive API key (get one at [0xarchive.io](https://0xarchive.io))
- `curl` and `jq` available in your shell
