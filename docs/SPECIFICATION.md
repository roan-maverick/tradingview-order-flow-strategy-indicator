# Order Flow Strategy: Specification

Version 1.1 · Pine Script v6 · Packet schema `order-flow-strategy/v1`

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
| Location | Session length | Daily (or Weekly, or Cash session 09:30-16:00 America/New_York) |
| Location | Value area % | 70 |
| Location | Profile row height | 0.5 x ATR(14) |
| Location | Developing value: minimum bars | 12 |
| Location | Swing strength | 5 bars each side |
| Location | Impulse leg lookback | 60 bars |
| Location | Zone levels | 0.705, 0.788, 0.886 |
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
| Stops and targets | Target | Swing (trail near POC / walls) |
| Stops and targets | Trail the stop behind aggression / start at | On / 1.5R (or the POC or gamma wall if reached first) |
| Order flow (Pro only) | Footprint row size / flip candle must light up imbalances | Auto / On |
| Stops and targets | Minimum reward / fixed R | 1.5R / 2.0R |
| Stops and targets | Bars to get back into value | 6 |
| Stops and targets | Move stop to entry at | 1.0R |
| AI agent packet | Send JSON packet | Setups and trade updates |
| General | Layered glow / color event candles / color other candles by delta / tint the trade window | On / On / Off / On |
| Colors | Buyers, sellers, cyan, magenta, value area, POC, amber, text, muted | #00FF9C, #FF2D55, #00E5FF, #FF2BD6, #A855F7, #FFC400, #FF9F1C, #C0C6D4, #6B7280 |
| Chart visuals | Value area, zone, leg anchor, absorption footprints and labels, first shift, entries, trapped zones, outcome tags, target map, gamma levels | All on |
| Chart visuals | Keep absorption footprints / keep trades on chart | 10 / 10 |
| Pressure wave | Show / core length / smoothing / amplitude | On / 21 / 8 / 1.2 x ATR |
| Resting liquidity | Zones / sweeps / equal tolerance / zones per side / swept kept | On / On / 0.15 x ATR / 4 / 2 |
| Inefficiencies and impulse | Corridors / min size / max / impulse / min body / max | On / 0.4 x ATR / 5 / On / 1.3 x ATR / 8 |
| Footprint | Show footprint / delta tag under each candle | On / On |
| Footprint | Footprint candles / rows per candle | 12 / 6 (rows are reduced automatically if candles x rows would exceed the drawing limit) |
| Footprint | Imbalance ratio | 4.0 (400%) |
| Dashboard | Mode / position / text size | Full / Top right / Small |
| Dashboard | Score meters, multi-timeframe structure, key levels | On |
| Dashboard | Extra structure rows | 15 and D |

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
| 0.788 | high - 0.788 x range | low + 0.788 x range |
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

### Pro edition: real volume footprint

The Pro file replaces the estimate with `request.footprint(row ticks, 70, imbalance ratio x 100)` (TradingView Premium or Ultimate).

| Value | Source |
|---|---|
| Delta, volume | `footprint.delta()`, `footprint.total_volume()` |
| Volume POC | middle of `footprint.poc()` |
| Imbalances | `volume_row.has_buy_imbalance()` / `has_sell_imbalance()`: a row's buy volume against the sell volume of the row below it (diagonal), at 400% |
| Flip candle | buyers' flip must include at least one buy imbalance (sellers': a sell imbalance) |
| Row size (auto) | NQ / MNQ 4 ticks, ES / MES 1, YM / MYM 5, other futures 2, BTC about $10, ETH about $1, other crypto 10 ticks, everything else 2 |

Bars where TradingView has no footprint data fall back to the estimate.

### Footprint (display)

For each of the last 12 candles (when a lower timeframe is available), the candle range is split into 6 equal price rows, drawn on the candle itself (sellers in the left half, buyers in the right half). Each lower-timeframe candle's volume is spread evenly across the rows between its low and high and counted as buying (up candle), selling (down candle) or half and half (unchanged).

| Element | Rule |
|---|---|
| Left cell | Selling volume in the row, red, opacity scaled to the largest cell of the candle |
| Right cell | Buying volume in the row, green, opacity scaled the same way |
| Imbalance | Standard: one side at least 4x the other in the same row. Pro: TradingView's diagonal imbalance flags. Bright magenta (sellers) or cyan (buyers) |
| POC outline | Gold border on the row with the most total volume |
| Delta label | Buying minus selling volume of the candle |

### Display-only metrics

These values are shown on the chart and the dashboard. They never change a signal.

| Metric | Definition |
|---|---|
| Absorption quality (AQ) | Long: 35 x min(-delta % / 40, 1) + 25 x (35 - POC position) / 35 + 25 x (close position - 30) / 70 + 15 x min(relative volume / 2, 1). Short mirrors it. Each part is clamped to 0..1, total 0 to 100 |
| Buy / sell aggression | Buy = (100 + delta %) / 2, sell = 100 - buy |
| Session delta | Sum of every candle's delta since the start of the day |
| Setup progress | 1 structure, 2 zone ready, 3 price in the zone, 4 absorption, 5 first shift, 6 entry |
| Target map (flat) | In the direction of the structure: the POC, the last swing extreme, the nearest liquidity zone and the call or put wall, nearest first, at least 0.5 ATR apart, up to 3 |
| Flow pressure | Per candle: buy share = (volume + delta) / (2 x volume), or the close position when there is no volume. Weighted by relative volume (0.5 to 2), EMA(8) of buy and sell shares. Flow share = buy / (buy + sell), net = 2 x share - 1 |
| Pressure wave | Core = EMA(hl2, 21). Buyers band = core + EMA(ATR x 1.2 x (0.25 + share), 3), sellers band = core - EMA(ATR x 1.2 x (1.25 - share), 3), inner lines at 55%. Core color changes when net crosses ±0.08 |
| Liquidity zones | Every confirmed swing (strength 5) adds a zone of ±0.12 ATR. A new swing within 0.15 ATR of a zone merges into it (equal highs / lows). Wick beyond and close back = sweep (zone kept, dashed). Close beyond = zone removed |
| Inefficiency | Up: low > high 2 bars ago, the middle candle bullish, gap ≥ 0.4 ATR (down mirrors it). Fill % = how far price has traded back into the corridor |
| Impulse | Body ≥ 1.3 ATR, body ≥ 60% of the range, close in the outer 30% and relative volume ≥ 1.2 |

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
| Swing, trail near POC / walls (default) | The last swing extreme (long leg high / short leg low) if at least 1.5R away, otherwise the POC if at least 1.5R away, otherwise 1.5R. Trailing does the rest |
| Auto | The nearer of the POC and the last swing extreme that is at least 1.5R away. If neither qualifies: 1.5R |
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
| `TRAILING` | trailing started, or the stop moved up behind an aggressive candle |
| `TRAIL_STOP` | the trailing stop was touched beyond entry, trade closed in profit, loss streak reset |
| `TARGET_HIT` | target touched, trade closed |
| `STOPPED` | stop touched before it was moved, trade closed, loss streak + 1 |
| `BREAKEVEN` | stop touched after it was moved to entry, trade closed |

A target hit or a trailing-stop exit resets the loss streak. Moving the stop to entry never loosens a stop that is already trailing beyond entry. The tracker is a reference: it ignores fees, slippage and partial exits.

### Trailing

| Rule | Long | Short |
|---|---|---|
| Trigger | the first of: the POC or the call wall (only if between entry and target), entry + 1.5R, never later than 75% of the way to the target | mirror with the put wall |
| Trail | on every closed candle with a bullish close and delta ≥ +10% (aggression with a result), stop = max(stop, candle low - 0.1 x ATR) | bearish close and delta ≤ -10%, stop = min(stop, candle high + 0.1 x ATR) |

---

## 7. Outputs

| Output | Details |
|---|---|
| JSON packet | `alert()` on bar close. Schema: [PACKET_SCHEMA.md](PACKET_SCHEMA.md) |
| Alert conditions | "OFS: LONG entry", "OFS: SHORT entry", "OFS: Absorption", "OFS: Trade closed" |
| Dashboard | Setup state and next step, order flow meters, 4-step process lights, multi-timeframe structure, key levels, active trade |
| Chart | Order flow pressure wave, liquidity zones and sweeps, inefficiency corridors, impulse candles, footprint and delta tags on recent candles, glowing value area with POC, discount or premium zone with Fibonacci tags and leg anchor, absorption circles, first shift squares, entry outlines and momentum trails, trapped traders zones, risk and reward boxes, outcome tags, target fields, trade window tint, gamma levels |
| Data Window | Structure, delta %, volume POC position, relative volume, long and short setup states, trade side |

---

## 8. Known limitations

- Standard edition: delta and the candle POC are estimates from lower-timeframe candles. The Pro edition reads real footprint data but needs a Premium or Ultimate plan, and TradingView does not have footprint data for every bar or symbol.
- Gamma exposure is not available in TradingView and must be entered manually.
- News is not detected. Avoid setups right before scheduled releases.
- Symbols without volume data cannot produce order flow signals. Set the volume filter to Off and expect no absorption signals.
- Lower-timeframe history is limited by your TradingView plan, so order flow is only available on the most recent part of very long charts.
