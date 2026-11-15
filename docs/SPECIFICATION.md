# Order Flow Strategy: Specification

Version 1.0 · Pine Script v6 · Packet schema `order-flow-strategy/v1`

Every calculation the indicator performs, so a trader or an AI agent can verify each value.

---

## 1. Design principles

| Principle | How it is enforced |
|---|---|
| No repainting | Setups, entries and trade tracking are evaluated only on closed bars (`barstate.isconfirmed`). |
| No lookahead | Higher-timeframe structure is requested as `expr[1]` with `lookahead = barmerge.lookahead_on`, so only closed higher-timeframe bars are used. Swing points are confirmed pivots and never move. |
| One process | Every signal passes environment, location, confirmation and the risk rules, in that order. |
| Honest order flow | TradingView does not expose tick-level bid/ask data to scripts, so delta is estimated from lower-timeframe candles. This is stated wherever order flow is shown. |

---

## 2. Inputs and defaults

| Group | Input | Default |
|---|---|---|
| Trading window | Trade window | New York open, first 90 minutes (09:30–11:00 America/New_York) |
| Trading window | Custom session and timezone | 0930-1100, America/New_York |
| Trading window | Max trades per day | 2 |
| Trading window | Stop after consecutive losses | 2 |
| Environment | Structure timeframes | 60 and 240 |
| Environment | Structure swing strength | 3 |
| Environment | Gamma regime, call wall, put wall, gamma flip | Not set, 0, 0, 0 |
| Environment | Skip setups in positive gamma | Off |
| Location | Value area | Developing (current session) |
| Location | Session length | Daily |
| Location | Value area % | 70 |
| Location | Profile row height | 0.5 x ATR(14) |
| Location | Developing value: minimum bars | 12 |
| Location | Swing strength | 5 bars each side |
| Location | Impulse leg lookback | 60 bars |
| Location | Zone levels | 0.705, 0.786, 0.886 |
| Location | Zone must sit outside value / require internal swing | On / On |
| Location | Thin volume threshold | 30% of POC volume |
| Order flow | Order flow timeframe | Auto |
| Order flow | Absorption: net aggression / POC in outer % / close away from extreme | 15% / 35% / 30% |
| Order flow | Dominance shift: net aggression | 10% |
| Order flow | Max bars from absorption to entry | 8 |
| Participation | Volume filter | Relative, 1.0 x the 50-bar average |
| Participation | Absolute minimum | 20,000 |
| Stops and targets | Stop beyond | Second failure |
| Stops and targets | Stop buffer | 0.1 x ATR(14) |
| Stops and targets | Target | Auto (POC, swing, R) |
| Stops and targets | Minimum reward / fixed R | 1.5R / 2.0R |
| Stops and targets | Bars to get back into value | 6 |
| Stops and targets | Move stop to entry at | 1.0R |
| AI agent packet | Send JSON packet | Setups and trade updates |

---

## 3. Environment

### Value structure

For each structure timeframe the indicator keeps the last two confirmed swing highs and lows (`ta.pivothigh` / `ta.pivotlow`, strength 3):

| Code | Condition |
|---|---|
| `1` (up) | last swing high > previous swing high AND last swing low > previous swing low |
| `-1` (down) | last swing high < previous AND last swing low < previous |
| `0` | anything else |

If a structure timeframe is not higher than the chart, the chart's own structure is used.

| Combined structure | Rule |
|---|---|
| `VALUE_UP` | timeframe 1 is up AND timeframe 2 is not down |
| `VALUE_DOWN` | timeframe 1 is down AND timeframe 2 is not up |
| `BALANCE` | anything else. No setups |

### Gamma

Entered manually. If "Skip setups in positive gamma" is on and the regime is Positive, no new setups are taken. Walls and flip are drawn when greater than 0.

### Trading window

New setups only while the bar is inside the window. "Always (24/7)" removes the window. Daily counters (trades, loss streak) reset at the start of each new day.

---

## 4. Location

### Volume profile and value area

The profile is built from every candle of the current session (daily or weekly):

1. Row height = max(tick size, ATR(14) x 0.5), fixed at the start of the session. Row `k` covers `[k x height, (k + 1) x height)`.
2. Each candle's volume is spread evenly across the rows between its low and high.
3. **POC** = the row with the most volume (its midpoint).
4. **Value area** = start at the POC row and repeatedly add the neighbouring row (above or below) with more volume until 70% of the session's volume is covered. VAH is the top of the highest row, VAL the bottom of the lowest row.

| Value area mode | Values used |
|---|---|
| Developing (default) | The current session's profile, once it has at least 12 bars |
| Prior session | The profile of the last completed session |

Position: `DISCOUNT` if close < VAL, `PREMIUM` if close > VAH, otherwise `INSIDE_VALUE`.

### Impulse legs

| Leg | Defined when | Origin | Extreme |
|---|---|---|---|
| Long leg | A swing high is confirmed that is higher than the current leg high (or no valid leg exists) | Lowest low of the 60 bars up to the swing high | The swing high, then the highest high since |
| Short leg | A swing low is confirmed that is lower than the current leg low (or no valid leg exists) | Highest high of the 60 bars up to the swing low | The swing low, then the lowest low since |

**Internal structure:** a confirmed swing low between the long leg's origin and its high, above the origin (mirror for shorts).

### Zones

| Level | Long (discount) | Short (premium) |
|---|---|---|
| 0.705 | high - 0.705 x range | low + 0.705 x range |
| 0.786 | high - 0.786 x range | low + 0.786 x range |
| 0.886 | high - 0.886 x range | low + 0.886 x range |

| Rule | Long | Short |
|---|---|---|
| Zone ready | leg valid, internal swing present, 0.705 level at or below VAL | leg valid, internal swing present, 0.705 level at or above VAH |
| In zone | low <= 0.705 level AND close >= 0.886 level | high >= 0.705 level AND close <= 0.886 level |
| Invalidation | close below the 0.886 level: the leg is discarded until a new one forms | close above the 0.886 level |

### Zone volume

The zone is compared with the active profile: `UNTESTED` if no volume traded at those prices, `THIN` if the average row volume across the zone is below 30% of the POC row, otherwise `NORMAL`.

---

## 5. Confirmation

### Order flow estimate

For every chart candle, the indicator loads its lower-timeframe candles with `request.security_lower_tf` (Auto: 1m for charts up to 15m, 5m up to 1h, 15m up to 4h, 60m above):

| Value | Definition |
|---|---|
| Delta | sum of the volume of up candles minus the volume of down candles |
| Delta % | delta / total volume x 100 |
| Volume POC | midpoint (hl2) of the lower-timeframe candle with the most volume |
| POC position | (volume POC - low) / (high - low) x 100: 0 = at the low, 100 = at the high |
| Close position | (close - low) / (high - low) x 100 |

On 1-minute charts and below there is no lower timeframe, so the candle itself is used (delta = its volume signed by direction, POC = hl2).

### Absorption

| | Rule |
|---|---|
| Sellers absorbed (for longs) | delta % <= -15 AND POC position <= 35 AND close position >= 30 |
| Buyers absorbed (for shorts) | delta % >= 15 AND POC position >= 65 AND close position <= 70 |

### Long setup state machine

Evaluated on closed bars only.

| State | Moves to | When |
|---|---|---|
| `IDLE` | `ABSORPTION` | all rules pass, price is in the discount zone, and sellers are absorbed. If that same candle closes bullish, it moves straight to `FIRST_SHIFT` |
| `ABSORPTION` | `ABSORPTION` (re-anchored) | a lower candle shows absorption again in the zone |
| `ABSORPTION` | `FIRST_SHIFT` | a candle closes bullish |
| `ABSORPTION` | `IDLE` | a close below the absorption low |
| `FIRST_SHIFT` | `ENTRY` | a candle closes bullish with delta % >= +10, the lowest low since the shift stayed above the absorption low (sellers failed higher), and price pulled back below the shift candle's close first |
| `FIRST_SHIFT` | `IDLE` | price trades at or below the absorption low |
| any | `IDLE` | more than 8 bars since absorption, close below 0.886, window closes, or structure is no longer value up |

Short setups mirror every rule.

### Rules checked before any setup

Inside the window, fewer than the max trades today, loss streak below the limit, no open trade, participation passes, and the gamma filter passes.

### Participation

| Mode | Passes when |
|---|---|
| Relative | volume >= 1.0 x SMA(volume, 50) |
| Absolute | volume >= 20,000 |
| Off | always |

---

## 6. Trade management

| Field | Long | Short |
|---|---|---|
| Entry | close of the entry candle | close of the entry candle |
| Stop | second-failure low (or absorption low) - 0.1 x ATR | second-failure high (or absorption high) + 0.1 x ATR |
| Risk (R) | entry - stop | stop - entry |
| Reclaim level | VAL at entry | VAH at entry |

### Targets

| Mode | Target |
|---|---|
| Auto (default) | The nearer of the POC and the last swing extreme (long leg high / short leg low) that is at least 1.5R away. If neither qualifies: 1.5R |
| Value area POC | The POC if at least 1.5R away, otherwise 1.5R |
| Last swing | The swing extreme if at least 1.5R away, otherwise 1.5R |
| Fixed R | 2.0R |

### Status updates on every closed bar

The stop is checked first (conservative when a bar touches both).

| Status | When |
|---|---|
| `ACTIVE` | trade opened |
| `VALUE_RECLAIMED` | a close back inside value (above VAL for longs, below VAH for shorts) |
| `STOP_AT_ENTRY` | price reached +1R, stop moved to entry |
| `NO_RECLAIM_STOP_AT_ENTRY` | value not reclaimed within 6 bars, stop moved to entry |
| `TARGET_HIT` | target touched, trade closed |
| `STOPPED` | stop touched before it was moved, trade closed, loss streak + 1 |
| `BREAKEVEN` | stop touched after it was moved to entry, trade closed |

A target hit resets the loss streak. The tracker is a reference: it ignores fees, slippage, partial exits and trailing beyond breakeven.

---

## 7. Outputs

| Output | Details |
|---|---|
| JSON packet | `alert()` on bar close. Schema: [PACKET_SCHEMA.md](PACKET_SCHEMA.md) |
| Alert conditions | "OFS: LONG entry", "OFS: SHORT entry", "OFS: Absorption" |
| Strategy panel | Current state and what it is waiting for, then each of the four steps |
| Chart | Value area band with POC, discount or premium zone, ABS markers, LONG / SHORT labels with a tooltip, risk and reward boxes, gamma levels |
| Data Window | Structure, delta %, volume POC position, relative volume, long and short setup states, trade side |

---

## 8. Known limitations

- Delta and the candle POC are estimates from lower-timeframe candles, not true bid/ask footprint data. 400% bid/ask imbalances cannot be measured in Pine Script, so the dominance shift uses net delta instead.
- Gamma exposure is not available in TradingView and must be entered manually.
- News is not detected. Avoid setups right before scheduled releases.
- Symbols without volume data cannot produce order flow signals. Set the volume filter to Off and expect no absorption signals.
- Lower-timeframe history is limited by your TradingView plan, so order flow is only available on the most recent part of very long charts.
