# TradingView publication notes

## Title

Liquidity Sweep, MSS and FVG Entry Model with Look-ahead A/B

## Description (paste as is, everything between the two rules)

---
WHAT IT DOES
This strategy turns one ICT-style entry sequence into objective, testable rules: a liquidity pool is swept, price shifts market structure with a displacement candle, leaves a fair value gap (FVG), and a limit order waits inside that gap. It is built for MNQ and MES futures on 1 to 15 minute charts.

WHAT IS ORIGINAL HERE
- Every ICT term is written as a precise condition. Where a term has more than one defensible definition, the choice is an input, not a silent default.
- Each setup lives in an explicit state machine (Idle, Swept, MSS, Armed) with its own expiry per phase, so a stale setup cannot leak into a trade. Each pool starts at most one setup.
- A built-in look-ahead A/B switch reproduces the classic request.security() leak on purpose, so anyone can measure its effect in the Strategy Tester instead of trusting a claim.

THE RULES AS IMPLEMENTED
- Pool: previous day high/low (last closed daily bar) and/or confirmed swing highs/lows (ta.pivothigh, ta.pivotlow). The nearest level on each side of price is the active pool. Optional equal highs/lows filter.
- Sweep: price trades through the pool and, by default, closes back on the origin side.
- MSS: within N bars of the sweep, a candle with body of at least ATR x multiplier closes through the opposing confirmed swing, away from the swept pool.
- FVG: the three-bar imbalance (low[2] > high for shorts, high[2] < low for longs), known at the close of the third bar. Thin gaps are ignored; the gap must sit inside the leg off the sweep.
- Entry: a resting limit order at the proximal, distal or midpoint gap edge, cancelled if unfilled after N bars.
- Stop: beyond the sweep extreme plus an ATR buffer.
- Target: the opposing liquidity pool, or an R multiple when no pool qualifies.
- Kill zones: London and New York windows in an explicit timezone.
- Risk guards: no new setups after the day's loss reaches a limit, and open positions are flattened in a fixed time window.

CHOICES EXPOSED AS INPUTS
Sweep confirmation, FVG entry edge, target mode, pool types, pivots, equal-high/low filter, displacement, stop buffer, kill zones, loss limit and flatten window.

HOW TO CHECK THE NON-REPAINTING CLAIM
Signals are evaluated on bar close, daily pools use high[1] and low[1] with lookahead on (the last closed daily bar), and swing pools are used only after the pivot is confirmed.
1. Add the script to MNQ1! or MES1! on 5 minutes with a fixed date range.
2. With the demo input OFF, record net profit, profit factor, max drawdown and trade count. The table reads "Honest build".
3. Turn the demo input ON and change nothing else. The table reads "LOOK-AHEAD DEMO": on historical bars, every intraday bar already knows its day's final high and low.
4. Compare. The gap is the look-ahead error, not an edge.
The demo must stay OFF for any real evaluation. It isolates one repainting vector: a method, not a certificate.

STRATEGY PROPERTIES (defaults, published as they are)
- Initial capital: 100,000. At 5% margin one NQ contract at 25,000 needs about 25,000 of collateral (point value 20 x price x 5%), so 100,000 holds one NQ or ES contract. MNQ and MES need about a tenth of the NQ/ES margin, so there the larger capital mostly changes the percentage figures.
- Order size: 1 contract, fixed.
- Margin: 5% long and short, in line with futures margin as a share of notional.
- Commission: 2.25 per contract per side, a plausible all-in cost for micro futures.
- Slippage: 1 tick.
- Orders are processed on bar close; the script does not calculate on every tick.

ALERTS
Entries and exits send a JSON alert_message; dedupe and bracket placement belong on the receiving server.

LIMITATIONS
- A prototype of an entry model, not a tuned system. Every threshold is a documented starting point; no parameter was selected on the tested data.
- Confirmed pivots lag by the right-hand pivot length, so structure is recognized late in fast moves. This is a deliberate trade-off against repainting.
- Backtest results do not guarantee future results. Not financial advice. Futures trading carries substantial risk of loss.

The chart shows MNQ1! on the 5-minute timeframe. The Strategy Tester report uses Deep Backtesting over the entire available history (first trade on 2019-05-06, last on 2026-10-21) with 1,157 closed trades and the default properties above. The default chart range alone covers a few months and far fewer trades.

---

## Publishing checklist

1. Open `ict-sweep-mss-fvg.pine` in the Pine Editor and fix any compile error. Compilation has not been checked locally.
2. Add the script to a clean MNQ1! 5m chart (standard candles, no other scripts, symbol, timeframe and script name visible).
3. Strategy Tester with the default inputs and default properties:
   - More than 100 closed trades. If fewer, widen the date range with Deep Backtesting before changing anything else.
   - No caution warning in the report.
   - Properties tab equals the STRATEGY PROPERTIES section above (100,000 capital, 1 contract, 5% margin, 2.25 per contract commission, 1 tick slippage, process orders on close).
4. Confirm the look-ahead demo input is OFF (the table reads "Honest build").
5. Fill the placeholder line at the end of the description with the actual date range and trade count.
6. Publish as Public and Open-source. Categories: Chart patterns or Support and resistance, plus Futures if available.
7. Check the description for links, contact info and promotion: there must be none, and no performance figures.

## Update 1 (2026-10-25): release notes (paste as is, everything between the two rules)

Flow (TradingView docs, "Publishing scripts"): switch back to the clean MNQ1! 5m chart of checklist item 2 (no other scripts, demo OFF, the updated script on the chart) before opening Publish…, because the page captures whatever chart is open > Pine Editor > Publish… > Update existing script > pick the publication in the first field > leave the chart checkbox unchecked so the page captures the current chart > release notes > Continue > Publish new version. Release notes are final the moment they are published. Refresh the last paragraph of the description (date range and closed-trade count) in the same dialog.

---
- Initial capital raised from 25,000 to 100,000. At 5% margin one contract needs point value x price x 5% of collateral: about 2,500 on MNQ at 25,000, but about 25,000 on NQ, the whole previous account. On NQ1! 15m (2025-11-30 to 2026-10-23) the previous version opened one trade, with about 6 of room above the margin, and closed it with a margin call. 100,000 holds one NQ or ES contract. MNQ and MES need about a tenth of the NQ/ES margin, so there the larger capital mostly changes the percentage figures: net profit % and drawdown % are not comparable with the previous version.
- New HUD row: margin per contract at the 5% of the defaults, current equity and the room between them, with a warning when that room is under one ATR (the strategy's ATR input).
- The HUD "Fill" counter now reads strategy.closedtrades + strategy.opentrades, so it counts filled trades. The previous counter compared the position size between bar closes and missed any trade that entered on a bar's close and exited inside the next bar, or opened and closed inside one bar, so the HUD could show Fill 0 with trades in the list.
- No change to pools, sweep, MSS, FVG, entry, stop, target, kill zones, guards or the look-ahead A/B.
---

Outcome (2026-10-25, about 15:15 Buenos Aires): published as update 1 from the «ICT publish» layout on MNQ1! 5m with Deep Backtesting «Entire history» (1,157 closed trades, 2019-05-06 to 2026-10-21). Checked on screen before publishing: NQ1! 5m, 33 trades in the tester and Fill 33 in the HUD (the previous version showed Fill 0 with one trade); MNQ1! 5m, 31 and 31, margin row 2,918 / equity 100,585 / room 97,667. Facts about the dialog: it opens on the «Publish new script» tab, so the first click is «Update existing script»; the update flow has no description field, and the script page's «···» menu only offers Copy link and sharing, so the description above still shows the 25,000 capital line and the 1,145-trade paragraph on the site; the release notes carry the change. Follow-up found the same day: a setup whose first trade closes within one bar stays armed and can place a second order; on MNQ1! 15m the HUD read FVG 70, Fill 96. Fixed in update 2 below.

## Update 2 (2026-10-25): release notes (paste as is, everything between the two rules)

Same flow as update 1: clean MNQ1! 5m chart with the updated script, Entire history on, Ctrl+S, Pine Editor > Publish… > «Update existing script» first, chart checkbox unchecked, release notes, Continue, Publish new version. No description field in this flow.

---
- One setup, one trade. The previous version closed a setup only when a position was open at a bar close, so a trade that entered and exited between two closes (an entry on a bar's close stopped out inside the next bar, or a fill and an exit inside one bar) left the setup armed, and the same limit order was placed again at the same gap, with the stop recomputed from the same sweep extreme. On MNQ1! 15m the HUD read FVG 70 and Fill 96. Now the setup closes as soon as its trade has been counted (strategy.closedtrades + strategy.opentrades grew since the previous bar), whether or not the position is still open, so Fill can no longer exceed FVG and the re-entries disappear. A setup now ends with its trade, so the state machine is free sooner: later sweeps, setups and trades can differ, and every HUD count and the trade list can change after the first former re-entry. Unfilled/expired no longer counts setups that traded.
- No change to the rules for pools, sweep, MSS, FVG detection, entry price, stop, target, kill zones, guards or the look-ahead A/B; the HUD keeps its rows.
---

Outcome (2026-10-25, about 17:33 Buenos Aires): published as update 2 from the «ICT publish» layout on MNQ1! 5m with Entire history on; the page shows both release notes, newest first. Checked on screen before publishing: MNQ1! 15m, Sweep 286, MSS 82, FVG 70, Fill 53, Unfilled/expired 233 (204 + 12 + 17, the funnel now sums exactly); MNQ1! 5m, Sweep 200, MSS 49, FVG 33, Fill 28, Unfilled/expired 172 (151 + 16 + 5). Entire history on 5m: 888 closed trades (update 1: 1,157), first 2019-05-06, last 2026-10-21; the 269 removed trades were re-entries. CSV exports in upwork.propuestas.ai/tradingview-traction/ict-update-1/.
