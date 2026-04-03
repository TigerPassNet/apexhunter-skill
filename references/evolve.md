# /tigertrader evolve — Weekly Strategy Evolution

This document defines the complete flow for `/tigertrader evolve`. The goal: review this week's trades, attribute wins/losses to specific signals and rules, calibrate parameters, and generate a weekly report. Run every Sunday.

## Pre-flight

1. Load `~/.apexhunter/portfolio.json` — need closed trades from this week
2. Load `~/.apexhunter/rule-history.json` — cumulative rule effectiveness
3. Load `~/.apexhunter/oracle-wallets.json` — oracle performance tracking
4. Load `~/.apexhunter/funding-tracker.json` — funding cost analysis

If fewer than 3 closed trades this week, still run evolution but note "insufficient data for statistical significance" in the report.

## Phase 1: Performance Review

### 1.1 Calculate Metrics

From this week's closed trades:

```
total_trades       = count of closed trades
winning_trades     = trades with realized_pnl > 0
losing_trades      = trades with realized_pnl ≤ 0
win_rate           = winning_trades / total_trades
avg_win_pnl        = mean PnL of winning trades (%)
avg_loss_pnl       = mean PnL of losing trades (%)
profit_factor      = sum(winning_pnl) / abs(sum(losing_pnl))
expectancy         = (win_rate × avg_win_pnl) - ((1 - win_rate) × abs(avg_loss_pnl))
max_drawdown       = largest peak-to-trough decline during the week
sharpe_ratio       = mean(daily_returns) / std(daily_returns) × sqrt(365)
sortino_ratio      = mean(daily_returns) / downside_std(daily_returns) × sqrt(365)
```

### 1.2 Direction Breakdown (Perps-specific)

Separate all metrics by direction:

| Metric | Long Trades | Short Trades |
|--------|-------------|--------------|
| Count | | |
| Win Rate | | |
| Avg Win % | | |
| Avg Loss % | | |
| Profit Factor | | |
| Avg Hold Duration | | |
| Avg Leverage Used | | |

**Key question**: Is there a direction bias? If short win rate > long win rate by >15%, the agent may be underweighting short signals — needs calibration.

### 1.3 Regime Breakdown

Group trades by the regime at entry:

| Metric | Risk-On | Ranging | Risk-Off |
|--------|---------|---------|----------|
| Trades | | | |
| Win Rate | | | |
| Avg PnL | | | |

**Key question**: Are we performing worse in one regime? If Risk-Off trades consistently lose, tighten the threshold or skip trading in Risk-Off entirely.

### 1.4 Leverage Analysis (Perps-specific)

| Leverage | Trades | Win Rate | Avg PnL |
|----------|--------|----------|---------|
| 2x | | | |
| 3x | | | |
| 4x | | | |
| 5x | | | |

**Key question**: Is higher leverage correlated with better or worse outcomes? If 4x+ trades have lower win rates, reduce the max conviction-based leverage.

### 1.5 Funding Cost Attribution (Perps-specific)

```
total_funding_paid     = sum of funding costs across all trades
total_funding_received = sum of funding income across all trades
net_funding            = received - paid
funding_as_pct_of_pnl  = net_funding / total_realized_pnl × 100
```

If funding costs > 20% of gross profits, this is a warning:
- Consider shortening max hold period
- Weight funding rate more heavily in Signal C
- Prefer positions where funding is income (long when funding is negative, short when positive)

## Phase 2: Attribution

For each closed trade, identify which signals contributed to the entry decision.

### 2.1 Signal Attribution

From portfolio.json, each trade stores `conviction_breakdown`:
```json
{
  "signal_a": 0.28,
  "signal_b": 0.15,
  "signal_c": 0.08,
  "signal_d": 0.10,
  "direction": "LONG",
  "conviction": 0.72
}
```

For each signal, compute:
```
signal_win_rate    = wins where this signal > 0.10 / total where this signal > 0.10
signal_avg_pnl     = mean PnL of trades where this signal > 0.10
signal_expected_value = signal_win_rate × signal_avg_pnl - (1-signal_win_rate) × abs(signal_avg_loss)
```

### 2.2 Sub-Signal Attribution

Drill deeper into each signal's components:

**Signal A sub-components:**
- SM consensus (≥3 traders same direction): win_rate, avg_pnl
- Fund label involvement: win_rate, avg_pnl
- High value trades (> $50K each): win_rate, avg_pnl
- SM reversal confirmation (closing opposite): win_rate, avg_pnl

**Signal B sub-components:**
- Elite oracle signal: win_rate, avg_pnl
- Verified oracle: win_rate, avg_pnl
- New position detection: win_rate, avg_pnl
- Multi-oracle consensus: win_rate, avg_pnl

**Signal C sub-components:**
- Funding rate extreme (contrarian): win_rate, avg_pnl
- OI surge + SM alignment: win_rate, avg_pnl
- Nansen momentum indicator: win_rate, avg_pnl

**Signal D sub-components:**
- EMA crossover alignment: win_rate, avg_pnl
- Dip buy / rally short bonus: win_rate, avg_pnl
- SM netflow direction: win_rate, avg_pnl

### 2.3 Exit Strategy Attribution

For each exit type, compute:

| Exit Type | Count | Avg PnL at Exit | Better Alternative? |
|-----------|-------|-----------------|---------------------|
| ATR stop-loss | | | Would wider stop have recovered? |
| Trailing stop | | | Did we miss >15% more upside? |
| Take-profit T1 | | | Did price continue >10% after? |
| Take-profit T2 | | | Did remaining 40% capture enough? |
| Time stop | | | Would 2 more days have helped? |
| Funding cost stop | | | Was position recovering? |
| Liquidation protection | | | Should leverage have been lower? |

**Missed opportunity analysis**:
- For each stop-loss exit: track price 48h after exit. If price recovered to above entry → "missed recovery"
- For each TP exit: track price 48h after exit. If price continued > 15% → "premature exit"
- For each time stop: track price 48h after exit. If position would have become profitable → "patience would have helped"

### 2.4 Exponential Decay Weighting

All historical data in rule-history.json uses exponential decay:

```
weight = 0.5 ^ (weeks_ago / 4)
```

4-week half-life: data from 4 weeks ago counts 50%, 8 weeks counts 25%, etc.

Update rule-history.json with this week's new data, applying decay to all existing entries.

## Phase 3: Calibration

Based on attribution results, adjust parameters. All changes bounded to prevent over-fitting.

### 3.1 Signal Weight Adjustment

For each signal (A, B, C, D):
- If signal_expected_value > 10%: increase weight by 0.02
- If signal_expected_value > 20%: increase weight by 0.03
- If signal_expected_value < -5%: decrease weight by 0.02
- If signal_expected_value < -10%: decrease weight by 0.03

**Bounds:**
- Maximum weight change per week: ±0.03
- Minimum signal weight: 0.05
- Maximum signal weight: 0.55
- All four weights must sum to 1.00 (normalize after adjustment)

### 3.2 Sub-Signal Weight Adjustment

Within each signal, adjust component weights using the same logic:
- Components with positive expected value → increase by 0.02
- Components with negative expected value → decrease by 0.02
- Bounds: each component [0.03, 0.30], ±0.03 max change per week

### 3.3 Exit Parameter Calibration

**ATR multiplier:**
- Current default: 1.8
- If >40% of stop-loss exits were "missed recoveries" → widen to +0.1 (max 2.5)
- If <20% of stop-loss exits were "missed recoveries" → good calibration, keep
- If stop-losses are rarely triggered AND losses are large → tighten by -0.1 (min 1.2)

**Trailing stop percentage:**
- Current default: 6%
- If >40% of trailing stops miss >15% more upside → widen by 0.5% (max 10%)
- If trailing stops consistently capture the move → keep or tighten by 0.5% (min 3%)

**Take-profit tiers:**
- If Tier 1 exits consistently see >20% more upside → raise Tier 1 to +18% or +20%
- If Tier 1 exits are good (price often reverses after) → keep at +15%
- Bounds: Tier 1 [12%, 25%], Tier 2 [25%, 40%]

**Time stop:**
- If time stop exits are consistently unprofitable → shorten by 1 day (min 4 days)
- If time stop exits are often followed by favorable moves → extend by 1 day (max 10 days)

**Funding cost stop:**
- If funding stops save money (price moved against position after exit) → keep at 2%
- If funding stops miss recovery → raise threshold to 2.5% or 3% (max 4%)

### 3.4 Position Sizing (Kelly-Inspired)

After week 2, adjust position sizes based on actual performance:

```
edge = (win_rate × avg_win%) - (loss_rate × avg_loss%)
kelly_fraction = edge / avg_win_percent
suggested_size = kelly_fraction × 0.5  # half-Kelly for safety
```

Compute separately for:
- Long trades
- Short trades
- Each conviction tier

**Bounds:**
- Minimum position size: 2%
- Maximum position size: 10% (hard ceiling)
- Maximum change per week: ±1%

### 3.5 Leverage Calibration

If leverage analysis shows:
- Higher leverage trades have worse win rates → reduce max leverage by 1x
- Higher leverage trades have better risk-adjusted returns → keep (but never exceed 5x)
- Shorts with > 3x leverage consistently lose → reduce short max leverage

### 3.6 Direction Bias Correction

If long win rate - short win rate > 15%:
- Raise short conviction threshold by 0.05
- Lower long conviction threshold by 0.05

If short win rate - long win rate > 15%:
- Raise long conviction threshold by 0.05
- Lower short conviction threshold by 0.05

Goal: both directions should contribute positively. If one direction is consistently negative, restrict it rather than eliminate it.

## Phase 4: Mutation (every 3 weeks)

Controlled experiments to discover new edges. Only runs if system has ≥3 weeks of data.

### 4.1 Hypothesis Generation

Based on Phase 2 attribution, generate one testable hypothesis. Examples:
- "Fund label trades hit 85% win rate — what if Fund signals alone are enough to enter?"
- "Funding rate reversal signals have 75% accuracy — increase Signal C funding weight to 0.20"
- "3x leverage on high conviction trades outperforms 4x — cap at 3x for all"
- "Short trades in Risk-On have negative expectancy — skip shorts when Risk-On"

### 4.2 Mutation Execution

- Cap mutation trades at 20% of total trades (1-2 positions per week)
- Tag mutation trades in portfolio.json: `is_mutation: true, mutation_id: "fund_solo_entry"`
- Apply different parameters only to mutation trades
- If mutation trades lose > 10% cumulative within 2 weeks → abort mutation

### 4.3 Mutation Evaluation (after 3 weeks)

Compare mutation trades to baseline trades:
- Win rate comparison
- Avg PnL comparison
- Risk-adjusted return comparison

**Decision:**
- Mutation significantly outperforms baseline → adopt mutation parameters
- Mutation slightly outperforms → extend observation by 2 weeks
- Mutation underperforms → reject and revert
- Mutation critically fails (> 15% loss) → abort early

### 4.4 Mutation History

Store in rule-history.json:
```json
{
  "mutations": [
    {
      "id": "fund_solo_entry",
      "hypothesis": "Fund label alone is sufficient for entry",
      "started": "2026-04-07",
      "status": "active|adopted|rejected|aborted",
      "trades": 4,
      "pnl": "+8.3%",
      "comparison_to_baseline": "+3.1%"
    }
  ]
}
```

## Phase 5: Self-Diagnosis (monthly — every 4th evolution)

Behavioral audit to catch systematic problems.

### 5.1 Overtrading Check
- If > 10 trades/week on average → raise entry thresholds by 0.05
- If < 2 trades/week on average → lower entry thresholds by 0.03 (but never below minimums)

### 5.2 Direction Bias
- Long/short trade ratio over 30 days
- If > 80% in one direction → force-flag: "direction bias detected, seek opposite signals"

### 5.3 Regime Blindness
- Compare win rates across regimes
- If similar loss rates in Risk-On and Risk-Off → regime classification isn't working, review BTC EMA parameters

### 5.4 Signal Decay
- Compare signal win rates: weeks 1-2 vs weeks 3-4
- If win rate trending down > 10% → consider resetting to default weights

### 5.5 Leverage Creep
- Average leverage over time
- If trending upward → force recalibration to defaults

### 5.6 Concentration Risk
- Were > 3 trades in the same token this month?
- Were > 50% of trades in top-2 tokens?
- If yes → diversification warning

### 5.7 Oracle Health
- Are oracle wallets still profitable?
- Any oracles demoted this month?
- Oracle accuracy rate: track oracle signal → trade outcome correlation

## Phase 6: Generate Evolution Report

Create `~/.apexhunter/reports/week-YYYY-WW.md`:

```markdown
# TigerTrader Evolution Report — Week YYYY-WW

## Market Context
- BTC: $X → $Y (Z%)
- Regime: Risk-On/Ranging/Risk-Off
- Major events: [if any]

## Performance Summary
- Starting capital: $X
- Ending capital: $Y
- Week return: Z%
- Cumulative return: Z%

| Direction | Trades | Win Rate | Avg PnL | Profit Factor |
|-----------|--------|----------|---------|---------------|
| Long      |        |          |         |               |
| Short     |        |          |         |               |
| Total     |        |          |         |               |

## What Worked
- [Signal/rule with positive attribution and why]

## What Didn't
- [Signal/rule with negative attribution and why]

## Funding Analysis
- Total funding paid: $X
- Total funding received: $Y
- Net: $Z (% of gross profits)

## Leverage Analysis
| Leverage | Trades | Win Rate | Avg PnL |
|----------|--------|----------|---------|
| 2x       |        |          |         |
| 3x       |        |          |         |
| 4x+      |        |          |         |

## Exit Analysis
| Exit Type | Count | Avg PnL | Missed Recovery? |
|-----------|-------|---------|------------------|
| ATR Stop  |       |         |                  |
| Trail Stop|       |         |                  |
| TP Tier 1 |       |         |                  |
| TP Tier 2 |       |         |                  |
| Time Stop |       |         |                  |
| Funding   |       |         |                  |

## Strategy Adjustments
- Signal weights: A=X.XX, B=X.XX, C=X.XX, D=X.XX (changes: ...)
- ATR multiplier: X.X (was X.X)
- Trailing stop: X% (was X%)
- [Other parameter changes]

## Mutation Status
- [Active/completed mutation details]

## Self-Diagnosis (if applicable)
- [Monthly audit findings]

## Oracle Wallet Health
- [Oracle status changes, accuracy rates]

## Next Week Plan
- Regime outlook
- Key tokens to watch
- Parameter changes taking effect
```

## Phase 7: Update State Files

1. Update `rule-history.json` with all new attribution data (with decay)
2. Update `oracle-wallets.json` with status changes
3. Update `strategy.json` with new signal weights (if changed)
4. Update `funding-tracker.json` (reset closed positions)

## Credit Budget

| Phase | Commands | Credits |
|-------|----------|---------|
| Phase 1: Review | portfolio.json only | 0 |
| Phase 2: Attribution | portfolio.json only | 0 |
| Phase 3: Calibration | computation only | 0 |
| Phase 4: Mutation | SM comparison (optional) | ~5 |
| Phase 5: Self-Diagnosis | computation only | 0 |
| Phase 6: Report | generation only | 0 |
| Oracle validation | positions check (×5-10) | ~10 |
| **Total** | | **~15-20** |
