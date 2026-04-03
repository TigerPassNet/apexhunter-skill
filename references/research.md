# /apexhunter research — Strategy Generation

This document defines the complete flow for `/apexhunter research`. The goal: classify assets by alpha potential, discover oracle wallets, scan SM trading activity, analyze market structure, and generate TWO strategies — directional trades (Strategy A) and funding arbitrage (Strategy B) — with conviction-scored watchlist entries.

## Pre-flight

1. Check `nansen account` — confirm sufficient credits (need ~35 for full research)
2. If `~/.apexhunter/strategy.json` exists, note when it was generated — skip research if < 2 days old unless user forces it
3. Load `~/.apexhunter/oracle-wallets.json` if it exists (carry forward discovered oracles)
4. **Load `~/.apexhunter/layer1-anchor.json`** — this is your compass. Read the regime classification and `allowed_bias` BEFORE any signal analysis. All subsequent decisions must be consistent with Layer 1 unless overwhelming evidence warrants a regime change.

## Step -1: Asset Alpha Classification (0 credits)

Before any signal analysis, classify all tradeable assets into tiers. This determines which strategy and signal weights apply.

**Tier Classification Rules:**

| Asset | Tier | Why | Primary Strategy |
|-------|------|-----|-----------------|
| BTC, ETH | Tier 1 (High Efficiency) | ETF-dominated, macro-correlated, SM edge minimal | Macro event-driven (Strategy A with modified weights) |
| SOL, HYPE, BNB, DOGE, PEPE, major alts | Tier 2 (Medium Efficiency) | SM data has genuine alpha, reflexive cycles, narrative-driven | SM signal-driven (Strategy A, full weights) |
| xyz: stocks, xyz: commodities, newly listed perps (<30 days) | Tier 3 (Low Efficiency) | Structural mispricings, thin SM coverage | Funding arb (Strategy B) + price discovery |

**Asset-Adaptive Signal Weights (Strategy A — directional scoring only):**

Signal E (funding arb) is NOT included — it operates as independent Strategy B with its own capital pool.

| Signal | Tier 1 (BTC/ETH) | Tier 2 (SOL/HYPE/BNB) | Tier 3 (xyz:/new) |
|--------|------------------|----------------------|-------------------|
| Signal A: SM Perp Trades | 20% (less edge) | 40% (core edge) | 40% (sparse but very high signal) |
| Signal B: Oracle Wallets | 15% | 25% | 15% |
| Signal C: Market Structure | 30% (funding/OI more meaningful) | 15% | 25% |
| Signal D: Trend | 35% (macro drives BTC) | 20% | 20% |
| **Sum** | **100%** | **100%** | **100%** |

**Tier 3 note:** Tier 3 assets can generate directional conviction scores, but their primary alpha source is Strategy B (funding arbitrage). When a Tier 3 asset has BOTH a directional signal AND extreme funding, the directional trade takes priority (captures price + carry).

**Why this matters:** A Fund-labeled trader opening a $500K SOL SHORT is a much stronger signal than the same trade on BTC, because SOL has fewer institutional participants and the information is less likely to be already priced in. Conversely, BTC trend structure and macro news matter far more than SM perp trades because BTC pricing is driven by ETF flows and Fed policy, not by individual HL traders.

Store tier classification in strategy.json as `asset_tiers`.

### Layer 1 Anchor Check (before any signal analysis)

Before scanning SM trades or screener data, assess whether the Layer 1 anchor needs updating:

```bash
# 4h candles for 14-day trend (free)
curl -s -X POST https://api.hyperliquid.xyz/info \
  -H 'Content-Type: application/json' \
  -d '{"type":"candleSnapshot","req":{"coin":"BTC","interval":"4h","startTime":'$(date -v-14d +%s000)',"endTime":'$(date +%s000)'}}'

# BTC ETF flows (BlockBeats, free)
curl -s -H "api-key: $BLOCKBEATS_API_KEY" \
  "https://api-pro.theblockbeats.info/v1/data/btc_etf"

# Market sentiment (BlockBeats, free)
curl -s -H "api-key: $BLOCKBEATS_API_KEY" \
  "https://api-pro.theblockbeats.info/v1/data/bottom_top_indicator"
```

Check each pillar:
1. **Trend pillar:** BTC 14d range, 4h EMA20 vs EMA50, position within range
2. **Oracle pillar:** Updated in Step 2. Any position closures?
3. **Institutional pillar:** ETF flow direction and magnitude. 2+ days of inflow > $500M = bullish shift.

If any `regime_change_trigger` from the anchor is met → UPDATE anchor. Otherwise confirm and carry forward.

Write updated anchor to `~/.apexhunter/layer1-anchor.json`.

## Step 0: Market Regime Classification

Before any signal analysis, classify the current market regime using BTC as the reference.

```bash
# Get BTC candles (free, HL public API)
# HL public API (free, unlimited) — tigerpass does NOT support candles
curl -s -X POST https://api.hyperliquid.xyz/info \
  -H 'Content-Type: application/json' \
  -d '{"type":"candleSnapshot","req":{"coin":"BTC","interval":"1h","startTime":'$(date -v-3d +%s000)',"endTime":'$(date +%s000)'}}'
```

Compute:
- **20-period EMA** and **50-period EMA** on 1h closes
- **ATR(14)** on 1h candles (used for stop-loss sizing throughout)
- Current BTC price relative to both EMAs

| 20 EMA vs 50 EMA | Price vs 20 EMA | Regime |
|-------------------|-----------------|--------|
| 20 > 50 (bullish cross) | Above 20 EMA | **Risk-On** |
| Within 0.5% of each other | Near 20 EMA | **Ranging** |
| 20 < 50 (bearish cross) | Below 20 EMA | **Risk-Off** |

Record regime in strategy.json with BTC price at generation time.

**Circuit breaker check**: If BTC has dropped >12% in the last 7 days, set regime to PAUSED — no new strategies generated, only process exits in existing portfolio.

## Step 1: Signal A — SM Perp Trades Scan (~5 credits)

The most important signal. What are Nansen-labeled smart money traders doing on Hyperliquid right now?

```bash
# Get recent SM perp trades (last 24h, value > $5K)
nansen research smart-money perp-trades \
  --filters '{"include_smart_money_labels":["Smart HL Perps Trader","Fund","Smart Trader"],"value_usd":{"min":5000}}' \
  --sort block_timestamp:desc \
  --limit 200 \
  --fields trader_address,trader_address_label,token_symbol,side,action,value_usd,price_usd,type
```

**Process the data:**

For each token that appears:

1. **Count unique SM addresses** trading this token in the last 24h
2. **Categorize by action + side** (action is "Open"/"Close"/"Add"/"Reduce", direction from side):
   - side=Long + action=Open/Add → long signal
   - side=Short + action=Open/Add → short signal
   - side=Short + action=Close/Reduce → confirms long bias (SM exiting shorts)
   - side=Long + action=Close/Reduce → confirms short bias (SM exiting longs)
3. **Sum value_usd** per direction per token
4. **Apply label-based weighting** to value_usd:
   - Fund → 1.5x weight
   - Smart HL Perps Trader / Smart Trader → 1.0x
   - High Balance / High Activity → 0.5x (bots and large holders, less signal)
   - Unlabeled → 0.3x (only significant if value > $50K)
5. **Detect consensus vs division** — are SM traders agreeing on direction or split?
6. **Detect pairs trades** — if the same trader_address is opening OPPOSITE directions on related tokens (e.g., short NVDA + long INTC, or short HYPE + long BTC), flag as a pairs/relative-value trade. These are higher conviction because the trader is hedged.
7. **Track systematic execution patterns** — if a single trader makes >10 trades in the same token+direction within 1 hour (same size, similar prices), it's algorithmic execution. Weight these higher (institutional signal).

**Signal A scoring per token (raw score 0-1, then multiplied by tier weight in conviction formula):**

Conditions are evaluated top-down. Use the HIGHEST matching base condition, then add applicable bonuses. Raw score is capped at 1.0.

| Condition | Score | Type |
|-----------|-------|------|
| ≥3 independent SM same direction Open/Add (24h) | 0.70 | Base (use highest) |
| ≥2 SM same direction, each > $50K value | 0.50 | Base |
| Only 1 SM or value < $10K | 0.15 | Base |
| No SM activity | 0.00 | Base |
| Fund label participating | +0.20 bonus | Additive |
| SM closing/reducing opposite direction | +0.10 bonus | Additive |
| SM divided (activity in both directions) | cap at 0.25 | Override (caps score) |

**Example:** 3 SM traders opening longs (0.70 base) + Fund label (+0.20) = 0.90 raw → in conviction formula: `0.90 × tier_weight_A`

Score is computed **per direction** (long and short scored separately for each token).

**Sell watchlist generation**: If SM is actively closing positions in a token you hold, flag it immediately.

## Step 2: Signal B — Oracle Wallet Discovery & Tracking (~10 credits)

### Discovery (weekly — skip if done within 7 days)

```bash
# Find top profitable HL traders over last 30 days
nansen research perp leaderboard --days 30 \
  --filters '{"total_pnl":{"min":10000},"roi":{"min":0.20},"include_smart_money_labels":["Smart HL Perps Trader","Fund","Smart Trader"]}' \
  --sort total_pnl:desc \
  --limit 50 \
  --fields trader_address,trader_address_label,total_pnl,roi,account_value
```

**Oracle candidate criteria:**
- total_pnl > $10,000 in 30 days
- ROI > 20%
- account_value > $50,000 (filters out lucky small accounts)

For the top 10 candidates not already in oracle-wallets.json:

```bash
# Verify with per-token PnL (are they good at specific tokens or just one lucky trade?)
nansen research token perp-pnl-leaderboard --symbol BTC --days 30 \
  --fields trader_address,pnl_usd_total,roi_percent_total,nof_trades \
  --limit 20
```

Add candidates to `oracle-wallets.json` with status `new_candidate` and weight 0.25.

### Leaderboard Direction Bias (new — critical insight from real data)

From the leaderboard results, compute the **aggregate direction bias** of top traders:

```
long_count = number of top-20 traders whose largest position is LONG
short_count = number of top-20 traders whose largest position is SHORT
direction_bias = "LONG" if long_count > short_count else "SHORT"
bias_strength = abs(long_count - short_count) / 20
```

Real data example (2026-03-31): 8/8 active top traders were SHORT-dominant (ETH, BTC shorts). bias_strength = 1.0 (maximum). This is a meta-signal that should influence ALL entry decisions:

- If bias_strength > 0.6 → add +0.05 to all conviction scores in the bias direction, -0.05 to opposite
- If bias_strength > 0.8 → add +0.10 / -0.10
- Store in strategy.json as `leaderboard_direction_bias`

**This signal caught the bearish regime that individual SM trades didn't show clearly.**

### Tracking (every research cycle)

For each oracle wallet (up to 10 tracked):

```bash
# Check their current positions
nansen research profiler perp-positions --address <oracle_address> \
  --fields token_symbol,side,position_value_usd,leverage,entry_price,mark_price,upnl_usd
```

**Oracle status progression:**

| Status | Weight | Promotion Criteria |
|--------|--------|-------------------|
| new_candidate | 0.25 | Just discovered from leaderboard |
| strong_candidate | 0.50 | Tracked ≥1 week, net PnL positive during tracking |
| verified_oracle | 0.75 | Tracked ≥2 weeks, consistent positive PnL |
| elite_oracle | 1.00 | Tracked ≥3 weeks, ROI > 30%, win_rate > 55% |

**Demotion**: If an oracle's tracked trades show negative PnL for 2 consecutive weeks → demote one tier. If negative for 3 weeks → remove from tracking.

**Signal B scoring per token (max 0.25 after weighting):**

| Condition | Raw Score |
|-----------|-----------|
| elite/verified oracle has position in this token+direction | 0.15 |
| Oracle opened NEW position this cycle (wasn't there last check) | 0.10 |
| Oracle increased position size (adding) | 0.05 |
| Multiple oracles agree on same token+direction | +0.05 per additional |
| Oracle reducing position | -0.05 (warning signal) |

## Step 3: Signal C — Market Structure Analysis (~5 credits)

### Perp Screener

```bash
# Scan all perp markets with SM data
nansen research perp screener --days 1 \
  --sort volume:desc \
  --limit 30 \
  --fields token_symbol,volume,funding,open_interest,mark_price
```

**Extract per token:**
- `funding` — current funding rate (per 8h)
- `open_interest` — total OI in USD
- `sm_net_position_change` — net change in SM positions (positive = SM adding longs, negative = adding shorts)
- `sm_long_position_usd` vs `sm_short_position_usd` — SM position imbalance
- Price change: `(mark_price - previous_price) / previous_price`

### Nansen Indicators (for tokens on watchlist)

For each token that scored > 0 in Signal A or B, and has an EVM token address (BTC→WBTC, ETH→WETH on ethereum):

```bash
nansen research token indicators --chain ethereum --token <wbtc_address>
```

Extract:
- `price-momentum` score (bearish/neutral/bullish) + percentile
- `funding-rate` score + signal value
- `btc-reflexivity` score (how correlated to BTC)
- `liquidity-risk` score (low/medium/high)

**Signal C scoring per token per direction (max 0.20 after weighting):**

| Condition | Long Score | Short Score |
|-----------|-----------|-------------|
| Funding > 0.05%/8h (overheated long) | -0.10 | +0.15 |
| Funding < -0.03%/8h (overheated short) | +0.15 | -0.10 |
| OI surge >20% (24h) + SM same direction | +0.10 | +0.10 |
| SM net position change strongly positive | +0.10 | 0 |
| SM net position change strongly negative | 0 | +0.10 |
| Nansen momentum = bullish + >70th percentile | +0.05 | -0.05 |
| Nansen momentum = bearish + >70th percentile | -0.05 | +0.05 |
| Funding between -0.01% and 0.01% | 0 | 0 |

## Step 4: Signal D — Trend Confirmation (~5 credits)

### Price Trend (free — HL public API)

For each token on the emerging watchlist:

```bash
# HL public API for each watchlist token's candles
curl -s -X POST https://api.hyperliquid.xyz/info \
  -H 'Content-Type: application/json' \
  -d '{"type":"candleSnapshot","req":{"coin":"<TOKEN>","interval":"1h","startTime":'$(date -v-3d +%s000)',"endTime":'$(date +%s000)'}}'
```

Compute:
- 20-period EMA and 50-period EMA
- Current price position relative to 20 EMA
- ATR(14) — stored for stop-loss calculation during execute

### Capital Flow (Nansen)

```bash
# SM capital flow on major EVM chains (HyperEVM data too thin, use ethereum/base instead)
nansen research smart-money netflow --chain ethereum \
  --filters '{"include_smart_money_labels":["Smart HL Perps Trader","Fund"]}' \
  --sort net_flow_7d_usd:desc \
  --limit 20 \
  --fields token_symbol,net_flow_7d_usd,net_flow_30d_usd,trader_count
```

**Signal D scoring per token per direction (max 0.15 after weighting):**

| Condition | Long Score | Short Score |
|-----------|-----------|-------------|
| 20 EMA > 50 EMA (uptrend) | +0.10 | -0.05 |
| 20 EMA < 50 EMA (downtrend) | -0.05 | +0.10 |
| Price above 20 EMA | +0.05 | 0 |
| Price below 20 EMA | 0 | +0.05 |
| SM Netflow on ethereum for BTC/ETH increasing (7d > 30d ratio > 1.5) | +0.05 | 0 |
| SM Netflow on ethereum for BTC/ETH decreasing (7d < 0) | 0 | +0.05 |

## Step 4.5: Strategy B — Funding Arbitrage (Independent Sub-Strategy)

**This is NOT a signal for directional trades. This is a separate, parallel strategy with its own capital pool (30% of total capital).** Funding arb has a mathematical edge — when funding is extreme, you earn carry regardless of price direction. The edge comes from market structure (imbalanced positioning), not from prediction.

From the Perp Screener data (already fetched in Step 3), scan ALL assets across ALL tiers:

**Extreme Positive Funding (longs paying shorts → short opportunity):**
- Funding > 0.03%/8h (>260% annualized) → add to Strategy B watchlist as SHORT
- Funding > 0.06%/8h (>525% annualized) → HIGH CONVICTION SHORT (funding alone is the edge)
- Real example: Korean stocks HYUNDAI 0.09%/8h, SMSN 0.06%/8h, SKHX 0.06%/8h

**Extreme Negative Funding (shorts paying longs → long opportunity):**
- Funding < -0.02%/8h (>175% annualized) → add to Strategy B watchlist as LONG
- Funding < -0.04%/8h (>350% annualized) → HIGH CONVICTION LONG
- Real example: NUCLEAR -0.04%/8h, ZORA -0.027%/8h

**Strategy B position rules (completely independent from Strategy A):**
- **Capital pool:** 30% of total capital reserved for funding arb (e.g., $90 of $300)
- **Max per position:** 15% of Strategy B capital (e.g., $13.50)
- **Max concurrent:** up to 5 funding arb positions (diversify the carry)
- **Leverage:** 2x max (holding for funding, not price moves)
- **Hold period:** until funding normalizes below threshold (check every monitor cycle)
- **Stop-loss:** -8% from entry (wider than directional — willing to absorb noise to earn carry)
- **Take-profit:** none — exit when funding rate drops below threshold
- **CRITICAL liquidity check:** OI must be > $100K AND 24h Volume > $500K. Low-liquidity funding arb = trap

**Strategy B capital allocation by volatility regime:**

| BTC ATR Regime | Strategy A (Directional) | Strategy B (Funding Arb) |
|----------------|--------------------------|--------------------------|
| Low vol (ATR < $800 on 1h) | 60% | 40% (more to arb — direction unclear) |
| Normal vol | 70% | 30% |
| High vol (ATR > $2000 on 1h) | 85% | 15% (directional opportunities dominate) |

**Capital allocation vs risk allocation — important distinction:**
- Capital allocation (30% of balance) = margin deployed for Strategy B positions
- Risk allocation (Iron Law #3: 15% total portfolio) = maximum stop-loss loss across ALL strategies
- Strategy B with 30% capital + 8% stops per position = fits within portfolio risk ceiling
- Example: $300 capital → $90 Strategy B pool → 5 positions × $18 each × 8% stop = $7.20 max loss = 2.4% of total capital

**Why 30% baseline:** Research shows trend-following generates big wins episodically, with long periods of whipsaw losses in between. Funding arb produces steady, small positive returns (1-3% per week at extreme rates) that COVER the whipsaw losses from Strategy A. This combination — defensive carry + offensive directional — is structurally superior to 100% directional.

**Interaction with Strategy A:** If a Tier 3 asset has extreme funding AND a directional signal from Strategy A, the directional trade takes priority (it captures both price movement AND funding). The funding arb capital is freed for another asset.

Store Strategy B watchlist separately in strategy.json as `funding_arb_watchlist`.

## Step 5: Generate Strategy

### Conviction Calculation

For each token, compute conviction for BOTH directions using **tier-adaptive weights** (Signal E is now independent Strategy B — excluded from directional scoring):

**How conviction works:** Each signal produces a raw score (0-1 range, see scoring tables above). The raw score is multiplied by the tier-specific weight. Final conviction = sum of weighted scores, clamped to 0-1. Then apply Leaderboard Direction Bias adjustment.

```
# Each signal_x_long / signal_x_short is a raw score in range [0.0, 1.0]
# See each signal's scoring table for how raw scores are computed

# Tier 2 (default — SOL, HYPE, BNB, major alts):
conviction_long  = signal_a_long × 0.40 + signal_b_long × 0.25 + signal_c_long × 0.15 + signal_d_long × 0.20
conviction_short = signal_a_short × 0.40 + signal_b_short × 0.25 + signal_c_short × 0.15 + signal_d_short × 0.20

# Tier 1 (BTC, ETH — macro-driven, SM edge reduced):
conviction_long  = signal_a_long × 0.20 + signal_b_long × 0.15 + signal_c_long × 0.30 + signal_d_long × 0.35
conviction_short = signal_a_short × 0.20 + signal_b_short × 0.15 + signal_c_short × 0.30 + signal_d_short × 0.35

# Tier 3 assets use Strategy B (funding arb) primarily — directional scoring optional

# Example (Tier 2, SOL SHORT):
#   Signal A: 3 SM shorting (0.70) + Fund label (+0.20) = 0.90 raw → 0.90 × 0.40 = 0.360
#   Signal B: 1 verified oracle short (0.50) = 0.50 raw → 0.50 × 0.25 = 0.125
#   Signal C: funding +0.04%/8h (0.60) = 0.60 raw → 0.60 × 0.15 = 0.090
#   Signal D: EMA bearish cross (0.80) = 0.80 raw → 0.80 × 0.20 = 0.160
#   conviction_short = 0.360 + 0.125 + 0.090 + 0.160 = 0.735
```

Then apply Leaderboard Direction Bias adjustment:
```
if leaderboard_bias_strength > 0.6:
  conviction in bias direction += 0.05
  conviction against bias direction -= 0.05
if leaderboard_bias_strength > 0.8:
  conviction in bias direction += 0.10
  conviction against bias direction -= 0.10
```

**Direction selection**: Take the direction with higher conviction. If both exceed their respective thresholds, take the higher one (never hold both directions on the same token).

### Fat Pitch Detection — The Big Win Trigger

Most trading days produce ordinary setups. Fat Pitches are rare (1-3 per month) but account for the majority of annual returns. PTJ spent 95% of his time in small probe positions waiting for these moments, then scaled to maximum conviction.

**A Fat Pitch requires ALL FIVE conditions (no exceptions):**

1. **Leaderboard Consensus:** `leaderboard_bias_strength > 0.8` (8/10+ top traders in same direction)
2. **Fund Participation:** At least one Fund-labeled SM trade in the same direction within 48h
3. **Fresh Trend Confirmation:** Trend JUST confirmed direction change (not mid-trend — you want the START of the move). Indicators: EMA crossover within last 3 days, OR structure break (new HH/HL or LH/LL), OR Weinstein stage transition.
4. **Macro Catalyst:** A specific, identifiable event supporting the direction (not "market feels bearish" — a real event: Fed decision, ETF flow spike, regulatory ruling, major hack/exploit). Must be from BlockBeats or verifiable source.
5. **Funding Alignment:** Current funding rate supports your direction OR is neutral. If funding is strongly against you (>0.03%/8h), the trade is crowded in your direction — NOT a fat pitch.

**Fat Pitch Execution (overrides normal sizing):**
- Skip probe. Enter with confirmed-level sizing immediately: 6-8% capital risk at stop
- If price confirms within first 2× ATR move: scale to maximum 12% total capital risk (Iron Law override — only for Fat Pitch)
- Use trailing stop from entry (1.5× ATR), NO fixed TP — let the winner run
- Total portfolio risk cap during Fat Pitch: 25% (up from 20% normal — one 12% Fat Pitch + Strategy A 8% + Strategy B 5%)

**Fat Pitch Disqualifiers (if ANY is true, it's NOT a fat pitch):**
- The move has already traveled > 5× ATR from the trend change point (you're late)
- You already have a losing position in the same asset (bias contamination)
- Last Fat Pitch trade was a loss (wait one full research cycle before next Fat Pitch)
- Current regime is PAUSED (circuit breaker active)

**Why this works:** The research shows that trend-following's main value is avoiding big losses (defensive). Fat Pitch is the offensive complement — it identifies the rare moments where trend + SM + macro + funding all converge, and concentrates capital there. Druckenmiller's rule: "When you see it, bet big. If you're right, you don't need diversification."

**Fat Pitch Capital Rules (Iron Law override — the ONLY exception):**
- Max 1 concurrent Fat Pitch position (they're rare — 1-3 per month)
- Single Fat Pitch: 6-8% at entry, scales to 12% max (overrides Iron Law #2's 8% ceiling)
- Portfolio risk during Fat Pitch: 25% max (= one 12% Fat Pitch + Strategy A ≤ 8% + Strategy B ≤ 5%)
- If last Fat Pitch was a loss: wait one full research cycle before next Fat Pitch

Store Fat Pitch candidates in strategy.json as `fat_pitch_candidates` with the five conditions documented.

**Entry thresholds (regime-dependent):**

**Standard thresholds (capital > $1,000):**

| Regime | Long Threshold | Short Threshold |
|--------|---------------|-----------------|
| Risk-On | ≥ 0.35 | ≥ 0.45 |
| Ranging | ≥ 0.50 | ≥ 0.60 |
| Risk-Off | ≥ 0.70 | ≥ 0.55 |

**Small capital thresholds (capital ≤ $1,000):**

| Regime | Long Threshold | Short Threshold |
|--------|---------------|-----------------|
| Risk-On | ≥ 0.30 | ≥ 0.40 |
| Ranging | ≥ 0.40 | ≥ 0.50 |
| Risk-Off | ≥ 0.60 | ≥ 0.45 |

**Why lower for small capital:** With $300-1000, the fixed costs of running the agent (compute, API credits, time) demand that we actually trade. A strategy that filters out every signal and sits in cash is a guaranteed loss due to overhead costs. The risk of a bad trade ($5-10 loss) is smaller than the cost of inaction. Lower thresholds mean more trades, more data for evolution, and faster learning.

Short thresholds are higher in Risk-On (shorting in a bull market requires extreme conviction) but lower in Risk-Off (shorting in a bear is the natural direction).

### Cold-Start Adjustment (first week, no oracle data)

When Signal B has no tracked oracles yet:
- Signal A gets 55% weight (up from 40%)
- Signal C gets 25% weight (up from 20%)
- Signal D gets 20% weight (up from 15%)
- Signal B: 0% (no data yet)

This ensures the agent can still trade effectively while building oracle history.

### Position Parameters

For each watchlist entry, calculate:
- **Direction**: LONG or SHORT
- **Conviction score**: 0.00 - 1.00
- **Position size**: % of capital — use the sizing table below
- **Leverage**: from leverage rules (3x default, 2x Risk-Off, up to 4x if conviction ≥ 0.80, max 3x for shorts)
- **ATR stop distance**: ATR(14) × 1.8 (1.5 in Risk-Off). For small capital, use ATR × 2.5 to avoid premature stop-outs on noise.
- **Take-profit tiers**: +15% (close 30%), +30% (close 30%), trailing on rest
- **Max hold days**: 7 (extendable to 10 if profitable)

**Position sizing — STOP-LOSS BASED (not margin-based):**

Size positions by how much you're willing to lose if the stop triggers, NOT by margin percentages.

Formula: `max_size = (capital × risk_percent) / stop_distance_per_unit`

| Entry Type | Max Loss at Stop (% of capital) |
|------------|--------------------------------|
| Probe (initial entry) | 3-4% ($9-$12 on $312) |
| Confirmed (scale after probe validated) | 5-8% ($15-$25 on $312) |
| Never exceed | 8% per trade, 15% total portfolio |

**Probe-then-scale example:** Capital $312, ETH SHORT.
- Probe: entry $2100, stop $2160, distance $60/ETH
- Max loss: $312 × 3% = $9.36
- Probe size: $9.36 / $60 = **0.156 ETH** (~$327 notional)
- If price drops to $2040 confirming thesis + oracle still short → scale to confirmed:
- Scale: additional $15.60 / $60 = 0.26 ETH added
- Total position: ~0.42 ETH (~$882 notional)
- If ETH drops to $1900: profit = 0.42 × $200 = **$84 (+27% of capital)**

Probe-then-scale preserves the verified principle (small initial risk, scale on confirmation) while keeping position sizes meaningful for small capital. The key insight from PTJ isn't the percentage — it's the PROCESS: probe first, never go full size blind.

### Output

Generate `~/.apexhunter/strategy.json` using the strategy template. Include:
- Regime classification with BTC data
- All four signal breakdowns per token
- Watchlist with conviction scores, direction, sizing, stops, take-profits
- Sell watchlist (tokens to watch for exit signals)
- Blacklist (tokens rejected and why)
- Credit usage for this research cycle
- Next research date (3 days out, or sooner if regime changes)

## Credit Budget

| Step | Commands | Credits |
|------|----------|---------|
| Step 0: Regime | tigerpass hl info (free) | 0 |
| Step 1: SM Trades | perp trades (1 call) | 5 |
| Step 2: Oracle Discovery | leaderboard + pnl-leaderboard + positions (×5-10) | 15 |
| Step 3: Market Structure | perp screener + indicators (×3) | 8 |
| Step 4: Trend | tigerpass candles (free) + SM netflow | 5 |
| **Total** | | **~33** |

If credits are low (< 100 remaining):
- Skip oracle discovery (use existing oracles only): saves ~10 credits
- Skip Nansen Indicators: saves ~3 credits
- Minimum viable research: ~15 credits (SM trades + screener + oracle positions)

## Dynamic Research Frequency

- Normal: every 3 days
- High volatility (ATR > 2× its 30-day average): every 2 days
- Market regime change (Risk-On ↔ Risk-Off): immediate research
- Post-major-event (BTC > 5% move in 24h): immediate research
- Low volatility + Ranging: every 5 days (save credits)
