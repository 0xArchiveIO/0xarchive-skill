---
name: 0xarchive
version: 1.14.0
description: >
  Query historical and real-time crypto market data from 0xArchive across two venues: Hyperliquid and Lighter.
  Lighter has two deployments: mainnet at /v1/lighter and Robinhood Chain at /v1/rh-lighter (USDG-quoted, e.g. BTC, AAPL-USDG).
  Hyperliquid includes HIP-3 builder perps (/v1/hyperliquid/hip3), HIP-4 outcome markets (/v1/hyperliquid/hip4), and Spot (/v1/hyperliquid/spot, e.g. HYPE-USDC).
  Covers orderbooks, trades, candles, funding, open interest, liquidations, account positions, outcome markets, TWAP, and data quality; data types are route-specific.
  WebSocket support is channel-specific: Lighter orderbook, trades, open interest, and funding stream live on both deployments; Lighter candles and L3 are replay-only.
  Use in Claude Code, Codex, and SKILL.md-compatible agents for market data, orderbooks, trades, candles, funding, prices, positions, real-time streams, prediction markets, or spot pairs on Hyperliquid, HIP-3, HIP-4, Hyperliquid Spot, Lighter, or Lighter on Robinhood Chain.
allowed-tools: Bash
argument-hint: "query, e.g. 'BTC funding rate', 'AAPL-USDG orderbook on Robinhood Chain', or 'positions for wallet 0x...'"
metadata: {"openclaw":{"requires":{"env":["OXARCHIVE_API_KEY"]},"primaryEnv":"OXARCHIVE_API_KEY"}}
---

# 0xArchive API Skill

Query historical and real-time crypto market data from **0xArchive** using `curl`. 0xArchive covers two venues: **Hyperliquid** and **Lighter**. **HIP-3** builder perps live under the Hyperliquid namespace at `/v1/hyperliquid/hip3`. **HIP-4** outcome markets (binary prediction markets) live at `/v1/hyperliquid/hip4`. **Hyperliquid Spot** has 326 authenticated inventory rows at `/v1/hyperliquid/spot`. Lighter has two deployments: mainnet at `/v1/lighter` and **Robinhood Chain** at `/v1/rh-lighter`. Robinhood Chain is a second deployment of Lighter, not a third venue. Data types are route-specific: orderbooks, trades, candles, funding rates, open interest, liquidations, account positions, outcome markets, spot, TWAP, and data quality metrics.

Orderbook depth limits apply to L2 snapshot endpoints only.

## Authentication

All endpoints require the `x-api-key` header. The key is read from `$OXARCHIVE_API_KEY`.

```bash
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" "https://api.0xarchive.io/v1/..."
```

## Venue Scopes & Coin Naming

| Scope | Path prefix | Coin format | Examples |
|----------|-------------|-------------|---------|
| Hyperliquid | `/v1/hyperliquid` | UPPERCASE | `BTC`, `ETH`, `SOL` |
| Hyperliquid HIP-3 | `/v1/hyperliquid/hip3` | Case-sensitive, `builder:NAME` | `km:US500`, `xyz:GOLD`, `hyna:BTC`, `vntl:SPACEX`, `flx:TSLA`, `cash:NVDA` |
| Hyperliquid HIP-4 | `/v1/hyperliquid/hip4` | Bare numeric `<10*outcome_id + side>` (legacy `#0` / `%230` also accepted) | `0`, `1`, `10`, `11` |
| Hyperliquid Spot | `/v1/hyperliquid/spot` | Dashed canonical `BASE-QUOTE` | `HYPE-USDC`, `PURR-USDC`, `AAPL-USDC` |
| Lighter (mainnet) | `/v1/lighter` | UPPERCASE | `BTC`, `ETH` |
| Lighter on Robinhood Chain | `/v1/rh-lighter` | UPPERCASE perps; dashed `BASE-USDG` spot | `BTC`, `ETH`, `AAPL-USDG` |

Hyperliquid and both Lighter deployments auto-uppercase the symbol server-side on market-data routes. Lighter and Robinhood Chain account positions routes (`/positions/{symbol}`, `/positions/{symbol}/summary`, and the `symbol` filter on account routes) match the symbol exactly, so pass it uppercase as `GET /instruments` lists it (`BTC`, not `btc`). HIP-3 coin names are passed through as-is. HIP-4 coins encode outcome and side: `0` is outcome 0 / side 0 (YES), `1` is outcome 0 / side 1 (NO), `10` is outcome 1 / side 0, etc. The bare numeric form is canonical; the legacy `#0` and `%230` forms still work for backward compatibility. Spot symbols are dashed (`HYPE-USDC`, `PURR-USDC`, `AAPL-USDC`); the server resolves the dashed form to the wire format (`PURR/USDC`, `@107`) internally, so always use the dashed form.

Lighter on Robinhood Chain markets and account indices are separate from Lighter mainnet even when the symbol text matches: `BTC` under `/v1/rh-lighter` is a different market from `BTC` under `/v1/lighter`. Keep the namespace with every result.

## Timestamps

All timestamps are **Unix milliseconds**. Use these shell helpers:

```bash
NOW=$(( $(date +%s) * 1000 ))
HOUR_AGO=$(( NOW - 3600000 ))
DAY_AGO=$(( NOW - 86400000 ))
WEEK_AGO=$(( NOW - 604800000 ))
```

## Response Format

Every response follows this shape:

```json
{
  "success": true,
  "data": [ ... ],
  "meta": {
    "count": 100,
    "request_id": "uuid",
    "next_cursor": "opaque-cursor"   // present when more pages exist
  }
}
```

Some routes add optional `meta` fields. Lighter trades (both deployments) carry `finalized_through`, plus `requested_end` and `clamped_to` when `end` was clamped, and `/recent` carries `preliminary_row_count`. Account positions routes add `as_of`, `snapshot_ts`, `source`, `quality`, `stale`, `built_through`, `finalized_through`, and `totals`; see [Account Positions](#account-positions). An empty page explained by coverage carries `notice` and `coverage_from`.

## Endpoint Reference

### Hyperliquid (`/v1/hyperliquid`)

| Endpoint | Params | Notes |
|----------|--------|-------|
| `GET /instruments` | -- | List all instruments |
| `GET /instruments/{symbol}` | -- | Single instrument details |
| `GET /orderbook/{symbol}` | `timestamp`, `depth` | Latest or at timestamp |
| `GET /orderbook/{symbol}/history` | `start`, `end`, `limit`, `cursor`, `depth` | Historical snapshots |
| `GET /trades/{symbol}` | `start`, `end`, `limit`, `cursor` | Trade history |
| `GET /candles/{symbol}` | `start`, `end`, `limit`, `cursor`, `interval` | OHLCV candles |
| `GET /funding/{symbol}/current` | -- | Current funding rate |
| `GET /funding/{symbol}` | `start`, `end`, `limit`, `cursor`, `interval` | Funding rate history |
| `GET /openinterest/{symbol}/current` | -- | Current open interest |
| `GET /openinterest/{symbol}` | `start`, `end`, `limit`, `cursor`, `interval` | OI history |
| `GET /liquidations/{symbol}` | `start`, `end`, `limit`, `cursor` | Liquidation events |
| `GET /liquidations/{symbol}/volume` | `start`, `end`, `limit`, `cursor`, `interval` | Aggregated liquidation volume (USD) |
| `GET /liquidations/user/{address}` | `start`, `end`, `limit`, `cursor`, `coin` | Liquidations for a user |
| `GET /freshness/{symbol}` | -- | Data freshness per data type |
| `GET /summary/{symbol}` | -- | Combined market summary (price, funding, OI, volume, liquidations) |
| `GET /prices/{symbol}` | `start`, `end`, `limit`, `cursor`, `interval` | Mark/oracle/mid price history |
| `GET /orders/{symbol}/history` | `start`, `end`, `user`, `status`, `order_type`, `limit`, `cursor` | Order history with user attribution |
| `GET /orders/{symbol}/flow` | `start`, `end`, `interval`, `limit` | Order flow aggregation |
| `GET /orders/{symbol}/tpsl` | `start`, `end`, `user`, `triggered`, `limit`, `cursor` | TP/SL order history |
| `GET /orderbook/{symbol}/l4` | `timestamp`, `depth` | L4 orderbook reconstruction |
| `GET /orderbook/{symbol}/l4/diffs` | `start`, `end`, `limit`, `cursor` | L4 orderbook diffs |
| `GET /orderbook/{symbol}/l4/history` | `start`, `end`, `limit`, `cursor` | L4 orderbook checkpoints |
| `GET /orderbook/{symbol}/l2` | `timestamp`, `depth` | L2 full-depth orderbook derived from L4 |
| `GET /orderbook/{symbol}/l2/history` | `start`, `end`, `limit`, `cursor`, `depth` | L2 full-depth checkpoints |
| `GET /orderbook/{symbol}/l2/diffs` | `start`, `end`, `limit`, `cursor` | L2 tick-level diffs |

### HIP-3 (`/v1/hyperliquid/hip3`)

Coin names are **case-sensitive** (e.g., `km:US500`). The authenticated August 22, 2026 inventory has 267 instruments across 10 builder prefixes: `abcd`, `cash`, `flx`, `hyna`, `io`, `km`, `mkts`, `para`, `vntl`, and `xyz`. Served trades, candles, and liquidation events begin February 1, 2026; native L2, funding, and OI begin February 16, 2026; L4 diffs and order-lifecycle rows have a March 10, 2026 family floor; reconstructable checkpoints and point-in-time state can begin later by symbol. All HIP-3 symbols are available on every tier.

| Endpoint | Params | Notes |
|----------|--------|-------|
| `GET /instruments` | -- | List HIP-3 instruments |
| `GET /instruments/{coin}` | -- | Single instrument |
| `GET /orderbook/{coin}` | `timestamp`, `depth` | All HIP-3 symbols, every tier. |
| `GET /orderbook/{coin}/history` | `start`, `end`, `limit`, `cursor`, `depth` | All HIP-3 symbols, every tier. |
| `GET /trades/{coin}` | `start`, `end`, `limit`, `cursor` | Trade history |
| `GET /trades/{coin}/recent` | `limit` | Recent trades (no time range needed) |
| `GET /candles/{coin}` | `start`, `end`, `limit`, `cursor`, `interval` | OHLCV candles |
| `GET /funding/{coin}/current` | -- | Current funding rate |
| `GET /funding/{coin}` | `start`, `end`, `limit`, `cursor`, `interval` | Funding history |
| `GET /openinterest/{coin}/current` | -- | Current OI |
| `GET /openinterest/{coin}` | `start`, `end`, `limit`, `cursor`, `interval` | OI history |
| `GET /liquidations/{coin}` | `start`, `end`, `limit`, `cursor` | Liquidation events |
| `GET /liquidations/{coin}/volume` | `start`, `end`, `limit`, `cursor`, `interval` | Aggregated liquidation volume (USD) |
| `GET /freshness/{coin}` | -- | Data freshness per data type |
| `GET /summary/{coin}` | -- | Combined market summary (price, funding, OI) |
| `GET /prices/{coin}` | `start`, `end`, `limit`, `cursor`, `interval` | Mark/oracle/mid price history |
| `GET /orders/{coin}/history` | `start`, `end`, `user`, `status`, `order_type`, `limit`, `cursor` | Order history with user attribution |
| `GET /orders/{coin}/flow` | `start`, `end`, `interval`, `limit` | Order flow aggregation |
| `GET /orders/{coin}/tpsl` | `start`, `end`, `user`, `triggered`, `limit`, `cursor` | TP/SL order history |
| `GET /orderbook/{coin}/l4` | `timestamp`, `depth` | L4 orderbook reconstruction |
| `GET /orderbook/{coin}/l4/diffs` | `start`, `end`, `limit`, `cursor` | L4 orderbook diffs |
| `GET /orderbook/{coin}/l4/history` | `start`, `end`, `limit`, `cursor` | L4 orderbook checkpoints |
| `GET /orderbook/{coin}/l2` | `timestamp`, `depth` | L2 full-depth orderbook derived from L4 |
| `GET /orderbook/{coin}/l2/history` | `start`, `end`, `limit`, `cursor`, `depth` | L2 full-depth checkpoints |
| `GET /orderbook/{coin}/l2/diffs` | `start`, `end`, `limit`, `cursor` | L2 tick-level diffs |

### HIP-4 (`/v1/hyperliquid/hip4`)

Outcome markets are binary prediction markets (e.g. "Will BTC be >= $X by date Y?"). Coin names are bare numerics `<10*outcome_id + side>` (e.g. `0`, `1`, `10`); the legacy `#0` / `%230` forms are still accepted. HIP-4 has **candles and outcome-side open interest from May 2, 2026**, with raw OI updates at roughly 10 seconds. HIP-4 has **no funding rates and no liquidations**. The `mark_price` field on HIP-4 instruments and prices is an **implied probability in the range 0..1**, not a USD price.

| Endpoint | Params | Notes |
|----------|--------|-------|
| `GET /outcomes` | -- | List all outcome markets (HIP-4 only; not present on other venues) |
| `GET /outcomes/{outcome_id}` | -- | Single outcome market detail (HIP-4 only) |
| `GET /instruments` | -- | List HIP-4 instruments (one per side per outcome) |
| `GET /instruments/{coin}` | -- | Single instrument. Coin is the bare numeric (e.g. `0`); legacy `%230` also accepted. |
| `GET /orderbook/{coin}` | `timestamp`, `depth` | Latest or at timestamp |
| `GET /orderbook/{coin}/history` | `start`, `end`, `limit`, `cursor`, `depth` | Historical snapshots |
| `GET /trades/{coin}` | `start`, `end`, `limit`, `cursor` | Trade history |
| `GET /trades/{coin}/recent` | `limit` | Recent trades |
| `GET /candles/{coin}` | `start`, `end`, `limit`, `cursor`, `interval` | Implied-probability OHLCV candles |
| `GET /openinterest/{coin}/current` | -- | Current open interest |
| `GET /openinterest/{coin}` | `start`, `end`, `limit`, `cursor`, `interval` | Outcome-side OI history; raw rows roughly every 10 seconds |
| `GET /freshness/{coin}` | -- | Data freshness per data type |
| `GET /summary/{coin}` | -- | Combined market summary (implied probability + OI; no funding) |
| `GET /prices/{coin}` | `start`, `end`, `limit`, `cursor`, `interval` | Implied-probability history (mark/oracle/mid in 0..1) |
| `GET /orders/{coin}/history` | `start`, `end`, `user`, `status`, `order_type`, `limit`, `cursor` | Order history |
| `GET /orders/{coin}/flow` | `start`, `end`, `interval`, `limit` | Order flow aggregation |
| `GET /orders/{coin}/tpsl` | `start`, `end`, `user`, `triggered`, `limit`, `cursor` | TP/SL order history |
| `GET /orderbook/{coin}/l4` | `timestamp`, `depth` | L4 orderbook reconstruction |
| `GET /orderbook/{coin}/l4/diffs` | `start`, `end`, `limit`, `cursor` | L4 orderbook diffs |
| `GET /orderbook/{coin}/l4/history` | `start`, `end`, `limit`, `cursor` | L4 orderbook checkpoints |
| `GET /orderbook/{coin}/l2` | `timestamp`, `depth` | L2 full-depth orderbook derived from L4 |
| `GET /orderbook/{coin}/l2/history` | `start`, `end`, `limit`, `cursor`, `depth` | L2 full-depth checkpoints |
| `GET /orderbook/{coin}/l2/diffs` | `start`, `end`, `limit`, `cursor` | L2 tick-level diffs |

### Hyperliquid Spot (`/v1/hyperliquid/spot`)

The authenticated inventory has 326 Hyperliquid Spot rows (HYPE-USDC, PURR-USDC, AAPL-USDC, ...). Symbols are **dashed canonical** (`BASE-QUOTE`); the server resolves to the wire format (`PURR/USDC`, `@107`) internally. Spot has **no funding rates, open interest, or liquidations**. Spot candles are served from 2025-03-22T10:50:22Z through a dedicated OHLCV route. Use spot for pair discovery, candles, current and historical L2 orderbooks, fills, L4 reconstruction, order lifecycle, and TWAP execution status.

Coverage:
- **Candles**: served from 2025-03-22T10:50:22Z at `1m`, `5m`, `15m`, `30m`, `1h`, `4h`, `1d`, and `1w`; maximum `limit` is 1000 and `next_cursor` is opaque.
- **Trades**: backfilled from 2025-03-22 (~284M rows). Pre-March 2025 spot fills are not available (no public archive existed).
- **Native L2 and TWAP**: live-forward from 2026-05-05. No native L2 history is claimed before that date.
- **L4**: raw diffs are served from 2026-03-10; reconstructable checkpoints and point-in-time state are observed from 2026-03-11. Exact starts vary by pair.

| Endpoint | Params | Notes |
|----------|--------|-------|
| `GET /pairs` | -- | List current Spot pairs; authenticated inventory has 326 rows |
| `GET /pairs/{symbol}` | -- | Single pair detail (e.g. `HYPE-USDC`) |
| `GET /candles/{symbol}` | `start`, `end`, `limit`, `cursor`, `interval` | OHLCV candles from 2025-03-22T10:50:22Z; intervals `1m` through `1w`; max `limit` 1000; cursor is opaque |
| `GET /orderbook/{symbol}` | `timestamp`, `depth` | Current L2 orderbook (live from 2026-05-05) |
| `GET /orderbook/{symbol}/history` | `start`, `end`, `limit`, `cursor`, `depth` | L2 history window (from 2026-05-05) |
| `GET /orderbook/{symbol}/l4` | `timestamp`, `depth` | Point-in-time L4 reconstruction; observed from 2026-03-11 and exact starts vary by pair |
| `GET /orderbook/{symbol}/l4/diffs` | `start`, `end`, `limit`, `cursor` | Raw L4 diffs |
| `GET /orderbook/{symbol}/l4/history` | `start`, `end`, `limit`, `cursor` | L4 checkpoints |
| `GET /trades/{symbol}` | `start`, `end`, `limit`, `cursor`, `user` | Spot trades. Pass `user` to filter to a wallet's fills. Backfilled to 2025-03-22. |
| `GET /orders/{symbol}/history` | `start`, `end`, `user`, `status`, `order_type`, `limit`, `cursor` | Spot order lifecycle |
| `GET /twap/{symbol}` | `start`, `end`, `limit`, `cursor` | TWAP execution statuses for a symbol |
| `GET /twap/user/{user}` | `start`, `end`, `limit`, `cursor` | TWAP execution statuses for a wallet |
| `GET /freshness/{symbol}` | -- | Data freshness per data type |

### Lighter mainnet (`/v1/lighter`)

Lighter has native L2, L3, trades, candles, funding, open interest, liquidation events and volume, freshness, summary, and price history. Candles begin August 1, 2025. Funding and OI begin August 25, 2025 and update roughly every 10 seconds. Per-fill Lighter trade rows begin January 17, 2025 and carry maker/taker context; exact starts vary by market. Native L2 begins January 29, 2026. L3 begins March 5, 2026 and is capped at 250 resting orders per side. Liquidations are live-only from capture start and have no public backfill before that capture window.

| Endpoint | Params | Notes |
|----------|--------|-------|
| `GET /instruments` | -- | List Lighter instruments |
| `GET /instruments/{symbol}` | -- | Single instrument |
| `GET /orderbook/{symbol}` | `timestamp`, `depth` | Latest or at timestamp |
| `GET /orderbook/{symbol}/history` | `start`, `end`, `limit`, `cursor`, `depth`, `granularity` | Default granularity: `checkpoint` |
| `GET /trades/{symbol}` | `start`, `end`, `limit`, `cursor` | Per-fill history with maker/taker context. Per-fill Lighter trade rows begin January 17, 2025; exact starts vary by market. Returns reconciled trades only; `end` is clamped to `meta.finalized_through` |
| `GET /trades/{symbol}/recent` | `limit` | Recent trades (no time range needed), including preliminary trades not yet reconciled |
| `GET /candles/{symbol}` | `start`, `end`, `limit`, `cursor`, `interval` | OHLCV candles from August 1, 2025 |
| `GET /funding/{symbol}/current` | -- | Current funding rate |
| `GET /funding/{symbol}` | `start`, `end`, `limit`, `cursor`, `interval` | Funding history from August 25, 2025; raw updates roughly every 10 seconds |
| `GET /openinterest/{symbol}/current` | -- | Current OI |
| `GET /openinterest/{symbol}` | `start`, `end`, `limit`, `cursor`, `interval` | OI history from August 25, 2025; raw updates roughly every 10 seconds |
| `GET /liquidations/{symbol}` | `start`, `end`, `limit`, `cursor` | Liquidation events; live-only from capture start, with no earlier public backfill |
| `GET /liquidations/{symbol}/volume` | `start`, `end`, `limit`, `cursor`, `interval` | Time-bucketed liquidation volume |
| `GET /freshness/{symbol}` | -- | Data freshness per data type |
| `GET /summary/{symbol}` | -- | Combined market summary (price, funding, OI) |
| `GET /prices/{symbol}` | `start`, `end`, `limit`, `cursor`, `interval` | Mark/oracle price history |
| `GET /l3orderbook/{symbol}` | `timestamp`, `depth`, `account` | L3 order-level snapshot; up to 250 orders per side |
| `GET /l3orderbook/{symbol}/history` | `start`, `end`, `limit`, `cursor`, `granularity`, `account` | Tick-level L3 snapshots from March 5, 2026; up to 250 orders per side |

Account positions for Lighter mainnet live under the same prefix (`/accounts/{account_index}/positions`, `/positions/{symbol}`, and the L1 lookup `/accounts?l1_address=`); see [Account Positions](#account-positions).

### Lighter on Robinhood Chain (`/v1/rh-lighter`)

Robinhood Chain is Lighter's second deployment, not a third venue. `/v1/rh-lighter` mirrors `/v1/lighter` with the same parameters and response shapes, except the two `l3orderbook` routes: this deployment has no L3 capture, so they return 404 with a message pointing to `/v1/rh-lighter/orderbook/{symbol}`. Markets are quoted in USDG: 84 at launch, 57 perps and 27 spot. Perp symbols are uppercase (`BTC`, `ETH`); spot symbols are the base and quote joined by a dash (`AAPL-USDG`). Funding, open interest, and liquidations exist for perps only. Start with `GET /instruments` to discover symbols.

Coverage (UTC):
- **Trades**: from 2026-06-26 20:10:26, the deployment's first trade.
- **Order book, open interest, and funding**: from 2026-08-22 18:43. Earlier history of these streams is not recoverable.
- **Liquidations**: from 2026-08-22 18:43. Requests that start earlier return 400 (`... from 2026-08-22 18:43 UTC onward`).
- **Candles**: from 2026-06-26 once candles are enabled for this deployment. Until then the route returns 400 `Candles are not yet available for Lighter (Robinhood Chain). ...`.
- **L3**: not captured. Use L2.

Trades are two-tier, exactly like Lighter mainnet. `GET /trades/{symbol}` returns finalized rows (`source: "bucket"`) and clamps `end` to `meta.finalized_through`, which trails the present by about a day; when it clamps, the response adds `meta.requested_end` and `meta.clamped_to`. `GET /trades/{symbol}/recent` includes newer preliminary rows (`source: "ws"`), counted by `meta.preliminary_row_count`. Prices and amounts are in USDG, and account fields hold Robinhood Chain account indices, unrelated to mainnet indices with the same number.

| Endpoint | Params | Notes |
|----------|--------|-------|
| `GET /instruments` | -- | List markets (perps and USDG-quoted spot) |
| `GET /instruments/{symbol}` | -- | Single market |
| `GET /orderbook/{symbol}` | `timestamp`, `depth` | Latest or at timestamp, from 2026-08-22 18:43 UTC |
| `GET /orderbook/{symbol}/history` | `start`, `end`, `limit`, `cursor`, `depth`, `granularity` | Default granularity: `checkpoint` |
| `GET /trades/{symbol}` | `start`, `end`, `limit`, `cursor` | Finalized per-fill history from 2026-06-26 20:10:26 UTC; `end` is clamped to `meta.finalized_through` |
| `GET /trades/{symbol}/recent` | `limit` | Recent trades, including preliminary rows (`meta.preliminary_row_count`) |
| `GET /candles/{symbol}` | `start`, `end`, `limit`, `cursor`, `interval` | OHLCV candles from 2026-06-26 once enabled for this deployment; 400 until then |
| `GET /funding/{symbol}/current` | -- | Current funding rate (perps) |
| `GET /funding/{symbol}` | `start`, `end`, `limit`, `cursor`, `interval` | Funding history (perps) from 2026-08-22 18:43 UTC |
| `GET /openinterest/{symbol}/current` | -- | Current OI (perps) |
| `GET /openinterest/{symbol}` | `start`, `end`, `limit`, `cursor`, `interval` | OI history (perps) from 2026-08-22 18:43 UTC |
| `GET /liquidations/{symbol}` | `start`, `end`, `limit`, `cursor` | Liquidation events (perps) from 2026-08-22 18:43 UTC |
| `GET /liquidations/{symbol}/volume` | `start`, `end`, `limit`, `cursor`, `interval` | Time-bucketed liquidation volume |
| `GET /freshness/{symbol}` | -- | Data freshness per data type |
| `GET /summary/{symbol}` | -- | Combined market summary |
| `GET /prices/{symbol}` | `start`, `end`, `limit`, `cursor`, `interval` | Mark/oracle price history |

Account positions for this deployment use `/v1/rh-lighter/accounts/{account_index}/positions` and the market routes under `/v1/rh-lighter/positions`; there is no L1 address lookup on Robinhood Chain. See [Account Positions](#account-positions). Check symbol-level windows with `GET /v1/data-quality/coverage/rh-lighter/{symbol}` before a long pull.

### Account Positions

Open perpetual positions of public venue accounts: what an address or Lighter account index holds now, what it held at any past instant, every fill that changed it, and how a whole market is positioned. Positions exist on Hyperliquid core, HIP-3, Lighter mainnet, and Lighter on Robinhood Chain. Spot and HIP-4 have no positions routes.

| Venue | Prefix | Account key | Change log from (UTC) | Hourly snapshots from (UTC) | Live snapshot |
|-------|--------|-------------|-----------------------|-----------------------------|---------------|
| Hyperliquid core | `/v1/hyperliquid` | `0x` address | 2025-05-25 | 2026-06-07 | Every 5 minutes |
| HIP-3 | `/v1/hyperliquid/hip3` | `0x` address, optional `dex` | 2025-10-13 | 2026-06-07 | Every 5 minutes |
| Lighter mainnet | `/v1/lighter` | Integer `account_index` | 2025-01-17 | 2025-01-17 | Every 2 minutes |
| Lighter on Robinhood Chain | `/v1/rh-lighter` | Integer `account_index` | 2026-06-26 | 2026-06-26 | Every 2 minutes |

Append these paths to the venue prefix:

| Hyperliquid and HIP-3 | Lighter and Robinhood Chain | Params | Notes |
|-----------------------|-----------------------------|--------|-------|
| `GET /wallets/{address}/positions` | `GET /accounts/{account_index}/positions` | `timestamp`, `symbol`, `dex` (HIP-3), `limit`, `cursor` | Open positions at the latest live snapshot, or as of `timestamp`. `data.positions`; `data.account` on the first page of a snapshot read (HIP-3 needs `dex` or a dex-prefixed `symbol`; null when reconstructed or when a Lighter `symbol` filter is set); `data.account_seen` when empty |
| `GET /wallets/{address}/positions/history` | `GET /accounts/{account_index}/positions/history` | `start`, `end`, `symbol`, `dex` (HIP-3), `limit`, `cursor` | One row per open position per committed hour in `[start, end)` |
| `GET /wallets/{address}/positions/changes` | `GET /accounts/{account_index}/positions/changes` | `start`, `end`, `symbol`, `dex` (HIP-3), `limit`, `cursor` | One row per fill leg that changed a position, oldest first, with `start_position`, `end_position`, `event_type`, and `cause` |
| `GET /wallets/{address}/account` | `GET /accounts/{account_index}/account` | `dex` (HIP-3) | Latest account summary. Lighter returns position aggregates only (no margin or collateral fields) |
| `GET /wallets/{address}/account/history` | `GET /accounts/{account_index}/account/history` | `start`, `end`, `dex` (HIP-3), `limit`, `cursor` | Hourly account summaries |
| `GET /positions/{symbol}` | `GET /positions/{symbol}` | `hour`, `side`, `min_value`, `include_system` (Lighter), `limit`, `cursor` | Every open position in one market, largest value first; `meta.totals` on the first page |
| `GET /positions/{symbol}/summary` | `GET /positions/{symbol}/summary` | `start`, `end`, `include_system` (Lighter), `limit`, `cursor` | Long/short counts, sizes, values, average entries, and top-10 share: now, or an hourly series over `[start, end)` |
| `GET /positions` | `GET /positions` | `hour`, `include_system` (Lighter), `limit`, `cursor`, `format` | Every open position across markets at one hour (bulk) |

Two more routes support these: `GET /v1/lighter/accounts?l1_address=0x...` lists the Lighter mainnet account indices an L1 address owns (`total_accounts`, `accounts[].account_index`), and `GET /v1/data-quality/positions` reports per venue the latest live snapshot and its age, the latest hourly snapshot, `built_through`, and `finalized_through`.

How to read positions:
- `timestamp`, `hour`, `start`, and `end` are Unix milliseconds; `hour` must be an exact UTC hour. `address` is a 42-character `0x` hex string; Lighter paths take integer account indices only.
- Without `timestamp`, wallet and account routes read the latest live snapshot; `meta.stale` is `true`, with a `meta.notice`, when it is more than 12 minutes old. With `timestamp`, the response is the state after every event before that instant: an exact hour with a committed snapshot serves that snapshot (`meta.source: "snapshot"`); any other instant is reconstructed (`meta.source: "reconstructed"`), with exact size, entry, and `opened_at` and mark-based values at that instant. Snapshot-only fields are empty there: on Hyperliquid and HIP-3, `leverage.value`, `max_leverage`, `margin_used`, `liquidation_price`, `return_on_equity`, and the `cum_funding` members are `null`, `leverage.type` is `"unknown"`, and `liquidation_price_status` is `"unavailable"`. Lighter rows leave those fields empty on every read, with `leverage.type` holding the margin mode. Use `meta.source`, not a null field, to tell a reconstructed read apart.
- Reads are clamped to `meta.built_through`, adding `meta.requested_end` and `meta.clamped_to` when they are. `meta.finalized_through` marks how far the data is final; on Lighter it follows the trade finalization watermark, about a day behind, and newer Lighter rows carry `finalized: false`.
- An empty wallet page sets `data.account_seen`: `flat` (activity recorded, nothing open), `never_seen` (no recorded activity in covered history; `meta.notice` and `meta.coverage_from` scope the claim), or `outside_coverage` (the instant is before coverage).
- Every row carries `quality` (`complete`, `partial`, `degraded`; Lighter rows can also be `preliminary`, `unreconciled`, or `incomplete`), and `meta.quality` reports the snapshot the page was read from.
- Numbers are decimal strings and `data` timestamps are RFC 3339 UTC. Lighter `account_index` values are strings in responses, with `account_kind` (`user`, `settlement`, `insurance`, `system`). Lighter market and bulk routes exclude system accounts unless `include_system=true`.
- Market routes (`/positions/{symbol}`, `/positions`) read committed snapshots only: the latest live tick by default for `/positions/{symbol}`, the latest hourly snapshot by default for `/positions`, or `hour=H`, echoed as `meta.snapshot_ts`.
- Cursors are signed and bound to the route, account or market, and filters: pass `meta.next_cursor` back unchanged with the same parameters. Market and bulk cursors pin the first page's snapshot; if it has been replaced or expired, the next page returns 409 `snapshot_advanced`, so restart without a cursor.
- Limits: wallet and account routes default 500, max 5,000 rows; market routes default 100, max 2,000; summary series cover at most 168 hours per page; bulk `/positions` defaults to 1,000 rows, max 2,000 as JSON. Where Arrow is enabled, `format=arrow` (or `Accept: application/vnd.apache.arrow.stream`) returns an Arrow IPC stream of up to 50,000 rows with pagination in the `x-next-cursor` header; check `Content-Type`, because JSON is the default.

**Billing:** positions routes bill like trades: one credit per 1,000 rows returned, with a minimum of one credit per request. Account summary routes (`/account`, `/account/history`) and the Lighter L1 lookup cost one credit per request. Plan history windows (Free: rolling 30 days) apply to `timestamp`, `hour`, and `[start, end)`.

### Data Quality (`/v1/data-quality`)

| Endpoint | Params | Notes |
|----------|--------|-------|
| `GET /status` | -- | System health status |
| `GET /coverage` | -- | Coverage summary across venue APIs |
| `GET /coverage/{exchange}` | -- | Coverage for one venue scope: `hyperliquid`, `hip3`, `hip4`, `spot`, `lighter`, or `rh-lighter` |
| `GET /coverage/{exchange}/{symbol}` | `from`, `to` | Symbol-level coverage + gaps |
| `GET /incidents` | `status`, `exchange`, `since`, `limit`, `offset` | List incidents |
| `GET /incidents/{id}` | -- | Single incident |
| `GET /latency` | -- | Ingestion latency metrics |
| `GET /sla` | `year`, `month` | SLA compliance report |
| `GET /positions` | -- | Account positions freshness per venue: live snapshot age, latest hourly snapshot, `built_through`, `finalized_through` |

### WebSocket Channels

Real-time + historical-replay channels available via WebSocket (`wss://api.0xarchive.io/ws?apiKey=KEY`). WebSocket access (including all L4 channels) is available on every tier, starting with Free (10 subscriptions / 2 connections / 10x replay). Lighter mainnet and Lighter on Robinhood Chain channels are listed separately under [Lighter channels](#lighter-channels).

**Trades + liquidations (live + replay, except Spot):**

| Channel | Notes |
|---------|-------|
| `trades` | Hyperliquid trades. One row per side per fill. |
| `hip3_trades` | HIP-3 trades. |
| `hip4_trades` | HIP-4 trades. |
| `spot_trades` | Hyperliquid Spot trades. Symbol is dashed (`HYPE-USDC`). Live only; replay is not supported. |
| `liquidations` | Hyperliquid liquidations. **Each event is a fill row with `is_liquidation: true` (same shape as `trades`).** |
| `hip3_liquidations` | HIP-3 liquidations. **Each event is a fill row with `is_liquidation: true` (same shape as `hip3_trades`).** |

**Orderbook + open interest:**

| Channel | Notes |
|---------|-------|
| `orderbook`, `hip3_orderbook` | Live and replayable L2 orderbook updates |
| `spot_orderbook` | Hyperliquid Spot L2 orderbook updates. Symbol is dashed (`HYPE-USDC`). Live only; replay is not supported. |
| `hip4_orderbook` | Stored replay only while the live HIP-4 L2 bridge is paused; use REST for current snapshots |
| `hip4_open_interest` | Stored replay only while the live HIP-4 OI bridge is paused; use REST for current outcome-side OI |

**Order-level (live; core L4 also replays):**

| Channel | Notes |
|---------|-------|
| `l4_diffs` | Hyperliquid L4 orderbook diffs with user attribution. Live and replay. |
| `l4_orders` | Hyperliquid order lifecycle events. Live and replay. |
| `hip3_l4_diffs` | HIP-3 L4 orderbook diffs |
| `hip3_l4_orders` | HIP-3 order lifecycle events |
| `hip4_l4_diffs` | HIP-4 L4 orderbook diffs |
| `hip4_l4_orders` | HIP-4 order lifecycle events |
| `spot_l4_diffs` | Hyperliquid Spot L4 orderbook diffs with user attribution |
| `spot_l4_orders` | Hyperliquid Spot order lifecycle events |

Core `l4_diffs` and `l4_orders` replay (single channel, history from 2026-03-10) starts with an `l4_snapshot` at the nearest checkpoint at or before `start`, then streams ordered `l4_batch` pages; `speed` and seek do not apply. HIP-3, HIP-4, and Spot L4 channels are live only.

**TWAP (realtime):**

| Channel | Notes |
|---------|-------|
| `spot_twap` | Hyperliquid Spot TWAP execution status updates. Symbol is dashed (`HYPE-USDC`). |

**HIP-4 outcome events:**

| Event type | Notes |
|------------|-------|
| `outcome_settled` | Fired once per HIP-4 outcome when it resolves (`is_settled` flips to true). Payload includes `outcome_id`, `winning_side`, and the settled timestamp. |

#### Lighter channels

Lighter channels, for both deployments, are served on `wss://api.0xarchive.io/ws` only; a Lighter subscribe on `stream.0xarchive.io` returns an error pointing back to `wss://api.0xarchive.io/ws`. Mainnet symbols are the same as `GET /v1/lighter/instruments`, case-insensitive on subscribe and echoed uppercase. Lighter on Robinhood Chain uses its own `rh_lighter_*` channels, described [below](#lighter-on-robinhood-chain-channels). Live Lighter data is available on every tier and is metered per message, the same as Hyperliquid live data; per-tier subscription and connection limits and the limit of 10 subscribe operations per second per connection apply.

| Channel | Live subscribe | Replay |
|---------|----------------|--------|
| `lighter_orderbook` | Yes. Full top-20 book, one per second by default (`interval_ms` 100 to 5000) | Yes |
| `lighter_trades` | Yes. Two fill legs per trade | Yes |
| `lighter_open_interest` | Yes. Same message as `lighter_funding` | Yes |
| `lighter_funding` | Yes. Same message as `lighter_open_interest` | Yes |
| `lighter_candles` | No. Subscribe returns an error | Yes |
| `lighter_l3_orderbook` | No. Subscribe returns an error | Yes |

```json
{"op":"subscribe","channel":"lighter_orderbook","symbol":"BTC"}
{"op":"subscribe","channel":"lighter_orderbook","symbol":"BTC","interval_ms":250}
{"op":"subscribe","channel":"lighter_trades","symbol":"BTC"}
{"op":"unsubscribe","channel":"lighter_trades","symbol":"BTC"}
```

Acks and data use the same envelope as Hyperliquid live data: `{"type":"subscribed","channel":"lighter_orderbook","coin":"BTC","symbol":"BTC"}`, then `{"type":"data","channel":"lighter_orderbook","coin":"BTC","symbol":"BTC","data":{...}}`. Unsubscribe is acknowledged with `{"type":"unsubscribed",...}`.

**`lighter_orderbook`** `data` (truncated to 2 levels per side):

```json
{"coin":"BTC","time":1790294171459,"levels":[[{"px":"84368.7","sz":"0.00020","n":1},{"px":"84368.6","sz":"0.00020","n":1}],[{"px":"84368.8","sz":"0.05720","n":1},{"px":"84368.9","sz":"0.14223","n":1}]]}
```

- Every message is a full book of up to 20 levels per side, not a diff. `levels[0]` is bids, highest first; `levels[1]` is asks, lowest first.
- `px` and `sz` are decimal strings exactly as Lighter publishes them. `n` is always `1` (Lighter does not publish per-level order counts). `time` is Lighter's book update time in ms.
- The newest book is sent at most once per interval: 1000 ms by default, or `interval_ms` between 100 and 5000 inclusive. Each book sent is one metered message. The current book is sent on subscribe when one is available; illiquid markets can go minutes without a change.
- `interval_ms` is accepted only on `lighter_orderbook`. On another channel the subscribe returns `interval_ms is only supported on lighter_orderbook.`; an out-of-range value such as `50` returns `interval_ms must be between 100 and 5000 for lighter_orderbook (got 50). Leave it out for one book a second.`

**`lighter_trades`** `data` is an array of fills, two legs per trade sharing one `tid`:

```json
[{"coin":"BTC","side":"A","px":"84367.9","sz":"0.00003","time":1790294182211,"hash":"0000001dc8774b28000001a0d5d94943000000000000000000000000000000000000000000000000","tid":31944180930,"oid":562953419896990,"crossed":false,"dir":null,"fee":null,"fee_token":null,"closed_pnl":null,"start_position":"109.79011","users":["281474976623827"]},
 {"coin":"BTC","side":"B","px":"84367.9","sz":"0.00003","time":1790294182211,"hash":"0000001dc8774b28000001a0d5d94943000000000000000000000000000000000000000000000000","tid":31944180930,"oid":844421425107071,"crossed":true,"dir":null,"fee":null,"fee_token":null,"closed_pnl":null,"start_position":"0.03940","users":["713845"]}]
```

- `side` `A` is the ask side and `B` the bid side; `crossed: true` is the taker leg. `users` holds the Lighter account index as a string, `oid` is that side's order id, `start_position` is that account's signed position before the trade, and `hash` is the Lighter transaction hash. `time` is in ms.
- `fee`, `fee_token`, `closed_pnl`, and `dir` are always `null` in live messages; Lighter's live stream does not carry them.
- Count trades by distinct `tid`, not by array length. For volume, sum `sz` over one leg per `tid` (for example the `crossed: true` leg).
- Live trades are delivered in small batches as they happen and are preliminary. `GET /v1/lighter/trades/{symbol}` serves the finalized record, including fields the live stream does not carry such as fees, and returns reconciled trades only (see `meta.finalized_through`); `GET /v1/lighter/trades/{symbol}/recent` serves the preliminary tier.

**`lighter_open_interest`** and **`lighter_funding`** carry the same `data`:

```json
{"coin":"BTC","ctx":{"openInterest":"172706178.266310","funding":"0.000012","premium":"-0.000327","markPx":"84363.5","oraclePx":"84397.0","midPx":"84368.8","dayNtlVlm":"908611371.550746","dayBaseVlm":"10808.97087","prevDayPx":"84285.9","impactPxs":null}}
```

- `openInterest` is Lighter's reported open interest, the same value as `open_interest` from `GET /v1/lighter/openinterest/{symbol}/current`.
- `funding` is Lighter's current funding rate as a fraction (Lighter publishes percent; the value is divided by 100), the same as REST `funding_rate`. `premium` is also a fraction.
- `markPx` is the mark price, `oraclePx` Lighter's index price, `midPx` the mid price, `dayNtlVlm` 24h quote volume, and `dayBaseVlm` 24h base volume. `prevDayPx` is derived from the last trade price and Lighter's 24h percent change. `impactPxs` is always `null` (Lighter has no impact prices).
- Updates arrive as Lighter publishes them, about once per second per market. The latest values are sent on subscribe when available.

**Slow connections:** a client that falls behind `lighter_trades`, `lighter_open_interest`, or `lighter_funding` receives an `error` notice such as `Dropped ~N live lighter_trades messages for BTC: your connection fell behind the Lighter stream, and those trades were not delivered.` Persistent lag stops that subscription with `Stopped the lighter_trades stream for BTC: your connection is too slow to keep up. Re-subscribe to resume.` `lighter_orderbook` always sends the newest book, so it never delivers an older book in place of a newer one.

**Replay** (`{"op":"replay",...}`) works on all six Lighter channels. Replay sends `historical_data` rows in the stored replay format (for example orderbook rows with `bids`/`asks` and fill rows with `price`/`size`/`tradeId`), which differs from the live payloads above; parse each with its own schema.

#### Lighter on Robinhood Chain channels

The Robinhood Chain deployment has its own channels, prefixed `rh_lighter_`. Live messages are identical in shape to the mainnet Lighter live messages above (`levels` books, two fill legs per trade, and `ctx` for open interest and funding), with prices in USDG. Symbols come from `GET /v1/rh-lighter/instruments`: uppercase perps (`BTC`) and dashed spot (`AAPL-USDG`). In `rh_lighter_trades`, `users` holds Robinhood Chain account indices, unrelated to mainnet indices with the same number. Live trades are preliminary; `GET /v1/rh-lighter/trades/{symbol}` serves the finalized record.

| Channel | Live subscribe | Replay |
|---------|----------------|--------|
| `rh_lighter_orderbook` | Yes. Full top-20 book, one per second by default (`interval_ms` 100 to 5000) | Yes, from 2026-08-22 18:43 UTC |
| `rh_lighter_trades` | Yes. Two fill legs per trade | Yes, from 2026-06-26 20:10:26 UTC |
| `rh_lighter_open_interest` | Yes. Perps; same message as `rh_lighter_funding` | Yes, from 2026-08-22 18:43 UTC |
| `rh_lighter_funding` | Yes. Perps; same message as `rh_lighter_open_interest` | Yes, from 2026-08-22 18:43 UTC |
| `rh_lighter_candles` | No. Subscribe returns an error | Yes, from 2026-06-26 once candles are enabled for this deployment |

```json
{"op":"subscribe","channel":"rh_lighter_orderbook","symbol":"AAPL-USDG","interval_ms":500}
{"op":"subscribe","channel":"rh_lighter_trades","symbol":"BTC"}
{"op":"replay","channel":"rh_lighter_trades","symbol":"BTC","start":1782518400000,"end":1782522000000,"speed":10}
```

- `interval_ms` is accepted only on `rh_lighter_orderbook` (and `lighter_orderbook` on mainnet); errors name the deployment's book channel, for example `interval_ms is only supported on rh_lighter_orderbook.`
- Slow-connection notices and stops work as on mainnet, naming the `rh_lighter_*` channel.
- A multi-channel replay cannot mix `lighter_*` and `rh_lighter_*` channels: they are separate exchange families.

### Web3 Authentication (`/v1`)

Use SIWE for existing wallet accounts and x402 for paid wallet access. Free accounts are created through the standard signup at `https://www.0xarchive.io/signup`. No API key is required for these Web3 endpoints.

| Endpoint | Params | Notes |
|----------|--------|-------|
| `POST /auth/web3/challenge` | `address` (wallet address) | Returns a SIWE message to sign; does not create an account |
| `POST /auth/web3/verify` | `message`, `signature` | Signs in an existing active wallet account; an unknown wallet returns 403 and no account is created |
| `POST /web3/signup` | -- | Retired: always returns HTTP 410 `wallet_free_signup_retired` and never creates an account or key |
| `POST /web3/keys` | `message`, `signature` | List all keys for wallet |
| `POST /web3/keys/revoke` | `message`, `signature`, `key_id` | Revoke a key |
| `POST /web3/subscribe` | `tier` (`build` or `pro`), `payment-signature` header | x402 USDC subscription (see flow below) |

**Existing-wallet flow:** Call `/auth/web3/challenge` with the wallet address → sign the returned message with `personal_sign` (EIP-191) → submit it to `/auth/web3/verify`, or to `/web3/keys` to list that wallet's keys. Verification only signs in an existing wallet account.

**Free tier:** sign up at `https://www.0xarchive.io/signup`. The legacy `/web3/signup` route always returns HTTP 410.

**Paid-tier flow (x402):**

1. `POST /web3/subscribe` with `{ "tier": "build" }` → server returns 402 with `payment.amount` (micro-USDC), `payment.pay_to` (treasury address), `payment.network`.
2. Sign an EIP-712 `TransferWithAuthorization` (EIP-3009) on USDC Base:
   - Domain: `{ name: "USD Coin", version: "2", chainId: 8453, verifyingContract: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913" }`
   - Type: `TransferWithAuthorization(address from, address to, uint256 value, uint256 validAfter, uint256 validBefore, bytes32 nonce)`
   - Message: `{ from: <wallet>, to: <pay_to>, value: <amount>, validAfter: 0, validBefore: <now+3600>, nonce: <32 random bytes hex> }`
3. Build x402 v2 payment payload:
   ```json
   {
     "x402Version": 2,
     "payload": {
       "signature": "0x<EIP-712 signature hex>",
       "authorization": {
         "from": "0x<wallet>",
         "to": "0x<pay_to from step 1>",
         "value": "<amount as string>",
         "validAfter": "0",
         "validBefore": "<unix timestamp as string>",
         "nonce": "0x<64 hex chars>"
       }
     }
   }
   ```
4. Base64-encode the JSON and retry: `POST /web3/subscribe` with `{ "tier": "build" }` and header `payment-signature: <base64 payload>` → receive API key + subscription.

**Important:** All `authorization` values (`value`, `validAfter`, `validBefore`) must be strings, not numbers. See `scripts/web3_subscribe.py` for a complete working Python implementation.

## Common Parameters

| Param | Type | Description |
|-------|------|-------------|
| `start` | int | Start timestamp (Unix ms). Defaults to 24h ago. |
| `end` | int | End timestamp (Unix ms). Defaults to now. |
| `limit` | int | Max records. Default 100; candle limits are route-specific: max 1000 for Spot and HIP-4, max 10000 for core Hyperliquid, HIP-3, and both Lighter deployments. Account positions routes have their own limits (see [Account Positions](#account-positions)). |
| `cursor` | string | Pagination cursor from `meta.next_cursor`. |
| `interval` | string | Candle interval: `1m`, `5m`, `15m`, `30m`, `1h`, `4h`, `1d`, `1w`. Default: `1h`. For OI/funding: `5m`, `15m`, `30m`, `1h`, `4h`, `1d`. Omit for raw data: core funding is roughly 1 minute; HIP-3 funding/OI, HIP-4 OI, and Lighter funding/OI are roughly 10 seconds. |
| `depth` | int | Route-specific orderbook depth. Hyperliquid-family native L2 caps at 20 levels per side; Lighter L3 caps at 250 orders per side. |
| `granularity` | string | Lighter orderbook resolution (both deployments): `checkpoint` (default), `30s`, `10s`, `1s`, `tick`. |
| `account` | int | Lighter mainnet L3 orderbook: filter by account index (e.g., `281474976710654` for LLP vault). |

## Smart Defaults

When the user does not specify a time range, default to the **last 24 hours**:

```bash
NOW=$(( $(date +%s) * 1000 ))
DAY_AGO=$(( NOW - 86400000 ))
```

For candles with no explicit range, default to a range that makes sense for the interval (e.g., last 7 days for 4h candles, last 30 days for 1d candles).

## Trade Response Fields

Each trade/fill record includes:

| Field | Type | Description |
|-------|------|-------------|
| `coin` / `symbol` | string | Trading pair symbol |
| `side` | string | `B` (buy) or `A`/`S` (sell) |
| `price` | string | Execution price |
| `size` | string | Trade size |
| `timestamp` | string | ISO 8601 timestamp |
| `trade_id` | integer | Unique trade ID |
| `order_id` | integer | Associated order ID |
| `crossed` | boolean | `true` = taker, `false` = maker |
| `fee` | string | Base trading fee |
| `fee_token` | string | Fee denomination (e.g., USDC) |
| `closed_pnl` | string | Realized PnL if closing position |
| `direction` | string | `Open Long`, `Close Short`, `Long > Short`, etc. |
| `start_position` | string | Position size before trade |
| `user_address` | string | User's wallet address |
| `builder_address` | string | Builder address that routed this order. Only present when the order was placed through a builder. |
| `builder_fee` | string | Builder fee charged on this fill, paid to the builder (quote currency, typically USDC). Only present when `builder_address` is set. |
| `deployer_fee` | string | HIP-3 deployer fee share (quote currency). Negative for the maker side (rebate), positive for the taker side. HIP-3 only. |
| `priority_gas` | number | Priority fee **burned in HYPE** (not USDC) for write priority on the Hyperliquid validator queue. Independent of `builder_fee` and `deployer_fee` (paid to the network, not to a builder or deployer). Only present when the order paid for priority. |
| `cloid` | string | Client order ID |
| `twap_id` | integer | TWAP execution ID |

`builder_address`, `builder_fee`, `deployer_fee`, `priority_gas`, `cloid`, and `twap_id` are optional. They are only present when non-zero/non-empty. `deployer_fee` is specific to HIP-3. `priority_gas` appears on any order that paid for write priority (most common on HIP-3 IOC orders).

## Pagination

When `meta.next_cursor` is present in the response, more data is available. Append `&cursor=VALUE` to fetch the next page:

```bash
# First page
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/trades/BTC?start=$START&end=$END&limit=1000"

# Next page (use next_cursor from previous response)
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/trades/BTC?start=$START&end=$END&limit=1000&cursor=1706000000000_12345"
```

## Tier Limits

| Tier | Price | Credits | Coins | Orderbook Depth | Lighter Granularity | Historical Depth | Rate Limit |
|------|-------|---------|-------|-----------------|---------------------|------------------|------------|
| Free | $0 | 50,000/mo | All symbols | Full depth | all granularities | Rolling 30 days (30-day span) | 15 RPS |
| Build | $49/mo | 80M/mo | All symbols | Full depth | all granularities | Full history | 50 RPS |
| Pro | $199/mo | 400M/mo | All symbols | Full depth | all granularities | Full history | 150 RPS |
| Scale | $799/mo | 2B/mo | All symbols | Full depth | all granularities | Full history | 500 RPS |
| Enterprise | Custom | Unlimited | All symbols | Full depth | + tick | Full history | Custom |

Scale ($799/mo, $639 annual) also includes 20,000 WebSocket subscriptions across 16 connections, 300x replay speed, and 200 API keys.

## Error Handling

| HTTP Status | Meaning | Action |
|-------------|---------|--------|
| 400 | Bad request / validation error | Check params (missing start/end, invalid interval) |
| 401 | Missing or invalid API key | Set `$OXARCHIVE_API_KEY` |
| 403 | Plan limit reached | Hit a credit, RPS, concurrency, WebSocket-cap, or export limit; upgrade plan or wait for reset (all markets and schemas are available on every tier; history age limits apply on Free) |
| 404 | Symbol not found | Check coin name spelling and exchange |
| 409 | Positions snapshot advanced (`snapshot_advanced`) | The snapshot a positions cursor was paging was replaced or expired; restart pagination without a cursor |
| 429 | Rate limited | Back off and retry |

Error responses return `{ "code": 400, "error": "description" }`.

## Example Queries

```bash
# List Hyperliquid instruments
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/instruments" | jq '.data | length'

# Current BTC orderbook (top 10 levels)
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/orderbook/BTC?depth=10" | jq '.data'

# ETH trades from the last hour
NOW=$(( $(date +%s) * 1000 )); HOUR_AGO=$(( NOW - 3600000 ))
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/trades/ETH?start=$HOUR_AGO&end=$NOW&limit=100" | jq '.data'

# SOL 4h candles for the last week
NOW=$(( $(date +%s) * 1000 )); WEEK_AGO=$(( NOW - 604800000 ))
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/candles/SOL?start=$WEEK_AGO&end=$NOW&interval=4h" | jq '.data'

# Current BTC funding rate
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/funding/BTC/current" | jq '.data'

# BTC open interest aggregated to 1h intervals (last week)
NOW=$(( $(date +%s) * 1000 )); WEEK_AGO=$(( NOW - 604800000 ))
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/openinterest/BTC?start=$WEEK_AGO&end=$NOW&interval=1h" | jq '.data'

# ETH funding rates aggregated to 4h intervals (last 30 days)
NOW=$(( $(date +%s) * 1000 )); MONTH_AGO=$(( NOW - 2592000000 ))
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/funding/ETH?start=$MONTH_AGO&end=$NOW&interval=4h" | jq '.data'

# HIP-3 km:US500 current orderbook
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/hip3/orderbook/km:US500" | jq '.data'

# HIP-3 km:US500 orderbook history
NOW=$(( $(date +%s) * 1000 )); HOUR_AGO=$(( NOW - 3600000 ))
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/hip3/orderbook/km:US500/history?start=$HOUR_AGO&end=$NOW&limit=10" | jq '.data'

# HIP-3 km:US500 candles (last 24h, 1h interval)
NOW=$(( $(date +%s) * 1000 )); DAY_AGO=$(( NOW - 86400000 ))
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/hip3/candles/km:US500?start=$DAY_AGO&end=$NOW&interval=1h" | jq '.data'

# HIP-4 list all outcome markets
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/hip4/outcomes" | jq '.data'

# HIP-4 single outcome market detail (outcome_id = 0)
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/hip4/outcomes/0" | jq '.data'

# HIP-4 orderbook for outcome 0 / side 0 (coin = "0", canonical bare numeric form)
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/hip4/orderbook/0?depth=10" | jq '.data'

# HIP-4 implied-probability price history for outcome 0 / side 1 (last 24h)
NOW=$(( $(date +%s) * 1000 )); DAY_AGO=$(( NOW - 86400000 ))
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/hip4/prices/1?start=$DAY_AGO&end=$NOW&interval=1h" | jq '.data'

# HIP-4 recent trades for outcome 0 / side 0
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/hip4/trades/0/recent?limit=20" | jq '.data'

# HIP-4 list active outcomes (not yet settled)
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/hip4/outcomes?is_settled=false" | jq '.data'

# Hyperliquid Spot list current pairs
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/spot/pairs" | jq '.data | length'

# Hyperliquid Spot single pair detail (HYPE-USDC, dashed canonical)
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/spot/pairs/HYPE-USDC" | jq '.data'

# Hyperliquid Spot current orderbook (top 10 levels)
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/spot/orderbook/HYPE-USDC?depth=10" | jq '.data'

# Hyperliquid Spot trades for the last hour (PURR-USDC)
NOW=$(( $(date +%s) * 1000 )); HOUR_AGO=$(( NOW - 3600000 ))
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/spot/trades/PURR-USDC?start=$HOUR_AGO&end=$NOW&limit=100" | jq '.data'

# Hyperliquid Spot trades filtered to a specific wallet
NOW=$(( $(date +%s) * 1000 )); DAY_AGO=$(( NOW - 86400000 ))
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/spot/trades/HYPE-USDC?start=$DAY_AGO&end=$NOW&user=0xYourWalletHere" | jq '.data'

# Hyperliquid Spot TWAP statuses for a symbol (last hour)
NOW=$(( $(date +%s) * 1000 )); HOUR_AGO=$(( NOW - 3600000 ))
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/spot/twap/HYPE-USDC?start=$HOUR_AGO&end=$NOW" | jq '.data'

# Hyperliquid Spot TWAP statuses for a wallet
NOW=$(( $(date +%s) * 1000 )); DAY_AGO=$(( NOW - 86400000 ))
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/spot/twap/user/0xYourWalletHere?start=$DAY_AGO&end=$NOW" | jq '.data'

# Hyperliquid Spot data freshness (per data type)
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/spot/freshness/HYPE-USDC" | jq '.data'

# Lighter BTC orderbook history (30s granularity, last hour)
NOW=$(( $(date +%s) * 1000 )); HOUR_AGO=$(( NOW - 3600000 ))
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/lighter/orderbook/BTC/history?start=$HOUR_AGO&end=$NOW&granularity=30s&limit=100" | jq '.data'

# Lighter on Robinhood Chain markets (USDG-quoted perps like BTC and spot like AAPL-USDG)
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/rh-lighter/instruments" | jq '.data | length'

# Lighter on Robinhood Chain AAPL-USDG orderbook (top 10 levels)
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/rh-lighter/orderbook/AAPL-USDG?depth=10" | jq '.data'

# Lighter on Robinhood Chain finalized BTC trades from 2 days ago to 1 day ago (end is clamped to meta.finalized_through)
NOW=$(( $(date +%s) * 1000 )); DAY_AGO=$(( NOW - 86400000 )); TWO_DAYS_AGO=$(( NOW - 172800000 ))
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/rh-lighter/trades/BTC?start=$TWO_DAYS_AGO&end=$DAY_AGO&limit=100" | jq '{finalized_through: .meta.finalized_through, trades: .data}'

# Lighter on Robinhood Chain recent trades, including preliminary rows
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/rh-lighter/trades/AAPL-USDG/recent?limit=20" | jq '{preliminary: .meta.preliminary_row_count, trades: .data}'

# Lighter on Robinhood Chain current BTC funding rate (perps only)
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/rh-lighter/funding/BTC/current" | jq '.data'

# Current positions and account summary for a Hyperliquid wallet
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/wallets/0xYourWalletHere/positions" | jq '{as_of: .meta.as_of, positions: .data.positions, account: .data.account}'

# Hyperliquid wallet positions as of 3 days ago (reconstructed unless the instant is an exact committed hour)
NOW=$(( $(date +%s) * 1000 )); T=$(( NOW - 259200000 ))
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/wallets/0xYourWalletHere/positions?timestamp=$T" | jq '{source: .meta.source, positions: .data.positions}'

# BTC position changes for a wallet over the last 24h (one row per fill leg)
NOW=$(( $(date +%s) * 1000 )); DAY_AGO=$(( NOW - 86400000 ))
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/wallets/0xYourWalletHere/positions/changes?start=$DAY_AGO&end=$NOW&symbol=BTC" | jq '.data'

# HIP-3 positions for a wallet on one dex
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/hip3/wallets/0xYourWalletHere/positions?dex=xyz" | jq '.data.positions'

# Largest BTC longs worth at least $1M, with market totals
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/positions/BTC?side=long&min_value=1000000&limit=20" | jq '{totals: .meta.totals, positions: .data}'

# BTC long/short positioning summary, hourly for the last 24h
NOW=$(( $(date +%s) * 1000 )); DAY_AGO=$(( NOW - 86400000 ))
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/positions/BTC/summary?start=$DAY_AGO&end=$NOW" | jq '.data'

# Lighter mainnet: resolve an L1 address to account indices, then read one account's positions
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/lighter/accounts?l1_address=0xYourWalletHere" | jq '.data.accounts'
ACCOUNT_INDEX=123456
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/lighter/accounts/$ACCOUNT_INDEX/positions" | jq '.data'

# Lighter on Robinhood Chain positions for one of its own account indices (no L1 lookup on this deployment).
# Take the index from a Robinhood Chain trade, liquidation, or /v1/rh-lighter/positions/{symbol} row,
# never from the mainnet lookup above: the same number is a different account here.
RH_ACCOUNT_INDEX=654321
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/rh-lighter/accounts/$RH_ACCOUNT_INDEX/positions" | jq '.data'

# Every open Hyperliquid position at one committed hour (bulk; follow meta.next_cursor)
NOW=$(( $(date +%s) * 1000 )); HOUR=$(( (NOW / 3600000 - 2) * 3600000 ))
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/positions?hour=$HOUR&limit=1000" | jq '{snapshot: .meta.snapshot_ts, next: .meta.next_cursor, rows: (.data | length)}'

# Account positions freshness per venue
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/data-quality/positions" | jq '.data'

# System health status
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/data-quality/status" | jq '.'

# SLA report for current month
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/data-quality/sla" | jq '.'

# BTC market summary (price, funding, OI, volume, liquidations in one call)
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/summary/BTC" | jq '.data'

# BTC data freshness (lag per data type)
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/freshness/BTC" | jq '.data'

# BTC price history (mark/oracle/mid) aggregated to 1h
NOW=$(( $(date +%s) * 1000 )); DAY_AGO=$(( NOW - 86400000 ))
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/prices/BTC?start=$DAY_AGO&end=$NOW&interval=1h" | jq '.data'

# BTC liquidation volume aggregated to 4h buckets
NOW=$(( $(date +%s) * 1000 )); WEEK_AGO=$(( NOW - 604800000 ))
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/liquidations/BTC/volume?start=$WEEK_AGO&end=$NOW&interval=4h" | jq '.data'

# Data coverage for Hyperliquid BTC
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/data-quality/coverage/hyperliquid/BTC" | jq '.'
```

