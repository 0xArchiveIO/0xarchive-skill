# 0xArchive Skill

A standalone skill that teaches Claude how to query the [0xArchive](https://0xarchive.io) REST API for historical crypto market data across Hyperliquid, Lighter.xyz, and HIP-3.

No dependencies required -- uses `curl` directly from the shell.

## Data Available

- **Orderbooks** -- L2 snapshots with configurable depth and granularity
- **Trades** -- Individual fills with maker/taker details
- **Candles** -- OHLCV aggregations (1m to 1w intervals)
- **Funding Rates** -- Historical and current
- **Open Interest** -- Historical and current
- **Liquidations** -- By symbol or user address
- **Data Quality** -- Coverage, latency, SLA, incidents

## Install for Claude Code

Symlink the skill into your project's `.claude/skills/` directory:

```bash
mkdir -p .claude/skills/0xarchive
ln -s ../../../skills/0xarchive-skill/SKILL.md .claude/skills/0xarchive/SKILL.md
```

Set your API key:

```bash
export OXARCHIVE_API_KEY="your-api-key"
```

Then use `/0xarchive` in Claude Code, e.g.:

```
/0xarchive BTC funding rate
/0xarchive ETH 4h candles last week
/0xarchive system health status
```

## Install for OpenClaw

```bash
openclaw install 0xarchive
```

Or add to your `openclaw.toml`:

```toml
[skills.0xarchive]
source = "clawhub:0xarchive/0xarchive"
env = ["OXARCHIVE_API_KEY"]
```

## Requirements

- An 0xArchive API key (get one at [0xarchive.io](https://0xarchive.io))
- `curl` and `jq` available in your shell
