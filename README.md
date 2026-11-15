# Order Flow Strategy for TradingView

**A complete auction-market, order-flow day trading process, turned into one open-source TradingView indicator.**

Most indicators give you a signal. This one runs a process. It reads the environment, waits for price to reach the right location, and only fires when order flow shows that one side of the market is trapped. Then it manages the trade with fixed risk rules.

Open source · MIT license · Pine Script v6 · No repainting · Any market with volume: futures, crypto, stocks

---

## The strategy in one picture

```
 1. ENVIRONMENT            2. LOCATION                 3. CONFIRMATION              4. MANAGEMENT
 ──────────────            ───────────                 ───────────────              ─────────────
 Value up / down /         Discount below VAL          Absorption: sellers          Stop beyond the failure
 balance on 1h & 4h        Premium above VAH           hit hard at the low,         Get back into value
                                                       no result                    or stop to entry
 Gamma regime:             Fibonacci zone              First shift: bullish close   Target: POC or swing
 expansion or chop         0.705 · 0.786 · 0.886       Second failure: sellers      Max 2 trades a day
                           outside value               fail again, higher           Stop after 2 losses
                                                       ENTER on the flip
```

Full explanation: **[docs/STRATEGY.md](docs/STRATEGY.md)**

---

## Install (2 minutes)

1. Open **[order-flow-strategy.pine](order-flow-strategy.pine)** and copy all of it.
2. In TradingView, open the **Pine Editor** at the bottom of the chart.
3. Delete the default code, paste, click **Save**, then **Add to chart**.
4. Use a **5-minute chart** (1 to 15 minutes works). The default window is the first 90 minutes after the New York open. For crypto, you can switch it to 24/7 in the settings.

The indicator works on the free TradingView plan. Only the webhook alerts for AI agents need a paid plan.

---

## Step-by-step: how to trade it

### Before the open: Step 1, Environment

1. **Read the structure** in the panel: *Value up*, *Value down* or *Balance*, measured on the 1-hour and 4-hour charts. Balance means no trades today.
2. **Set the gamma regime.** TradingView has no options data. Open the settings and enter today's regime (Negative or Positive), call wall, put wall and gamma flip from your GEX source. Negative gamma means bigger, faster moves. Positive gamma means failed breakouts and chop. You can skip this step; the indicator works without it.

### Before the open: Step 2, Location

3. **Find the value area.** The glowing purple band is the value area and the gold line is the point of control. Below the band is **discount**, above it is **premium**.
4. **Find the zone.** In a value-up market a glowing green **DISCOUNT ZONE** appears on the last impulse leg (the dotted line shows which leg is measured), but only if it sits outside value and the leg has an internal swing. In a value-down market it is a red **PREMIUM ZONE**. The amber dashed line is 0.886: if price closes beyond it, the zone is gone.

### Live: Step 3, Confirmation

5. **Wait for price to reach the zone.** The panel switches to *IN DISCOUNT*.
6. **Absorption.** A cyan circle with a **Sellers Absorbed** tag marks a candle where sellers were aggressive, the volume traded at the low, and price closed away from the low. The tag shows the delta and the absorption quality (AQ, 0 to 100). On the footprint you see heavy red cells at the bottom of the candle with the gold POC outline down there. The dashboard shows *ABSORPTION*.
7. **First shift.** A candle closes bullish and a small cyan square appears under it. The dashboard shows *LONG ARMED*: waiting for sellers to fail again, higher.
8. **Second failure and entry.** Price pulls back, sellers fail above their first failure, and a candle closes bullish with aggressive buying (cyan imbalance cells light up on the footprint). The entry candle gets a glowing outline, a **2nd Fail ↑** tag marks the second failure, and **▲ LONG · 2nd Failure** appears with the reward to risk, a cyan **Trapped Sellers ⚠** zone, a risk box and a reward box. Hover the label for the absorption level, the second failure and the target.

### Live: Step 4, Management

9. **Value reclaim checkpoint.** Price should get back into the value area. If it does not within 6 candles, the stop moves to entry.
10. **Breakeven.** At +1R the stop moves to entry.
11. **Target.** The point of control or the last swing high, whichever is nearer and at least 1.5R away.
12. **Daily limits.** After 2 trades, or 2 losses in a row, the dashboard shows *DONE FOR TODAY* or *STOPPED FOR TODAY*. Every closed trade gets an outcome tag: **✔ TARGET**, **✕ STOPPED** or **● BREAKEVEN** with the result in R.

Shorts work exactly the same in reverse.

---

## What you see on the chart

Dark navy tags with neon text, layered glow on every zone, and one color language: **cyan = sellers trapped (bullish)**, **magenta = buyers trapped (bearish)**, green and red = buyers and sellers, purple = value, gold = point of control, amber = warnings and invalidation. All colors can be changed in the settings.

| Element | Meaning |
|---|---|
| **Footprint on the candles** | The most recent candles are split into price rows: **sellers on the left, buyers on the right**, shaded by how much volume traded there. Bright cyan or magenta cells are **imbalances** (one side 4x the other, the 400% rule). The **gold outline** is the candle's own point of control. A **Δ tag** under each candle shows its delta. Zoom in to read the numbers |
| **Purple glowing band + gold line** | Value area (VAH to VAL), brightest around the point of control, with VAH, POC and VAL tags on the right |
| **Discount / premium zone** | Glowing green or red zone on the impulse leg with 0.705, 0.786 and ✕ 0.886 tags, a dotted leg anchor line and a tag showing the zone volume (thin, untested or normal) |
| **Absorption circles** | Cyan circle at the low (sellers absorbed) or magenta circle at the high (buyers absorbed), with delta and absorption quality |
| **Small squares** | First shift: the first close against the absorbed side |
| **▲ LONG / ▼ SHORT · 2nd Failure** | Entries, with a glowing outline on the entry candle, a momentum trail from the second failure, the reward to risk and a tooltip explaining the trade |
| **Trapped Sellers / Trapped Buyers ⚠** | The price band where the absorbed side got trapped. Their forced exits fuel the move |
| **Risk and reward boxes** | Entry to stop in red, entry to target in green. A dashed gold line appears when the stop moves to entry |
| **Outcome tags** | ✔ TARGET, ✕ STOPPED or ● BREAKEVEN with the result in R |
| **Target fields** | To the right of price. With a trade open: the target, with the distance and R still to go. When flat: T1 / T2 in the direction of the structure (POC, last swing, call or put wall) |
| **ENTRY / STOP tags** | Live levels of the open trade |
| **Cyan tint** | The active trade window (first 90 minutes of the New York open by default) |
| **Gamma lines** | Call wall, put wall and gamma flip, when you enter them |

### The dashboard

| Section | What it shows |
|---|---|
| Header | Market, timeframe, price, ATR, structure (with gamma regime) |
| **Setup State** | The state of the process in one line (*VALUE UP*, *IN DISCOUNT*, *ABSORPTION*, *LONG ARMED*, *LONG ACTIVE*...) and **Next**: what it is waiting for, or how to manage the open trade |
| **Order Flow · Last Candle** | Buy and sell aggression meters, participation, absorption quality of the active setup, where the volume traded inside the candle, session delta |
| **4-Step Process** | Setup progress (0 to 6 stages) and a ✔ / ◐ / ✕ light for environment, location, confirmation and the risk rules |
| **Structure · MTF** | Value up, value down or balance on four timeframes. ◆ marks the two that drive the filter |
| **Key Levels** | Value area, POC, the zone, the invalidation level and the next target |
| **Active Trade** | Status, entry, stop, target and open P&L in R, with a progress meter to the target |

Full or Compact mode, six positions, three text sizes. Meters, the MTF section and key levels can each be turned off.

---

## Settings

| Setting | Default | What it changes |
|---|---|---|
| Trade window | New York open, first 90 min | Or a custom session, or 24/7 |
| Max trades per day / stop after losses | 2 / 2 | The daily risk rules |
| Structure timeframes | 1h and 4h | The higher timeframes that define value up or down |
| Gamma regime, walls, flip | Not set | Enter from your GEX provider before the open |
| Value area | Developing (current session) | Or the prior session's value area |
| Zone levels | 0.705 / 0.786 / 0.886 | The Fibonacci zone |
| Zone must sit outside value | On | Turn off to allow zones that overlap value |
| Order flow timeframe | Auto | The lower timeframe used to estimate delta |
| Absorption thresholds | 15% delta, POC in outer 35%, close 30% off the extreme | How strict absorption is |
| Volume filter | 1.0x average | Or an absolute number, for example 20,000 for MNQ 5-minute |
| Target | Auto (POC, swing, R) | Or POC only, swing only, or a fixed R |
| Minimum reward | 1.5R | Targets closer than this are replaced by 1.5R |
| Footprint candles / rows | 12 / 6 | How many recent candles get footprint cells, and how many price rows each |
| Colors | Neon palette | Every color of the indicator |
| Layered glow effects | On | Turn off for a flatter look |
| Imbalance ratio | 4 (400%) | When a footprint cell lights up |
| Color event candles | On | Absorption candles in cyan or magenta, entry candles in green or red |
| Color all other candles by delta | Off | Green or red candles shaded by net aggression |
| Dashboard | Full, top right, small | Mode, position, text size, score meters, MTF rows (15m and 1D extra) and key levels |

Every formula is in **[docs/SPECIFICATION.md](docs/SPECIFICATION.md)**.

---

## Use it with an AI agent

The indicator sends a JSON packet with all four steps (environment, location, order flow, trade) on every setup and trade update. Any AI agent can read it and act as your trading assistant.

```
 TradingView               Webhook                 AI agent (Claude Code + Fable 5.2)        You
 Order Flow Strategy  ───► JSON packet       ───►  pre-market plan, reads every setup,  ───► take it,
 closes a candle           per setup/update        manages the trade, grades the day         wait or skip
```

### 1. Create one alert

1. Open the alert dialog (`Alt + A`).
2. **Condition:** `Order Flow Strategy` → `Any alert() function call`.
3. **Notifications:** tick **Webhook URL** and paste your agent's URL.
4. Click **Create**. One alert covers one chart.

Webhook alerts need a paid TradingView plan with two-factor authentication turned on.

### 2. Receive the packets

```bash
git clone https://github.com/roan-maverick/tradingview-order-flow-strategy-indicator
cd tradingview-order-flow-strategy-indicator

python3 examples/webhook_receiver.py --port 8080 --token YOUR_SECRET   # no installs needed
cloudflared tunnel --url http://localhost:8080                          # public HTTPS address
```

Use `https://<your-tunnel>/webhook?token=YOUR_SECRET` as the webhook URL. Packets are saved to `packets/latest.json`, `packets/latest/<SYMBOL>_<TF>.json` and `packets/history.jsonl`.

No-code option: point the webhook at n8n, Make or Zapier, add an AI step with the agent prompt, and send the reply to Telegram or Discord.

### 3. Let the agent run the process

Open Claude Code in the folder and ask:

```
Use agent/AGENT_PROMPT.md as your instructions. Write my pre-market plan,
then read packets/latest.json every time it changes and manage the session.
```

The agent writes the pre-market plan (structure, gamma, value area, scenarios), explains every absorption and entry, manages the open trade, and grades the session A, B or C at the end, separating good losses from bad losses.

Packet fields: **[docs/PACKET_SCHEMA.md](docs/PACKET_SCHEMA.md)** · Agent prompt: **[agent/AGENT_PROMPT.md](agent/AGENT_PROMPT.md)** · Example: **[examples/sample_packet.json](examples/sample_packet.json)**

---

## FAQ

**Does it repaint?**
No. Setups, entries and trade tracking use closed candles only. Higher-timeframe structure uses only closed higher-timeframe bars, and swing points never move once confirmed.

**Is this real order flow?**
It is the closest Pine Script allows. TradingView does not give scripts tick-level bid and ask data, so the footprint, delta and the volume point of control inside each candle are estimated from lower-timeframe candles: volume on up candles counts as buying, volume on down candles as selling, spread across the price rows each one traded through. Imbalances compare buyers and sellers in the same row. It reads the same story as an exchange footprint, at lower resolution.

**I can't see the footprint numbers.**
Zoom in on the last candles. The cells are drawn inside each candle's width, so the numbers appear once the candles are wide enough. The footprint needs a chart timeframe above 1 minute (it reads 1-minute candles inside a 5-minute chart).

**Which markets?**
Any symbol with volume data: index futures (MNQ, NQ, ES), crypto, stocks. Forex feeds without real volume will not produce order flow signals.

**Why so few signals?**
By design. The process takes 0 to 2 trades a day, and most days are a pass. If you want more, widen the trade window or turn off "Zone must sit outside value".

**Where do I get gamma data?**
From any GEX provider for the options market behind your instrument (QQQ and NDX for Nasdaq futures). Enter the regime and levels before the open.

**Does it place trades?**
No. It produces setups, levels and data. You decide.

---

## Repository

```
tradingview-order-flow-strategy-indicator/
├── order-flow-strategy.pine      the indicator (Pine Script v6)
├── docs/STRATEGY.md              the full strategy, step by step
├── docs/SPECIFICATION.md         every calculation and rule
├── docs/PACKET_SCHEMA.md         every field in the JSON packet
├── agent/AGENT_PROMPT.md         prompt for your AI trading assistant
├── examples/webhook_receiver.py  receive packets locally (Python 3, no installs)
├── examples/sample_packet.json   example packet
└── LICENSE                       MIT
```

---

## Disclaimer

For education and research. Not financial advice. The indicator describes the market with fixed rules; it does not predict the future. Win rates and results mentioned in the strategy guide are stated results of the approach, not guarantees. Test on replay or paper first, size small, and never risk money you cannot afford to lose.

## License

MIT
