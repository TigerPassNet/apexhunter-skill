# /apexhunter execute & /apexhunter monitor — Trade Execution

This document defines the complete flow for both `/apexhunter execute` (full signal evaluation + trade execution, ~10 credits) and `/apexhunter monitor` (price-only check + stop/TP enforcement + news scan + thesis check, 0 Nansen credits).

**Before ANY action, load `~/.apexhunter/layer1-anchor.json`** and read `allowed_bias`. This is your compass. Every decision this cycle must be consistent with the anchor unless you detect a regime change trigger.

## Mandatory Decision Protocol (MDP)

Every trading decision — open, close, add, reduce, hold-through-volatility — must pass through this protocol. Monitor and execute differ only in data depth (free vs paid), not in decision rigor. Skipping any step produces a blind spot that costs money.

### MDP Step 1: Collect All Three Data Pillars

You cannot make a decision without all three. A decision based on two out of three is a guess.

| Pillar | Monitor (free) | Execute (paid) |
|--------|---------------|----------------|
| **News** | BlockBeats newsflash API (free, mandatory) | Same |
| **SM Intelligence** | Last known oracle positions from portfolio.json + strategy.json | Fresh Nansen SM trades (~5 credits) + oracle position refresh (~5 credits) |
| **Technicals** | HL candles (free) + current prices (free) | Same + Perp Screener funding/OI data |

**News is never optional.** The BlockBeats newsflash API is free and takes one curl call. If the API key is not set, tell the user to set it — do not proceed with trading decisions without news context. The v1 Drift hack at 02:25 caused SOL to dump and ETH to pump — we saw the price action but didn't know WHY because we weren't reading news. Trading blind is how you get killed.

```bash
# MANDATORY every cycle — both monitor and execute
curl -s -H "api-key: $BLOCKBEATS_API_KEY" \
  "https://api-pro.theblockbeats.info/v1/newsflash" \
  -G --data-urlencode "size=10" --data-urlencode "lang=en"
```

If `BLOCKBEATS_API_KEY` is not set, check with the user. If truly unavailable, you may proceed but MUST note "NEWS BLIND" in all decision outputs as a risk flag.

### MDP Step 2: Agent Debate

After collecting all three pillars, spawn two parallel subagents with opposite mandates for every actionable decision (entry, exit, size change, hold-vs-close).

```
Bull/Long/Hold Agent (model: sonnet):
  "Given this data, argue aggressively for [LONG/HOLD/ADD]. 
   Data: [prices, news, SM activity, trend, position details, trade history].
   Be SPECIFIC. No hedging. Max 150 words."

Bear/Short/Close Agent (model: sonnet):
  "Given this data, argue aggressively for [SHORT/CLOSE/REDUCE].
   Data: [identical data as Bull agent].
   Be SPECIFIC. No hedging. Max 150 words."
```

Both agents run in parallel. Each gets identical raw data including:
- Current prices and technicals (EMAs, ATR)
- News headlines from this cycle
- SM trade activity / oracle positions
- Current portfolio state and trade history
- Any relevant lessons from memory (past mistakes)

### MDP Step 3: Synthesize and Decide

Read both arguments. Decide using these filters in order:

1. **Data vs Emotion** — which argument cites specific data points vs vague feelings?
2. **Layer 1 Alignment** — which argument is consistent with the regime anchor?
3. **Regret Minimization** — if wrong, which action would you regret MORE?
4. **Default to Less** — when arguments are 50/50, do less (don't over-manage)

### When to run MDP

| Situation | Run full MDP? |
|-----------|--------------|
| Opening a new position | YES — full 3-pillar + debate |
| Closing a position early (before stop/TP) | YES — full 3-pillar + debate |
| Stop-loss triggered mechanically | NO — execute immediately, debrief after |
| TP1 hit | NO — execute the pre-planned partial close |
| Major price move (>3% in 30 min) | YES — escalate to full data pull + debate |
| Routine monitor with no signals | NO — just log "no action, thesis intact" |
| Holding through volatility (position losing >3%) | YES — debate hold vs close |
| **Fat Pitch candidate detected** | YES — full 3-pillar + debate, but with Fat Pitch sizing rules |

The point: mechanical rules (stops, TPs) execute without debate. Judgment calls (new entries, early exits, sizing changes) require the full protocol. This prevents both impulsive trades AND analysis paralysis.

### Fat Pitch Execution Mode

When strategy.json contains `fat_pitch_candidates`, and a candidate's entry trigger fires during execute/monitor:

1. **Verify all 5 conditions are STILL true** (they were true at research time, but markets move):
   - Re-check leaderboard bias (from cached data if monitor, fresh if execute)
   - Confirm catalyst hasn't reversed (check BlockBeats)
   - Confirm funding hasn't flipped against you
   - Confirm price hasn't already moved > 5× ATR (too late)

2. **Run MDP with Fat Pitch framing:**
   - Bull/Bear debate gets the Fat Pitch context: "This is flagged as a Fat Pitch — all 5 conditions met"
   - The debate's job is to find the HIDDEN RISK the conditions don't capture
   - If the Bear agent cannot identify a specific, data-backed risk → proceed with Fat Pitch sizing
   - If the Bear agent identifies a concrete risk → downgrade to normal entry

3. **Execute with Fat Pitch sizing:**
   - Enter at confirmed-level immediately: 6-8% capital risk at stop (skip probe)
   - Set trailing stop at 1.5× ATR from entry (no fixed TP)
   - Log with entry_type: "fat_pitch" and document all 5 conditions

4. **Fat Pitch position management (different from normal):**
   - After 2× ATR move in your favor: scale to 12% capital risk (add position)
   - After 4× ATR move: tighten trailing to 1.0× ATR (protect the big win)
   - NO time stop — hold until trailing stop or thesis death
   - Thesis check every monitor cycle: if ANY of the 5 original conditions reverses → tighten to 0.75× ATR trailing

### MDP Output Format

Every decision must be logged with this structure:

```
DECISION: [ACTION] [TOKEN] [DIRECTION]
├── News Pillar: [summary of relevant news, or "no relevant news"]
├── SM Pillar: [SM activity summary]
├── Technical Pillar: [price, EMA, ATR, funding]
├── Bull Case: [agent's argument, 1-2 sentences]
├── Bear Case: [agent's argument, 1-2 sentences]
├── Synthesis: [why you chose this action]
├── Layer 1 Alignment: [yes/no + explanation]
└── R:R Ratio: [X:1]
```

---

## Monitor Loop (every 30 minutes, 0 credits)

The monitor is the most critical safety mechanism. It runs every 30 minutes and uses ONLY free HL API data (plus free BlockBeats news).

### Step M0: Collect Three Data Pillars (MDP Step 1)

Before any position checks, gather all three pillars. This is the foundation — without it, every subsequent step is guessing.

**Pillar 1: News** (mandatory, free)
```bash
curl -s -H "api-key: $BLOCKBEATS_API_KEY" \
  "https://api-pro.theblockbeats.info/v1/newsflash" \
  -G --data-urlencode "size=10" --data-urlencode "lang=en"
```
If API key missing, warn user and flag "NEWS BLIND" on all decisions.

**Pillar 2: Technicals** (free)
```bash
tigerpass hl info --type mids        # all asset prices
tigerpass hl info --type positions   # current positions
```
Plus 1h candles for held assets + watchlist (free HL API).

**Pillar 3: SM Intelligence**
- Monitor: use last known data from strategy.json + oracle-wallets.json (0 credits)
- Execute: pull fresh SM trades + oracle positions (~10 credits)

Run news and price fetches in parallel — they're independent.

### Step M1: Parse News Headlines

Read the newsflash results from M0. Assess each headline:
1. Does this change the macro regime?
2. Does this impact any held position or watchlist token?
3. Does this create a new opportunity?

If HIGH IMPACT news is found that threatens an open position, flag it — the thesis check in M7.5 will decide whether to close using full MDP (debate included).

### Step M2: Get Current Positions

```bash
tigerpass hl info --type positions
```

Returns: each position's entry_price, mark_price, size, leverage, liquidation_price, unrealized_pnl.

### Step M3: Liquidation Protection (HIGHEST PRIORITY)

For each open position, compute liquidation distance:

```
liq_distance = |mark_price - liquidation_price| / mark_price × 100
```

- **liq_distance < 10%** → CLOSE 100% IMMEDIATELY
  ```bash
  tigerpass hl order --coin <TOKEN> --side <opposite> --price <market_approx> --size <full_size> --reduce-only --order-type ioc
  ```
- **liq_distance < 20%** → REDUCE 50%
  ```bash
  tigerpass hl order --coin <TOKEN> --side <opposite> --price <market_approx> --size <half_size> --reduce-only --order-type ioc
  ```

Log the action to portfolio.json with exit_reason: "liquidation_protection".

### Step M4: Stop-Loss Check

For each open position, load the stop-loss price from portfolio.json.

**Long position**: if `mark_price ≤ stop_price` → CLOSE
**Short position**: if `mark_price ≥ stop_price` → CLOSE

Execute with IOC order (immediate execution):
```bash
tigerpass hl order --coin <TOKEN> --side <close_side> --price <market_approx> --size <full_remaining_size> --reduce-only --order-type ioc
```

Log with exit_reason: "atr_stop_loss".

**Small capital stop-loss adjustment (≤ $1K):**
Use ATR × 2.5 instead of ATR × 1.8 for stop distance. Rationale: with small positions, a premature stop-out costs more in fees and spread than absorbing extra noise. The tighter stop generates more losing trades due to normal volatility. Wider stops mean fewer false exits, and the absolute dollar risk per trade is still small (< 3% of capital enforced by Iron Law #6).

### Step M5: Take-Profit Check

For each open position, check against the three take-profit tiers:

**Small Capital TP Tiers (≤ $1,000) — tighter, faster profit capture:**

TP targets must be SYMMETRIC with stop distance. If stop is 2.5% away, first TP should be within reach — not 5x further.

**Tier 1: price moves 1× stop distance in your favor**
- Example: stop is $2.45 away → TP1 triggers at $2.45 profit per unit
- Close 30% of position
- This ensures you capture SOMETHING on most winning trades
- Do NOT move stop yet — let the remaining 70% breathe
- Log with exit_reason: "take_profit_tier_1"

**Tier 2: price moves 2× stop distance in your favor**
- Close another 30% (40% remains)
- NOW move stop to breakeven on remaining position
- Activate trailing stop
- Log with exit_reason: "take_profit_tier_2"

**Tier 3: Trailing stop on remaining 40%**
- Trail at 1.5× ATR from trough (short) or peak (long)
- Update trough/peak each cycle
- If triggered: close remaining position
- Log with exit_reason: "trailing_stop"

**Why this is better:** The old system had TP1 at +15% leveraged (5.4% price move) but stop at 2.9%. This meant most trades got stopped before reaching TP1. The new system makes TP1 reachable — same distance as stop. SOL Trade #1 would have hit TP1 at $82.50 (we reached $82.64) and locked in $2.44 profit on 30% of position.

**Standard Capital TP (> $1,000) — keep original tiers:**
- Tier 1: +15% leveraged, Tier 2: +30%, Tier 3: trailing 6%

**Profit calculation:**
- Long: `(mark_price - entry_price) / entry_price × leverage × 100`
- Short: `(entry_price - mark_price) / entry_price × leverage × 100`

### Step M5.5: Winner Management (Druckenmiller + Minervini Rules)

**Lock in gains — but don't strangle the trade:**
- Move stop to breakeven ONLY when price has moved ≥ 2× ATR from entry (not based on capital %)
- This ensures the stop is outside normal noise range. A 0.6% move on a 2.8% ATR asset is noise — don't lock BE on noise.
- Once BE is justified (2× ATR move confirmed), then:
  - uPnL ≥ +3% of capital → move stop to +1% profit (lock minimum gain)
- **Learned from SOL Trade #1:** BE stop at +0.6% move got triggered by normal volatility. The thesis was right (Oracle still winning $727K) but tight BE killed the trade.

**Scale into winners (Druckenmiller concentration):**
- Position profitable AND thesis strengthening (SM adding same direction, trend confirmed) → consider adding 50% more size
- Only scale in if R:R from CURRENT price to stop/target still ≥ 3:1
- Max one scale-in per position

This prevents the Day 1 disaster: BTC was at +$0.44 (+0.13% of capital), then slid to -$1.7 because the stop was never moved. The gain evaporated because winning wasn't actively managed.

### Step M6: Time Stop Check

For each position, compute hold duration:
```
hold_days = (now - entry_time) / 86400
```

| Condition | Action |
|-----------|--------|
| hold_days ≥ 7 AND profitable | Close (default max hold) |
| hold_days ≥ 7 AND losing AND regime favorable | Extend to 10 days (one extension) |
| hold_days ≥ 7 AND losing AND regime unfavorable | Close immediately |
| hold_days ≥ 5 AND losing > -5% | Close early (cut losses) |
| hold_days ≥ 10 | Close regardless (extended deadline reached) |

Log with exit_reason: "time_stop".

### Step M6.5: Strategy B — Funding Arb Position Management

For each Strategy B (funding arb) position in portfolio.json:

1. **Check current funding rate** from Perp Screener data or cached strategy.json:
   - If funding has normalized (dropped below threshold: 0.02%/8h for shorts, -0.01%/8h for longs) → close position, the edge is gone
   - If funding has INCREASED → hold, the carry is getting better
   
2. **Check for new funding arb opportunities** from strategy.json `funding_arb_watchlist`:
   - If Strategy B capital is available AND a watchlist asset has extreme funding → open position
   - No MDP debate needed — funding arb is mechanical (math-based, not judgment-based)
   - Just verify liquidity check passes: OI > $100K, Volume > $500K

3. **Track cumulative funding earned** per position in funding-tracker.json:
   - This is Strategy B's P&L — it should be steadily positive
   - If a funding arb position's price loss exceeds 3× its cumulative funding earned → close (the carry isn't covering the adverse move)

### Step M7: Funding Cost Check (Strategy A positions only)

Load `~/.apexhunter/funding-tracker.json`. For each Strategy A (directional) position:

```bash
# Get current funding info from position data
# funding_usd field in position response shows cumulative funding
```

If cumulative funding PAID (not received) > 2% of position margin:
- Close the position — funding is eating the profit potential
- Log with exit_reason: "funding_cost_stop"

If cumulative funding RECEIVED > 1% of position margin:
- This is a favorable funding position — extend max hold by 2 days
- Update funding-tracker.json

### Step M7.5: Breaking News Scan (MANDATORY — part of MDP Step 1)

News was already fetched in MDP Step 1 at the start of this cycle. This step processes it for position-specific impact.

Each newsflash has an `id` (integer) and `create_time`. To avoid re-processing:
- Load `~/.apexhunter/funding-tracker.json` field `last_newsflash_id`
- Only process newsflashes with `id` > `last_newsflash_id`
- After processing, update `last_newsflash_id` to the highest id seen

**Read each new headline and assess its trading relevance yourself.** You are an LLM — don't reduce yourself to keyword matching. Read the news, understand the context, and judge:

1. **Does this news change the macro regime?** (geopolitics, monetary policy, regulation)
2. **Does this news directly impact any token I hold or am watching?** (hack, delist, partnership, legal action)
3. **Does this news create a new trading opportunity?** (sector rotation, narrative shift)

**If you judge any news as HIGH IMPACT for your positions:**
1. Assess direction: risk-on or risk-off? which tokens affected?
2. If counter-positioned: run MDP debate (hold vs close), then act on synthesis
3. If aligned: consider adding (if within risk limits) — run MDP debate for sizing
4. If new opportunity: flag for next execute cycle (don't open positions in monitor unless emergency)
5. Log the news event and your reasoning in portfolio.json

**If no news is trading-relevant: move on.** Most 30-minute cycles will have nothing actionable — that's fine. The value is catching the 1-in-50 cycle where something matters.

### Thesis Invalidation Rule (critical — learned from Day 1)

Every position exists because of a thesis. Every monitor cycle, ask: **"Is my entry thesis still alive?"**

- If the thesis is dead (catalyst reversed, narrative broken, counter-evidence overwhelming): **CLOSE IMMEDIATELY. Do not wait for stop-loss.** A stop-loss is for when you're wrong about price direction. Thesis death means you're wrong about the REASON — the stop-loss is irrelevant.
- If counter-evidence is ambiguous: tighten stop to breakeven or small loss.
- If thesis is intact but price is against you: hold. This is what stops are for.

**Examples:**
- Entered BTC LONG on "Iran peace" → Iran attacks Kuwait, US deploys troops → thesis DEAD → close now
- Entered ETH SHORT on "SM consensus" → frankfrankbank.eth flips long → thesis WEAKENED → tighten stop
- Entered HYPE SHORT on "oracle consensus + trend" → HYPE drops 3% → thesis CONFIRMED → hold

**Think like the big traders:** When the BRENTOIL whale flipped $2.7M long to $3.2M short in hours, he didn't wait for a stop-loss on his long. He recognized the thesis changed and acted. That speed is the difference between -$1 and -$10. Adapt or die.

This step costs 0 Nansen credits (BlockBeats API is separate). It fills the gap identified on Day 1 when the Iran peace news required human intervention.

### Step M8: Circuit Breaker Check

Compute daily and weekly PnL from portfolio.json:

- **Daily loss > 5%**: Set `freeze_new_entries = true`. Only process exits.
- **Weekly loss > 8%**: Set `full_freeze = true`. Only process stop-losses.

### Step M9: Update Portfolio

After processing all exits:
- Update portfolio.json with closed trades, new capital state
- Update funding-tracker.json with latest funding data
- Update price history for all held tokens
- Compute and store current portfolio metrics

### Monitor Output

```
TigerTrader Monitor | <timestamp> | Regime: <Risk-On/Ranging/Risk-Off>

Exits:
  [TOKEN] SHORT closed @ $X → +Y% (trailing_stop)
  [TOKEN] LONG reduced 50% @ $X (liquidation_protection, liq distance was 18%)

Open Positions:
  BTC  LONG  3x  entry $95,000  current $97,200  +6.9%  stop $91,400 (4.0% away)  liq $76,000
  ETH  SHORT 2x  entry $3,800   current $3,650   +7.9%  stop $4,050  (5.2% away)  tp1 hit ✓

Portfolio: $5,240 (+4.8%) | Margin used: 22% | Next execute: 2h 15m
```

---

## Execute Flow (every 4 hours, ~10 credits)

Execute does everything Monitor does, PLUS evaluates new entry signals.

### Step E0: Run Monitor

First, run the full Monitor loop (Steps M1-M9). Process all exits before considering entries.

### Step E1: Pre-flight Checks

1. Load `~/.apexhunter/strategy.json` — check it hasn't expired
2. If strategy is > 5 days old, warn and suggest running `/tigertrader research` first
3. Check `freeze_new_entries` and `full_freeze` flags — if either is set, skip all entry logic
4. Check remaining capital: `tigerpass hl info --type balances`
5. Count open positions — check against max_positions limit (default: 6)

### Step E2: Regime Override Check

```bash
# HL public API for BTC candles (free)
curl -s -X POST https://api.hyperliquid.xyz/info \
  -H 'Content-Type: application/json' \
  -d '{"type":"candleSnapshot","req":{"coin":"BTC","interval":"1h","startTime":'$(date -v-1d +%s000)',"endTime":'$(date +%s000)'}}'
```

Compute BTC's 24h price change:
- If BTC dropped > 8% since strategy generation → override regime to Risk-Off
- If BTC dropped > 12% in 7 days → set FULL PAUSE, skip all entries
- Log the override

### Step E3: Scan Latest SM Activity (~5 credits)

```bash
# Get latest SM perp trades since last execute
nansen research smart-money perp-trades \
  --filters '{"include_smart_money_labels":["Smart HL Perps Trader","Fund","Smart Trader"],"value_usd":{"min":5000}}' \
  --sort block_timestamp:desc \
  --limit 50 \
  --fields trader_address,token_symbol,side,action,value_usd,price_usd
```

Update Signal A scores for watchlist tokens based on latest activity.

### Step E4: Check Oracle Positions (~5 credits)

For each tracked oracle wallet (up to top 5):

```bash
nansen research profiler perp-positions --address <oracle_addr> \
  --fields token_symbol,side,position_value_usd,leverage,entry_price,upnl_usd
```

Compare to last known positions:
- New position? → Boost Signal B score for that token+direction
- Increased size? → Boost Signal B
- Closed position? → If we hold same position, add to sell watchlist
- Reduced? → Warning signal

### Step E4.5: Thesis Check on Existing Positions (BEFORE new entries)

For every open position, answer ONE question: **"Would I open this trade RIGHT NOW with today's data?"**

If NO → close it. Don't wait. The sunk cost of the position is irrelevant.

Specific checks:
- Has the catalyst/narrative that triggered entry reversed? (news, geopolitical shift)
- Has SM consensus flipped? (majority of SM now trading opposite direction)
- Has a major whale visibly reversed on the same thesis? (e.g., oil trader flipping $3M from long to short)
- Has the oracle direction bias shifted?

This step runs BEFORE evaluating new entries because freeing capital from dead positions enables better new trades. Holding a position whose thesis is dead costs both the unrealized loss AND the opportunity cost of that margin.

### Step E5: Update Conviction Scores

Recalculate conviction for each watchlist token using:
- Updated Signal A from Step E3
- Updated Signal B from Step E4
- Signal C and D from strategy.json (only refreshed during research)

```
# Use tier-adaptive weights from research.md (Tier 2 default shown):
conviction_long  = signal_a_long  × 0.40 + signal_b_long  × 0.25 + signal_c_long  × 0.15 + signal_d_long  × 0.20
conviction_short = signal_a_short × 0.40 + signal_b_short × 0.25 + signal_c_short × 0.15 + signal_d_short × 0.20
# For Tier 1 (BTC/ETH): A=0.20, B=0.15, C=0.30, D=0.35
# Signal E is Strategy B (independent) — not included in directional conviction
```

### Step E6: Evaluate Entry Signals

For each watchlist token, evaluate in order of conviction (highest first):

**Gate 1: Conviction threshold**
- Compare conviction to regime-dependent threshold (see design spec)
- If below threshold → SKIP

**Gate 2: Not already held**
- If already have a position in this token → SKIP (no doubling)
- Exception: if held in opposite direction, this is a reversal signal → close existing first, then evaluate entry

**Gate 3: Price action filter**
- Get 24h price change from candles or `(mark_price - previous_price) / previous_price`
- **For longs**:
  - 24h change > +10% → SKIP (don't chase pump)
  - 24h change > +15% even with high conviction → SKIP
  - 24h change between -5% and -15% with conviction ≥ 0.70 → IDEAL dip buy, +0.05 bonus
  - 24h change < -20% → SKIP (potential crash)
- **For shorts**:
  - 24h change < -10% → SKIP (don't chase dump)
  - 24h change > +5% to +15% with conviction ≥ 0.70 → IDEAL short entry on relief rally
  - 24h change > +20% → SKIP (potential breakout, not a short)

**Gate 4: Capacity check**
- Would this trade exceed max positions? → SKIP
- Would this trade exceed max exposure (30% total, 20% same-direction)? → SKIP
- Would the position size + existing margin > 30% of capital? → Reduce size to fit

**Gate 5: Minimum trade size AND minimum impact**
- Position notional (margin × leverage) must be > $10 (HL minimum)
- **Small capital rule:** Position margin must be ≥ 5% of total capital. A $1 margin position on $300 capital wastes an execute cycle. If the signal isn't worth 5%+ margin, skip it — don't open trivial positions that can't cover operating costs.

**Gate 6: Cost-benefit check (small capital only, ≤ $1K)**
- Estimate the expected PnL of this trade: `notional × expected_move% × win_probability`
- If expected PnL < $0.50, skip — it won't cover the credit cost of one execute cycle
- This gate prevents "correct but meaningless" trades

### Step E6.5: Risk-Reward Filter (PTJ 5:1 Rule)

For each trade that passes conviction gates, calculate the R:R ratio BEFORE placing the order:

```
reward = distance from entry to TP1 (conservative target)
risk = distance from entry to stop loss
R:R = reward / risk
```

- R:R ≥ 5:1 → IDEAL. Enter probe at 3-4% risk. Scale to 8% on confirmation.
- R:R 3:1 to 5:1 → ACCEPTABLE. Enter probe at 3% risk. Scale to 5% on confirmation.
- R:R < 3:1 → REJECT. The math doesn't work — even high win rates can't overcome bad R:R.

**Log the R:R ratio for every trade.** This is the single most important number in trading. PTJ can be wrong 80% of the time and still profit because of 5:1.

### Step E6.7: Agent Debate (MDP Step 2 — mandatory before any entry)

For each token that passed Gates 1-6 AND the R:R filter, run the full MDP debate before placing orders.

Spawn two parallel subagents per token:

```
Bull Agent (model: sonnet):
  "Argue for opening [TOKEN] [DIRECTION]. Data:
   - Price: $X, EMA20: $Y, EMA50: $Z, ATR: $W
   - News this cycle: [headlines]
   - SM activity: [summary from E3]
   - Oracle positions: [summary from E4]
   - Conviction score: X.XX
   - Trade history: [relevant past trades on this token, lessons learned]
   Max 150 words. Be specific — cite data, not feelings."

Bear Agent (model: sonnet):
  "Argue AGAINST opening [TOKEN] [DIRECTION]. Data: [identical].
   Max 150 words. Be specific — cite data, not feelings."
```

After both return, synthesize per MDP Step 3 rules. Log the full debate output. If the synthesis says DON'T ENTER → skip this token even though it passed all gates. Gates are necessary but not sufficient — the debate is the final quality filter.

### Step E7: Execute Entries

For each token that passes all gates AND the MDP debate:

1. **Calculate entry price**: Use current mid price with a small offset
   - Long: mid_price × 0.999 (slightly below mid for better fill)
   - Short: mid_price × 1.001 (slightly above mid)
   - Use GTC order type (persistent until filled or cancelled)

2. **Place order**:
   ```bash
   # Long entry example
   tigerpass hl order --coin BTC --side buy --price 94905 --size 0.05

   # Short entry example
   tigerpass hl order --coin ETH --side sell --price 3810 --size 1.5
   ```

3. **Set leverage** (if different from current): HL remembers leverage per-asset, adjust if needed via the API

4. **Record in portfolio.json**:
   - entry_price, entry_time, direction, size, leverage
   - conviction_score, conviction_breakdown (A/B/C/D scores)
   - regime_at_entry
   - stop_loss_price (entry ∓ ATR × 1.8)
   - tp1_price (+15%), tp2_price (+30%)
   - max_hold_days: 7
   - tp1_hit: false, tp2_hit: false
   - trailing_stop_active: false
   - peak_price: entry_price
   - funding_cumulative: 0

### Step E8: Sell Watchlist Processing

Check if any currently held tokens appear on the sell watchlist:

Sell watchlist triggers:
- SM opening large opposite-direction positions (> $100K)
- Oracle wallets closed their position in same token
- Funding rate flipped against position direction AND is extreme (> 0.05%/8h)

If triggered:
- High severity (multiple triggers): close position immediately
- Low severity (single trigger): tighten stop-loss by 30% and set flag for next monitor to watch closely

### Step E9: Scale-In Check (for existing positions)

For positions opened in the last 24h with conviction ≥ 0.70:
- If currently profitable > 2%: consider adding to position
- Maximum scale-in: 50% of original size
- Only one scale-in per position
- New average entry updates stop-loss accordingly

### Execute Output

```
TigerTrader Execute | <timestamp> | Regime: Risk-On | Credits used: 10

SM Activity (last 4h):
  3 SM opened BTC LONG (total $450K) — Signal A: 0.28
  2 SM added ETH SHORT ($180K) — Signal A: 0.18
  1 Fund opened SOL LONG ($500K) — Signal A: 0.22 (+0.10 fund bonus = 0.32)

Oracle Updates:
  oracle_0x7a3f: NEW BTC LONG position $120K, 3x leverage
  oracle_0x91b2: increased ETH SHORT by 40%

Entries:
  ✅ BTC LONG | conviction 0.72 | 5% margin, 3x leverage | entry $95,100 | stop $91,400 | tp1 $109,365
     Signals: A=0.28 B=0.15 C=0.08 D=0.10 | dip buy (-3% 24h, +0.05 bonus)

  ❌ SOL LONG | conviction 0.42 | below threshold (0.50 for Ranging) — skipped
  ❌ DOGE SHORT | conviction 0.38 | below threshold (0.60 for Ranging) — skipped

Exits (from monitor):
  ETH LONG closed @ $3,850 → +12.3% | exit_reason: take_profit_tier_1 (30% closed)

Portfolio: $5,480 (+9.6%) | 3 open positions | Margin used: 18% | Next execute: 4h
Next research: 1d 12h
```

## Entry Decision Output Format

For every watchlist token, show the decision process transparently:

```
Token: BTC | Direction: LONG
├── Signal A (SM Trades):  0.28/0.40 — 3 SM opened long, $450K total, no Fund
├── Signal B (Oracles):    0.15/0.25 — verified_oracle 0x7a3f has BTC long
├── Signal C (Structure):  0.08/0.20 — funding +0.02% (neutral), OI +5% (mild)
├── Signal D (Trend):      0.10/0.15 — 20EMA > 50EMA, price above 20EMA
├── Raw Conviction:        0.67 (weighted sum)
├── Dip Buy Bonus:         +0.05 (24h change: -3.2%, conviction ≥ 0.70 after bonus)
├── Final Conviction:      0.72
├── Threshold (Ranging):   0.50
├── Gate 1 (Conviction):   ✅ PASS (0.72 ≥ 0.50)
├── Gate 2 (Not held):     ✅ PASS
├── Gate 3 (Price action): ✅ PASS (-3.2% 24h, within bounds)
├── Gate 4 (Capacity):     ✅ PASS (margin 18% + 5% = 23% < 30%)
├── Gate 5 (Min size):     ✅ PASS ($4,750 notional > $10)
└── DECISION: ✅ ENTER LONG | 5% margin | 3x leverage | stop $91,400 | tp1 $109,365
```
