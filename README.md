# 0xArchive Skill

A Claude Code plugin that teaches Claude how to query the [0xArchive](https://0xarchive.io) REST API for historical crypto market data across Hyperliquid, Lighter.xyz, and HIP-3.

No dependencies required -- uses `curl` directly from the shell.

## Data Available

- **Orderbooks** -- L2 snapshots with configurable depth and granularity
- **Trades** -- Individual fills with maker/taker details
- **Candles** -- OHLCV aggregations (1m to 1w intervals)
- **Funding Rates** -- Historical and current
- **Open Interest** -- Historical and current
- **Liquidations** -- By symbol or user address
- **Data Quality** -- Coverage, latency, SLA, incidents

## Install as Plugin

Clone the repo and point Claude Code at it:

```bash
git clone https://github.com/0xArchiveIO/0xarchive-skill.git
claude --plugin-dir ./0xarchive-skill
```

Set your API key:

```bash
export OXARCHIVE_API_KEY="your-api-key"
```

Then use `/0xarchive:query` in Claude Code, e.g.:

```
/0xarchive:query BTC funding rate
/0xarchive:query ETH 4h candles last week
/0xarchive:query system health status
```

## Install as Standalone Skill

Alternatively, copy the skill directly into your project:

```bash
mkdir -p .claude/skills/0xarchive
cp 0xarchive-skill/skills/query/SKILL.md .claude/skills/0xarchive/SKILL.md
```

With the standalone approach, the skill is invoked as `/0xarchive` (no namespace prefix).

## Requirements

- An 0xArchive API key (get one at [0xarchive.io](https://0xarchive.io))
- `curl` and `jq` available in your shell
