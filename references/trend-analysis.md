# Trend Analysis — The Foundation of Every Decision

Trend structure is ground truth. SM data and news are catalysts and validation. This document defines the exact technical analysis process for each timeframe.

## Analysis Order: Top-Down, Always

```
Weekly (30 seconds) → Daily (1 minute) → 4H (1 minute) → 1H (trigger)
```

Higher timeframes ALWAYS override lower ones. A 1H buy signal inside a daily downtrend is a SHORT entry opportunity, not a long.

## Weekly: Regime Classification (Weinstein Stage)

Use BTC weekly candles as the market-wide reference. Individual assets get their own staging.

```bash
# BTC weekly candles (free, HL public API)
curl -s -X POST https://api.hyperliquid.xyz/info \
  -H 'Content-Type: application/json' \
  -d '{"type":"candleSnapshot","req":{"coin":"BTC","interval":"1w","startTime":'$(date -v-6m +%s000)',"endTime":'$(date +%s000)'}}'
```

**Weinstein 4-Stage Classification:**

Compute 30-period SMA on weekly closes. Then classify:

| Stage | Price vs 30w SMA | SMA Direction | Action |
|-------|-----------------|---------------|--------|
| Stage 1 (Base) | Oscillating around flat SMA | Flat | WAIT — no trend |
| Stage 2 (Uptrend) | Above rising SMA | Rising | LONG ONLY |
| Stage 3 (Top) | Oscillating around flat/rolling SMA | Flattening | REDUCE longs, prepare shorts |
| Stage 4 (Downtrend) | Below falling SMA | Falling | SHORT ONLY |

**This single filter eliminates most losing trades.** If the asset is in Stage 4, don't look for longs no matter what SM data says.

## Daily: Structure + EMA Alignment

```bash
# Daily candles for each watchlist asset (free)
curl -s -X POST https://api.hyperliquid.xyz/info \
  -H 'Content-Type: application/json' \
  -d '{"type":"candleSnapshot","req":{"coin":"<TOKEN>","interval":"1d","startTime":'$(date -v-60d +%s000)',"endTime":'$(date +%s000)'}}'
```

**Three checks on daily:**

### 1. EMA Alignment (trend health)

Compute EMA21, EMA50, SMA200 on daily closes.

| Alignment | Meaning | Trading Bias |
|-----------|---------|-------------|
| EMA21 > EMA50 > SMA200 | Strong bull | Aggressive longs |
| EMA21 > EMA50, both < SMA200 | Bull within bear context | Cautious longs, watch for failure |
| EMA21 < EMA50 < SMA200 | Strong bear | Aggressive shorts |
| EMA21 < EMA50, both > SMA200 | Bear within bull context | Cautious shorts, may be just pullback |

### 2. Structure: HH/HL vs LH/LL

Identify the last 3-4 swing highs and swing lows on daily candles.

- **Uptrend confirmed**: Each swing low is higher than the previous swing low AND each swing high is higher than previous
- **Downtrend confirmed**: Each swing high is lower AND each swing low is lower
- **Structure break warning**: Most recent swing low violated in uptrend (or swing high violated in downtrend)

**Pullback vs Reversal distinction:**
- Pullback: price retraces but HOLDS above the last swing low, volume/OI declining during retrace
- Reversal: price breaks below the last swing low, volume/OI expanding on the break

### 3. RSI(14) on daily

- RSI > 50: bull regime confirmed
- RSI < 50: bear regime confirmed
- RSI divergence: price makes new high/low but RSI doesn't — EARLY WARNING of reversal. This is one of the most reliable leading indicators (Chong & Ng 2008).

## 4H: Entry Zone Identification

```bash
# 4H candles (free)
curl -s -X POST https://api.hyperliquid.xyz/info \
  -H 'Content-Type: application/json' \
  -d '{"type":"candleSnapshot","req":{"coin":"<TOKEN>","interval":"4h","startTime":'$(date -v-14d +%s000)',"endTime":'$(date +%s000)'}}'
```

### Keltner Channel (EMA20 +/- 2x ATR)

Compute on 4H candles:
```
middle = EMA(close, 20)
upper = middle + 2 × ATR(14)
lower = middle - 2 × ATR(14)
```

| Price Position | Meaning | Action |
|---------------|---------|--------|
| Above upper band | Strong trend breakout | Don't fade, but don't chase either. Wait for retrace to middle |
| Near middle band | Pullback in trend | **PRIMARY ENTRY ZONE** — this is where you enter with the trend |
| Below lower band | Oversold / panic | Potential reversal if daily structure supports. Spring/upthrust territory |

### Key Levels That Matter

Not all S/R is equal. Prioritize these (in order of importance):

1. **Daily swing highs/lows** visible on the zoomed-out chart (if you can't see it in 10 seconds, it's not structural)
2. **Breakout origin points** — the price level from which a strong directional move started. These retest with high reliability.
3. **Round numbers** ($2000, $80, $35) — heavy limit order and stop clustering in crypto
4. **EMA21/EMA50 on 4H** as dynamic S/R — trend-following entries
5. **Volume profile HVN** (High Volume Nodes) — where price spent the most time = strong S/R

### OI + Funding Context

From Perp Screener data:
- **Price rising + OI rising** = new longs entering, trend healthy
- **Price rising + OI falling** = short covering rally, less sustainable
- **Price falling + OI rising** = new shorts entering, downtrend accelerating
- **Price falling + OI falling** = long liquidation/exit, may be near exhaustion

Funding rate:
- > +0.01%/8h: longs paying shorts, mild short bias
- > +0.03%/8h: crowded long, strong short signal
- < -0.01%/8h: shorts paying longs, mild long bias
- < -0.03%/8h: crowded short, strong long signal

## 1H: Trigger Signals

The 1H timeframe ONLY answers "enter NOW or wait." It never determines direction.

### High-Probability 1H Triggers (in order of reliability):

**1. Spring/Upthrust (Wyckoff) — highest win rate**
- Price breaks a key level (S/R from Layer 2) by 1-3 candles
- Then SNAPS BACK above/below the level
- OI drops during the break (= stops/liquidations were hit)
- This is a stop-hunt / liquidation sweep. The "real" move is the snap-back.
- Enter on the snap-back candle close. Stop just beyond the wick of the spring.

**2. Reversal candle at key level**
- Engulfing candle or pin bar / hammer at EMA21/50 or structural S/R
- Volume/OI spike on the reversal candle
- Enter on close of reversal candle. Stop beyond the candle's wick.

**3. EMA21 bounce in trend**
- Price touches or slightly pierces 1H EMA21, then closes back above/below
- This works ONLY when 4H trend is aligned (EMA21 > EMA50 for longs, < for shorts)
- Enter on close. Stop = 2x ATR(14) on 1H.

**4. RSI divergence + level confluence**
- 1H RSI divergence (price new low, RSI higher low) AT a key 4H level
- This is confluence — two independent signals pointing the same way
- Enter when 1H candle closes above the RSI divergence trigger candle's high

### What is NOT a trigger:
- A single news headline without price confirmation
- SM data without price action at a key level
- "It feels like it should move" — that's not a trigger

## ATR-Based Parameters

For each asset, compute ATR(14) on the timeframe you're trading (1H for most entries):

| Use | Formula |
|-----|---------|
| Stop-loss distance | 2.0 × ATR(14) from entry |
| BE stop trigger | Price must move ≥ 2.0 × ATR from entry first |
| TP1 | 1× stop distance (symmetric) |
| TP2 | 2× stop distance |
| Trailing stop | 1.5 × ATR from peak/trough |
| "Don't chase" filter | If price moved > 3× ATR in last 4 candles, wait for retrace |
| Low-vol regime | ATR(14) < 50% of ATR(14) 20-period average → expect breakout, tighten watchlist |
| High-vol regime | ATR(14) > 150% of average → trend running or about to exhaust |

## Regime Change Detection

The most valuable skill: identifying when the market transitions between stages/regimes.

**Early warnings (appear in order):**
1. **Momentum divergence** — RSI on daily makes lower high while price makes higher high. This is the FIRST signal, often 1-2 weeks before price confirms.
2. **EMA convergence** — EMA21 and EMA50 gap narrowing after a long trend. The trend is losing steam.
3. **Failed new extreme** — price attempts new high/low but fails and closes back within range. Equal High or Lower High in uptrend = distribution starting.
4. **Structure break** — the definitive confirmation. Last swing low broken in uptrend (or swing high in downtrend). After this, the trend has officially changed.
5. **EMA crossover** — the lagging confirmation. By the time EMA21 crosses EMA50, the trend change is established. Use this to confirm, not to trigger.

**For the bot:** Check conditions 1-3 every research cycle. When any appear, flag the asset as "regime change watch" and reduce position sizing. Condition 4 is the confirmation to flip bias.

## Trend Phase Assessment — Where Are We in the Cycle?

Knowing the trend DIRECTION (Stage 4 = bear) is only half the picture. Knowing WHERE you are within that stage determines HOW to trade it.

Every trend has a lifecycle. The same Stage 4 downtrend at -10% from peak behaves very differently from one at -50%. The market's remaining energy, the likelihood of bounces, and the risk of reversal all shift as the trend matures.

**How to assess phase maturity:**

1. **Distance from the moving average** — how far has price deviated from SMA30w?
   - < 10% below: early phase, momentum building, trend likely to extend
   - 10-25% below: mid phase, trend well-established, pullbacks are entry opportunities
   - > 25% below: late phase, trend extended, mean-reversion pressure building, bounces become violent
   - > 40% below: exhaustion zone, shorting here is catching pennies in front of a potential steamroller reversal

2. **RSI context within the trend** — RSI tells you about momentum WITHIN the phase
   - RSI 40-50 in Stage 4: healthy downtrend, plenty of room to fall. Bounces to RSI 50 are short entries.
   - RSI 30-40 in Stage 4: trend is working but getting stretched. Bounces are likely, and they're SHORT ENTRY opportunities (not reversal signals).
   - RSI < 30 in Stage 4: immediate downside is limited. A bounce is almost certain. The question isn't IF but HOW FAR the bounce goes. Wait for the bounce, then re-enter short at resistance.

3. **Volume/OI exhaustion** — is new money still entering the trend?
   - OI expanding during price drops = fresh sellers entering, trend has fuel
   - OI contracting during price drops = existing longs getting liquidated, fuel running low
   - OI contracting + price stabilizing = selling exhaustion, bounce imminent

**The critical thinking question every cycle:** "Am I early enough in this trend to hold through drawdowns, or am I late enough that I should trade the bounces?"

An oracle that entered ETH SHORT at $3,131 can hold through a $200 bounce because they're up $1,000. You entering at $2,050 cannot — a $200 bounce wipes your stop. Same trend, different phase, different strategy.

## Oversold ≠ Buy Signal — It's a Timing Signal

The most common mistake: seeing RSI < 30 and thinking "it's oversold, time to go long." In a Stage 4 downtrend, oversold means "the immediate move down is exhausted, expect a BOUNCE before continuation."

**How to think about oversold conditions in a downtrend:**

1. **Oversold is a TIMING input, not a DIRECTION input.** The direction is still set by the trend (Stage 4 = SHORT). Oversold tells you WHEN to enter, not WHICH WAY.

2. **The bounce is predictable in destination, not in timing.** Oversold conditions (RSI < 30, below Keltner lower band) almost always produce a bounce. The bounce target is usually:
   - 4H EMA21 (weak bounce, trend is very strong)
   - 4H EMA50 / Keltner middle (normal bounce)
   - Daily EMA21 (strong bounce in a weaker downtrend)
   
   These levels are where the bounce runs into selling pressure and the downtrend resumes. That's your SHORT ENTRY ZONE.

3. **Don't short oversold. Don't buy oversold. Wait for the resolution.** The correct action when something is oversold in a downtrend:
   - Step 1: Identify the likely bounce target (EMA21/50, Keltner middle)
   - Step 2: Set alerts or limit orders at that level
   - Step 3: Wait for price to arrive AND show a 1H reversal signal (Layer 3 trigger)
   - Step 4: Enter short with tight stop above the bounce high

4. **The exception: Spring/Upthrust.** If price is oversold AND you see a Wyckoff spring (price spikes below support, triggers liquidations, then snaps back), that IS an immediate entry — but it's a LONG scalp within a SHORT trend, or a signal that the downtrend leg is done and a bounce is starting. Trade it only if the risk is tiny and you exit quickly.

## Trading Style Selection — Hold vs Swing vs Scalp

The framework doesn't prescribe a fixed trading style. The right style emerges from the intersection of:

1. **Trend phase** (where in the cycle)
2. **Capital size** (drawdown tolerance)  
3. **Data edge** (oracle coverage for this asset)

**The thinking process:**

Ask: "How much room is left in this trend, and how much drawdown can I survive?"

- If the answer is "lots of room, I can survive drawdowns" → trend hold (wider stops, larger targets, hold days-weeks)
- If the answer is "trend is mature, bounces are likely, my capital is small" → swing trade (short bounces to resistance, take profit at support, hold hours-days)
- If you have no oracle/SM data on the asset → the only remaining edge is pure technicals, which favors shorter timeframes where price patterns are more reliable

**This is NOT a setting you configure once.** It's a question you answer EVERY research cycle for EVERY asset independently. ETH might warrant a trend hold (3 oracles with massive positions backing you), while BNB might warrant a swing (no oracle data, purely technical). The same asset might switch from swing to trend-hold if you get early entry on a fresh move.

**Capital size and phase interact:**

- $312 capital + Stage 4 early phase = you can trend-hold IF you entered early with the oracles. But you didn't — they entered months ago.
- $312 capital + Stage 4 mid/late phase = swing trading is more appropriate. You can't absorb the drawdowns that trend-holding requires at this stage.
- $10K+ capital + Stage 4 early = trend hold like the oracles. You have the cushion.

**The oracle behavior tells you what style THEY chose.** Your job is to assess whether you have the same conditions they had when they chose it. 0xcac1 can hold HYPE SHORT through -$260K because that's 3.6% of their account. -$260K equivalent for you would be -$11. If your ETH SHORT moves against you $11 ($42 per ETH on 0.26 size), your thesis needs to be very strong to hold.

## Integration with SM Data

Trend analysis is the FRAMEWORK. SM data sits INSIDE this framework:

- **Trend confirms + SM confirms**: HIGH CONVICTION. Enter with full probe size.
- **Trend confirms + SM silent**: MEDIUM. Enter with reduced size.
- **Trend confirms + SM conflicts**: CAUTION. Wait for either SM to flip or trend to break.
- **Trend unclear + SM confirms**: WAIT. SM alone is not enough — oracles can hold underwater positions for weeks (0xcac1 HYPE SHORT -$260K).
- **Trend conflicts + SM confirms**: DO NOT TRADE. If daily trend is bullish but SM is shorting, either SM is early (and will suffer drawdown) or they see something structural you don't. Either way, you lack edge.

The oracle wallets ARE sophisticated trend traders themselves — their positions reflect their own multi-timeframe analysis, just expressed as capital allocation rather than chart markup.
