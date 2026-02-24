# 0xArchive Skill

A Claude Code plugin that teaches Claude how to query the [0xArchive](https://0xarchive.io) REST API for historical crypto market data across Hyperliquid, Lighter.xyz, and HIP-3.

No dependencies required -- uses `curl` directly from the shell.

## Data Available

- **Orderbooks** -- L2 snapshots with configurable depth and granularity
- **Trades** -- Individual fills with maker/taker details
- **Candles** -- OHLCV aggregations (1m to 1w intervals)
- **Funding Rates** -- Historical and current, with aggregation intervals
- **Open Interest** -- Historical and current, with aggregation intervals
- **Liquidations** -- By symbol, by user address, aggregated volume
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

## Requirements

- An 0xArchive API key (get one at [0xarchive.io](https://0xarchive.io))
- `curl` and `jq` available in your shell
