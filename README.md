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

3. **Find the value area.** The blue band is the value area and the gold line is the point of control. Below the band is **discount**, above it is **premium**.
4. **Find the zone.** In a value-up market a green **DISCOUNT 0.705 – 0.886** box appears on the last impulse leg, but only if it sits outside value and the leg has an internal swing. In a value-down market it is a red **PREMIUM** box. If price closes beyond 0.886, the zone is gone.

### Live: Step 3, Confirmation

5. **Wait for price to reach the zone.** The panel switches to *IN DISCOUNT*.
6. **Absorption.** A cyan **ABS** diamond marks a candle where sellers were aggressive, the volume traded at the low, and price closed away from the low. The panel shows *LONG · ABSORPTION*.
7. **First shift.** A candle closes bullish. The panel shows *LONG · ARMED*: waiting for sellers to fail again.
8. **Second failure and entry.** Price pulls back, sellers fail above their first failure, and a candle closes bullish with aggressive buying. **▲ LONG** appears with a risk box and a reward box. Hover the label to see the absorption level, the second failure and the target.

### Live: Step 4, Management

9. **Value reclaim checkpoint.** Price should get back into the value area. If it does not within 6 candles, the stop moves to entry.
10. **Breakeven.** At +1R the stop moves to entry.
11. **Target.** The point of control or the last swing high, whichever is nearer and at least 1.5R away.
12. **Daily limits.** After 2 trades, or 2 losses in a row, the panel shows *DONE FOR TODAY* or *STOPPED FOR TODAY*.

Shorts work exactly the same in reverse.

---

## What you see on the chart

| Element | Meaning |
|---|---|
| **Strategy panel** | The current state (*VALUE UP*, *IN DISCOUNT*, *LONG · ARMED*, *LONG ACTIVE*...), what it is waiting for next, and a row for each of the four steps |
| **Blue band + gold line** | Value area and point of control, with VAH, POC and VAL tags |
| **Green or red zone box** | Discount or premium zone, 0.705 to 0.886, with a dotted 0.786 line |
| **ABS diamonds** | Absorption: cyan under the candle (sellers absorbed), gold above it (buyers absorbed) |
| **▲ LONG / ▼ SHORT** | Entries, with a tooltip explaining the trade |
| **Risk and reward boxes** | Entry to stop in red, entry to target in green. A dashed gold line appears when the stop moves to entry |
| **ENTRY / STOP / TARGET tags** | Live levels of the open trade |
| **Gamma lines** | Call wall, put wall and gamma flip, when you enter them |

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
It is the closest Pine Script allows. TradingView does not give scripts tick-level bid and ask data, so delta and the volume point of control inside each candle are estimated from lower-timeframe candles. Footprint-style 400% imbalances cannot be measured; the dominance shift uses net delta instead.

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
