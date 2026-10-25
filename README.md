# ICT — Liquidity Sweep · MSS · FVG (index futures)

A TradingView **Pine Script v6 `strategy()`** that translates the ICT entry
model into objective, non-repainting rules:

```
Liquidity pool → Sweep → Market Structure Shift (displacement) → FVG → limit entry
```

Built for index futures (MNQ / MES / NQ / ES) on 1–15 minute charts, with kill
zones, an opposing-liquidity target, and risk guards (daily loss lock,
timed flatten).

The point of this repository is not that it implements ICT concepts. It is
**how each concept was turned into a testable condition, and how the
non-repainting claim is proven rather than asserted** — see
[REPAINTING.md](REPAINTING.md).

## The rules, stated plainly

| Concept | The condition actually implemented |
|---|---|
| **Liquidity pool (BSL / SSL)** | Previous day high/low, and/or confirmed swing highs/lows. The active pool is the **nearest** level on each side of price — liquidity is taken nearest-first. Optionally restricted to **equal highs/lows** (two consecutive confirmed swings within a tick tolerance). |
| **Sweep** | Price trades through the pool and — in the strict mode — **closes back on the origin side**. The signal stamps on that bar's close and never moves. The alternative (wick through, confirmation deferred to the MSS) is an input, not a hidden default. |
| **MSS** | A **displacement** close (body ≥ ATR × multiplier) through the opposing confirmed swing, away from the swept pool, within N bars of the sweep. |
| **Displacement** | Candle body ≥ `dispMult` × ATR. Measured on the body, not the range: a long wick is not displacement. |
| **FVG** | The 3-bar imbalance left by the displacement leg (`low[2] > high` bearish, `high[2] < low` bullish), known in full at the close of the third bar. Gaps thinner than N ticks are ignored, and the gap must sit inside the leg that came off the sweep. |
| **Entry** | A resting **limit order** at the proximal, distal or midpoint edge of the gap. A fill is a real touch, not a same-bar assumption. Cancelled if unfilled after N bars. |
| **Stop** | Beyond the **sweep extreme** plus an ATR buffer — the level that invalidates the premise, not a fixed distance. |
| **Target** | The **opposing liquidity pool**, with an R-multiple fallback when no pool qualifies. |
| **Kill zones** | London and New York windows, timezone-aware via `input.session` + an explicit timezone. Setups outside the windows are not taken. |
| **Risk guards** | Daily loss lock (no new setups after the day's P&L breaches the limit) and a timed flatten window. |

Each setup lives in a typed `Setup` object with an explicit phase
(`Idle → Swept → MSS → Armed`) and its own expiry per phase, so a stale setup
can never leak into a trade and phases cannot be skipped. Each pool fires at
most one setup — a sweep is an event, not a state.

## What is deliberately an input, not a decision

Where an ICT concept has more than one defensible definition, this build
exposes the choice instead of silently picking one:

- What confirms a sweep — close back inside, or deferred to the MSS displacement.
- Which FVG edge is the entry — proximal, distal, or midpoint.
- What the target is — opposing liquidity, or an R multiple.
- Whether pools must be equal highs/lows.

This is the part that matters in a real engagement. Most disputes on ICT
projects are not bugs; they are two people using one term for two different
rules.

## Usage

1. TradingView → Pine Editor → paste `ict-sweep-mss-fvg.pine` → *Add to chart*.
2. Suggested context: **MNQ1! or MES1!, 5m**, kill zones on.
3. The on-chart HUD shows the live setup state and the funnel counters
   (`Sweep → MSS → FVG → Fill`, plus expiries and day P&L) — the instrumentation
   used to debug where setups die. `Fill` is read from `strategy.closedtrades +
   strategy.opentrades`, so it counts filled trades, including one that enters on
   a bar's close and exits inside the next bar. It counts trades, not setups: a
   setup whose first trade closes within one bar stays armed and can place a
   second order. A last row shows the margin one contract needs at the header's
   5%, the current equity and the room between them (the available funds once
   one contract is open), and warns when that room is under one ATR: below
   zero, entries did not fill in the observed run; between zero and one ATR an
   entry can fill, and an adverse move of about one ATR or less triggers a
   margin call. NQ at 25,000 needs about 25,000 of margin; MNQ about a tenth
   of that.
4. Alerts emit a JSON `alert_message` for a webhook → broker bridge. Order
   dedupe, position sync and bracket placement belong server-side; Pine only
   signals.

## Verifying the non-repainting claim

Do not take it on faith — the script ships with the counter-example built in.
Flip the `Look-ahead A/B demo` input and compare the two equity curves. Full
protocol and caveats in [REPAINTING.md](REPAINTING.md).

## Honest scope

This is a **working prototype of an entry model**, published as verifiable work,
not a tuned or profitable system. Every threshold is a documented starting
point. No parameter was selected by searching the same data it is measured on,
and no backtest figure is quoted here as a performance claim.

Related: [brrc-pine-strategy](https://github.com/guillemustafa-ux/brrc-pine-strategy) —
the same engineering approach applied to a continuation setup.

## License

MIT. See [LICENSE](LICENSE).

## Disclaimer

Educational prototype for evaluating setup logic. Not financial advice; futures
carry substantial risk. Backtest results do not guarantee future performance.
