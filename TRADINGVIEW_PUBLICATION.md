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
- Initial capital: 25,000, realistic for one micro futures contract.
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

The chart shows MNQ1! on the 5-minute timeframe. The Strategy Tester report covers [DATE RANGE] with [N] closed trades using the default properties above.

---

## Publishing checklist

1. Open `ict-sweep-mss-fvg.pine` in the Pine Editor and fix any compile error. Compilation has not been checked locally.
2. Add the script to a clean MNQ1! 5m chart (standard candles, no other scripts, symbol, timeframe and script name visible).
3. Strategy Tester with the default inputs and default properties:
   - More than 100 closed trades. If fewer, widen the date range with Deep Backtesting before changing anything else.
   - No caution warning in the report.
   - Properties tab equals the STRATEGY PROPERTIES section above (25,000 capital, 1 contract, 5% margin, 2.25 per contract commission, 1 tick slippage, process orders on close).
4. Confirm the look-ahead demo input is OFF (the table reads "Honest build").
5. Fill the placeholder line at the end of the description with the actual date range and trade count.
6. Publish as Public and Open-source. Categories: Chart patterns or Support and resistance, plus Futures if available.
7. Check the description for links, contact info and promotion: there must be none, and no performance figures.
