---
name: 0xarchive
version: 1.10.0
description: >
  Query historical and real-time crypto market data from 0xArchive across two top-level venue APIs: Hyperliquid and Lighter.xyz.
  HIP-3 builder perps live under the Hyperliquid namespace at /v1/hyperliquid/hip3.
  HIP-4 outcome markets (binary prediction markets like 'Will BTC be >= X by date Y?') live at /v1/hyperliquid/hip4.
  Hyperliquid Spot lives at /v1/hyperliquid/spot with 294 spot pairs (HYPE-USDC, PURR-USDC, AAPL-USDC, ...).
  Covers orderbooks, trades, candles, funding rates, open interest, liquidations (historical + realtime WS), outcome markets, spot, TWAP, and data quality.
  Real-time WebSocket channels include trades, liquidations, hip3_liquidations, orderbooks, HIP-4 channels, spot channels, L4 order-level data, TWAP, and the outcome_settled event for HIP-4 resolutions.
  Use in Claude Code, Codex with skills enabled, and SKILL.md-compatible agents when the user asks about crypto market data, orderbooks, trades, funding rates, historical prices, real-time streams, prediction-market outcomes, or spot pairs on Hyperliquid, Lighter.xyz, Hyperliquid HIP-3, Hyperliquid HIP-4, or Hyperliquid Spot.
allowed-tools: Bash
argument-hint: "query, e.g. 'BTC funding rate' or 'HYPE-USDC spot trades last hour'"
metadata: {"openclaw":{"requires":{"env":["OXARCHIVE_API_KEY"]},"primaryEnv":"OXARCHIVE_API_KEY"}}
---

# 0xArchive API Skill

Query historical and real-time crypto market data from **0xArchive** using `curl`. 0xArchive exposes two top-level venue APIs: **Hyperliquid** and **Lighter.xyz**. **HIP-3** builder perps live under the Hyperliquid namespace at `/v1/hyperliquid/hip3`. **HIP-4** outcome markets (binary prediction markets) live at `/v1/hyperliquid/hip4`. **Hyperliquid Spot** (294 pairs) lives at `/v1/hyperliquid/spot`. Data types: orderbooks, trades, candles, funding rates, open interest, liquidations, outcome markets, spot, TWAP, and data quality metrics.

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
| Lighter | `/v1/lighter` | UPPERCASE | `BTC`, `ETH` |

Hyperliquid and Lighter auto-uppercase the symbol server-side. HIP-3 coin names are passed through as-is. HIP-4 coins encode outcome and side: `0` is outcome 0 / side 0 (YES), `1` is outcome 0 / side 1 (NO), `10` is outcome 1 / side 0, etc. The bare numeric form is canonical; the legacy `#0` and `%230` forms still work for backward compatibility. Spot symbols are dashed (`HYPE-USDC`, `PURR-USDC`, `AAPL-USDC`); the server resolves the dashed form to the wire format (`PURR/USDC`, `@107`) internally, so always use the dashed form.

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
    "next_cursor": "1706000000000"   // present when more pages exist
  }
}
```

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
| `GET /orders/{symbol}/history` | `start`, `end`, `user`, `status`, `order_type`, `limit`, `cursor` | Order history with user attribution (Build+) |
| `GET /orders/{symbol}/flow` | `start`, `end`, `interval`, `limit` | Order flow aggregation (Build+) |
| `GET /orders/{symbol}/tpsl` | `start`, `end`, `user`, `triggered`, `limit`, `cursor` | TP/SL order history (Pro+) |
| `GET /orderbook/{symbol}/l4` | `timestamp`, `depth` | L4 orderbook reconstruction (Pro+) |
| `GET /orderbook/{symbol}/l4/diffs` | `start`, `end`, `limit`, `cursor` | L4 orderbook diffs (Pro+) |
| `GET /orderbook/{symbol}/l4/history` | `start`, `end`, `limit`, `cursor` | L4 orderbook checkpoints (Pro+) |
| `GET /orderbook/{symbol}/l2` | `timestamp`, `depth` | L2 full-depth orderbook derived from L4 (Build+) |
| `GET /orderbook/{symbol}/l2/history` | `start`, `end`, `limit`, `cursor`, `depth` | L2 full-depth checkpoints (Build+) |
| `GET /orderbook/{symbol}/l2/diffs` | `start`, `end`, `limit`, `cursor` | L2 tick-level diffs (Pro+) |

### HIP-3 (`/v1/hyperliquid/hip3`)

Coin names are **case-sensitive** (e.g., `km:US500`). Free tier includes km:US500 for orderbook and orderbook history; Build+ includes the broader HIP-3 symbol set.

| Endpoint | Params | Notes |
|----------|--------|-------|
| `GET /instruments` | -- | List HIP-3 instruments |
| `GET /instruments/{coin}` | -- | Single instrument |
| `GET /orderbook/{coin}` | `timestamp`, `depth` | Free: km:US500 only. Build+: all HIP-3 symbols. |
| `GET /orderbook/{coin}/history` | `start`, `end`, `limit`, `cursor`, `depth` | Free: km:US500 only. Build+: all HIP-3 symbols. |
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
| `GET /orders/{coin}/history` | `start`, `end`, `user`, `status`, `order_type`, `limit`, `cursor` | Order history with user attribution (Build+) |
| `GET /orders/{coin}/flow` | `start`, `end`, `interval`, `limit` | Order flow aggregation (Build+) |
| `GET /orders/{coin}/tpsl` | `start`, `end`, `user`, `triggered`, `limit`, `cursor` | TP/SL order history (Pro+) |
| `GET /orderbook/{coin}/l4` | `timestamp`, `depth` | L4 orderbook reconstruction (Pro+) |
| `GET /orderbook/{coin}/l4/diffs` | `start`, `end`, `limit`, `cursor` | L4 orderbook diffs (Pro+) |
| `GET /orderbook/{coin}/l4/history` | `start`, `end`, `limit`, `cursor` | L4 orderbook checkpoints (Pro+) |
| `GET /orderbook/{coin}/l2` | `timestamp`, `depth` | L2 full-depth orderbook derived from L4 (Build+) |
| `GET /orderbook/{coin}/l2/history` | `start`, `end`, `limit`, `cursor`, `depth` | L2 full-depth checkpoints (Build+) |
| `GET /orderbook/{coin}/l2/diffs` | `start`, `end`, `limit`, `cursor` | L2 tick-level diffs (Pro+) |

### HIP-4 (`/v1/hyperliquid/hip4`)

Outcome markets are binary prediction markets (e.g. "Will BTC be >= $X by date Y?"). Coin names are bare numerics `<10*outcome_id + side>` (e.g. `0`, `1`, `10`); the legacy `#0` / `%230` forms are still accepted. HIP-4 has **no funding rates, no liquidations, and no candles** -- those endpoints do not exist on this venue. The `mark_price` field on HIP-4 instruments and prices is an **implied probability in the range 0..1**, not a USD price.

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
| `GET /openinterest/{coin}/current` | -- | Current open interest |
| `GET /openinterest/{coin}` | `start`, `end`, `limit`, `cursor`, `interval` | OI history |
| `GET /freshness/{coin}` | -- | Data freshness per data type |
| `GET /summary/{coin}` | -- | Combined market summary (implied probability + OI; no funding) |
| `GET /prices/{coin}` | `start`, `end`, `limit`, `cursor`, `interval` | Implied-probability history (mark/oracle/mid in 0..1) |
| `GET /orders/{coin}/history` | `start`, `end`, `user`, `status`, `order_type`, `limit`, `cursor` | Order history (Build+) |
| `GET /orders/{coin}/flow` | `start`, `end`, `interval`, `limit` | Order flow aggregation (Build+) |
| `GET /orders/{coin}/tpsl` | `start`, `end`, `user`, `triggered`, `limit`, `cursor` | TP/SL order history (Pro+) |
| `GET /orderbook/{coin}/l4` | `timestamp`, `depth` | L4 orderbook reconstruction (Pro+) |
| `GET /orderbook/{coin}/l4/diffs` | `start`, `end`, `limit`, `cursor` | L4 orderbook diffs (Pro+) |
| `GET /orderbook/{coin}/l4/history` | `start`, `end`, `limit`, `cursor` | L4 orderbook checkpoints (Pro+) |
| `GET /orderbook/{coin}/l2` | `timestamp`, `depth` | L2 full-depth orderbook derived from L4 (Build+) |
| `GET /orderbook/{coin}/l2/history` | `start`, `end`, `limit`, `cursor`, `depth` | L2 full-depth checkpoints (Build+) |
| `GET /orderbook/{coin}/l2/diffs` | `start`, `end`, `limit`, `cursor` | L2 tick-level diffs (Pro+) |

### Hyperliquid Spot (`/v1/hyperliquid/spot`)

294 spot pairs on Hyperliquid (HYPE-USDC, PURR-USDC, AAPL-USDC, ...). Symbols are **dashed canonical** (`BASE-QUOTE`); the server resolves to the wire format (`PURR/USDC`, `@107`) internally. Spot has **no funding rates, no open interest, no liquidations, and no candles** -- those are perp-only constructs and the endpoints do not exist on this venue. Use spot for pair discovery, current and historical L2 orderbooks, fills, L4 reconstruction, order lifecycle, and TWAP execution status.

Coverage:
- **Trades**: backfilled from 2025-03-22 (~284M rows). Pre-March 2025 spot fills are not available (no public archive existed).
- **Orderbook, L4, TWAP**: live-forward from 2026-05-05. No historical orderbook data prior to that date.

| Endpoint | Params | Notes |
|----------|--------|-------|
| `GET /pairs` | -- | List all 294 spot pairs |
| `GET /pairs/{symbol}` | -- | Single pair detail (e.g. `HYPE-USDC`) |
| `GET /orderbook/{symbol}` | `timestamp`, `depth` | Current L2 orderbook (live from 2026-05-05) |
| `GET /orderbook/{symbol}/history` | `start`, `end`, `limit`, `cursor`, `depth` | L2 history window (from 2026-05-05) |
| `GET /orderbook/{symbol}/l4` | `timestamp`, `depth` | Point-in-time L4 reconstruction (Pro+) |
| `GET /orderbook/{symbol}/l4/diffs` | `start`, `end`, `limit`, `cursor` | Raw L4 diffs (Pro+) |
| `GET /orderbook/{symbol}/l4/history` | `start`, `end`, `limit`, `cursor` | L4 checkpoints (Build+) |
| `GET /trades/{symbol}` | `start`, `end`, `limit`, `cursor`, `user` | Spot trades. Pass `user` to filter to a wallet's fills. Backfilled to 2025-03-22. |
| `GET /orders/{symbol}/history` | `start`, `end`, `user`, `status`, `order_type`, `limit`, `cursor` | Spot order lifecycle (Pro+) |
| `GET /twap/{symbol}` | `start`, `end`, `limit`, `cursor` | TWAP execution statuses for a symbol |
| `GET /twap/user/{user}` | `start`, `end`, `limit`, `cursor` | TWAP execution statuses for a wallet |
| `GET /freshness/{symbol}` | -- | Data freshness per data type |

### Lighter (`/v1/lighter`)

Same data types as Hyperliquid except: no liquidations. Adds `granularity` on orderbook history and `/recent` trades.

| Endpoint | Params | Notes |
|----------|--------|-------|
| `GET /instruments` | -- | List Lighter instruments |
| `GET /instruments/{symbol}` | -- | Single instrument |
| `GET /orderbook/{symbol}` | `timestamp`, `depth` | Latest or at timestamp |
| `GET /orderbook/{symbol}/history` | `start`, `end`, `limit`, `cursor`, `depth`, `granularity` | Default granularity: `checkpoint` |
| `GET /trades/{symbol}` | `start`, `end`, `limit`, `cursor` | Trade history |
| `GET /trades/{symbol}/recent` | `limit` | Recent trades (no time range needed) |
| `GET /candles/{symbol}` | `start`, `end`, `limit`, `cursor`, `interval` | OHLCV candles |
| `GET /funding/{symbol}/current` | -- | Current funding rate |
| `GET /funding/{symbol}` | `start`, `end`, `limit`, `cursor`, `interval` | Funding history |
| `GET /openinterest/{symbol}/current` | -- | Current OI |
| `GET /openinterest/{symbol}` | `start`, `end`, `limit`, `cursor`, `interval` | OI history |
| `GET /freshness/{symbol}` | -- | Data freshness per data type |
| `GET /summary/{symbol}` | -- | Combined market summary (price, funding, OI) |
| `GET /prices/{symbol}` | `start`, `end`, `limit`, `cursor`, `interval` | Mark/oracle price history |
| `GET /l3orderbook/{symbol}` | `timestamp`, `depth`, `account` | L3 order-level orderbook (Pro+) |
| `GET /l3orderbook/{symbol}/history` | `start`, `end`, `limit`, `cursor`, `granularity`, `account` | Historical L3 snapshots (Pro+) |

### Data Quality (`/v1/data-quality`)

| Endpoint | Params | Notes |
|----------|--------|-------|
| `GET /status` | -- | System health status |
| `GET /coverage` | -- | Coverage summary across venue APIs |
| `GET /coverage/{exchange}` | -- | Coverage for one venue scope |
| `GET /coverage/{exchange}/{symbol}` | `from`, `to` | Symbol-level coverage + gaps |
| `GET /incidents` | `status`, `exchange`, `since`, `limit`, `offset` | List incidents |
| `GET /incidents/{id}` | -- | Single incident |
| `GET /latency` | -- | Ingestion latency metrics |
| `GET /sla` | `year`, `month` | SLA compliance report |

### WebSocket Channels

Real-time + historical-replay channels available via WebSocket (`wss://api.0xarchive.io/ws?apiKey=KEY`). Build+ tier required for any WS access; Pro+ for L4 channels.

**Trades + liquidations (realtime + replay):**

| Channel | Notes |
|---------|-------|
| `trades` | Hyperliquid trades. One row per side per fill. |
| `hip3_trades` | HIP-3 trades. |
| `hip4_trades` | HIP-4 trades. |
| `spot_trades` | Hyperliquid Spot trades (Build+). Symbol is dashed (`HYPE-USDC`). |
| `lighter_trades` | Lighter trades. |
| `liquidations` | Hyperliquid liquidations. **Each event is a fill row with `is_liquidation: true` (same shape as `trades`).** |
| `hip3_liquidations` | HIP-3 liquidations. **Each event is a fill row with `is_liquidation: true` (same shape as `hip3_trades`).** |

**Orderbook + open interest (realtime + replay):**

| Channel | Notes |
|---------|-------|
| `orderbook`, `hip3_orderbook`, `hip4_orderbook`, `lighter_orderbook` | L2 orderbook updates (~1.2 sec resolution) |
| `spot_orderbook` | Hyperliquid Spot L2 orderbook updates (Build+). Symbol is dashed (`HYPE-USDC`). |
| `hip4_open_interest` | HIP-4 per-side OI snapshots |

**Order-level (realtime only, Pro+):**

| Channel | Notes |
|---------|-------|
| `l4_diffs` | Hyperliquid L4 orderbook diffs with user attribution |
| `l4_orders` | Hyperliquid order lifecycle events |
| `hip3_l4_diffs` | HIP-3 L4 orderbook diffs |
| `hip3_l4_orders` | HIP-3 order lifecycle events |
| `hip4_l4_diffs` | HIP-4 L4 orderbook diffs |
| `hip4_l4_orders` | HIP-4 order lifecycle events |
| `spot_l4_diffs` | Hyperliquid Spot L4 orderbook diffs with user attribution |
| `spot_l4_orders` | Hyperliquid Spot order lifecycle events |
| `lighter_l3_orderbook` | Lighter L3 order-level orderbook snapshots |

**TWAP (realtime, Build+):**

| Channel | Notes |
|---------|-------|
| `spot_twap` | Hyperliquid Spot TWAP execution status updates. Symbol is dashed (`HYPE-USDC`). |

**HIP-4 outcome events:**

| Event type | Notes |
|------------|-------|
| `outcome_settled` | Fired once per HIP-4 outcome when it resolves (`is_settled` flips to true). Payload includes `outcome_id`, `winning_side`, and the settled timestamp. |

### Web3 Authentication (`/v1`)

Get API keys programmatically using an Ethereum wallet (SIWE). No API key required for these endpoints.

| Endpoint | Params | Notes |
|----------|--------|-------|
| `POST /auth/web3/challenge` | `address` (wallet address) | Returns SIWE message to sign |
| `POST /web3/signup` | `message`, `signature` | Returns free-tier API key |
| `POST /web3/keys` | `message`, `signature` | List all keys for wallet |
| `POST /web3/keys/revoke` | `message`, `signature`, `key_id` | Revoke a key |
| `POST /web3/subscribe` | `tier` (`build` or `pro`), `payment-signature` header | x402 USDC subscription (see flow below) |

**Free-tier flow:** Call `/auth/web3/challenge` with wallet address → sign the returned message with `personal_sign` (EIP-191) → submit to `/web3/signup` with the message and signature → receive API key.

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
| `limit` | int | Max records. Default 100, max 1000 (max 10000 for candles). |
| `cursor` | string | Pagination cursor from `meta.next_cursor`. |
| `interval` | string | Candle interval: `1m`, `5m`, `15m`, `30m`, `1h`, `4h`, `1d`, `1w`. Default: `1h`. For OI/funding: `5m`, `15m`, `30m`, `1h`, `4h`, `1d`. Omit for raw data. |
| `depth` | int | Orderbook depth (number of price levels per side). |
| `granularity` | string | Lighter orderbook resolution: `checkpoint` (default), `30s`, `10s`, `1s`, `tick`. |
| `account` | int | Lighter L3 orderbook: filter by account index (e.g., `281474976710654` for LLP vault). |

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

| Tier | Price | Coins | Orderbook Depth | Lighter Granularity | Historical Depth | Rate Limit |
|------|-------|-------|-----------------|---------------------|------------------|------------|
| Free | $0 | BTC only (HIP-3: km:US500 only) | 20 levels | -- | 30 days | 15 RPS |
| Build | $49/mo | All | 200 levels | checkpoint, 30s, 10s | 1 year | 50 RPS |
| Pro | $199/mo | All | Full depth | + 1s | Full history | 150 RPS |
| Enterprise | Custom | All | Full depth | + tick | Full history | Custom |

## Error Handling

| HTTP Status | Meaning | Action |
|-------------|---------|--------|
| 400 | Bad request / validation error | Check params (missing start/end, invalid interval) |
| 401 | Missing or invalid API key | Set `$OXARCHIVE_API_KEY` |
| 403 | Tier restriction | Upgrade plan (e.g., non-BTC coin on Free tier) |
| 404 | Symbol not found | Check coin name spelling and exchange |
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

# HIP-3 km:US500 current orderbook (Free tier safe)
curl -s -H "x-api-key: $OXARCHIVE_API_KEY" \
  "https://api.0xarchive.io/v1/hyperliquid/hip3/orderbook/km:US500" | jq '.data'

# HIP-3 km:US500 orderbook history (Free tier safe)
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

# Hyperliquid Spot list all 294 pairs
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

