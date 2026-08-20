# How this strategy avoids repainting — and how to prove it

"Non-repainting" is the single most-requested and least-verified claim in ICT
tooling. This document states where repainting actually comes from, what this
implementation does about each case, and how to reproduce the difference
yourself in the Strategy Tester.

The short version: **a signal is non-repainting when the bar that produced it
could not have known anything that happened after its own close.** Everything
below is an application of that one sentence.

---

## The four vectors

### 1. Higher-timeframe leakage — `request.security()`

The bug:

```pine
pdh = request.security(syminfo.tickerid, "D", high, lookahead = barmerge.lookahead_on)
```

On historical bars this returns the **finished** high of the day the bar
belongs to. A 09:45 bar is handed the day's final high — a number that will
not exist for another six hours. Sweeps of that level look surgical. Live, the
value crawls upward all day and the signals never reproduce.

The fix used here:

```pine
pdh = request.security(syminfo.tickerid, "D", high[1], lookahead = barmerge.lookahead_on)
```

The `[1]` offset is what makes this safe, and it is the reason `lookahead_on`
appears in correct code at all. The two together are the documented idiom for
"the last **closed** higher-timeframe bar", and they return the same value on
historical and realtime bars — which is the only definition of non-repainting
that matters.

`lookahead_on` alone is the bug. `lookahead_on` with the `[1]` offset is the
fix. They are not the same thing, and code review that flags the keyword
without reading the offset gets this backwards.

### 2. Pivot lag — `ta.pivothigh` / `ta.pivotlow`

`ta.pivothigh(left, right)` does not exist until `right` bars after the extreme
prints. Drawn at the pivot's own index — which is what every "swing high"
indicator does visually — the level appears in the past, on a chart, at a
moment when nobody could have known it.

This implementation accepts the lag rather than hiding it. A swing pool becomes
tradeable `pivR` bars after the extreme, never before. That is a real cost:
fast moves get their structure confirmed late.

**The consequence worth arguing about before writing code:** a Market Structure
Shift defined as "close through the last *confirmed* pivot" will miss breaks in
fast trends, because in a strong leg the pivot has not confirmed yet. The
alternative is tracking the running extreme separately, which is legitimate but
changes what MSS *means*. There is no universally right answer — there is only
a decision, made explicitly, and documented. This build uses the confirmed
pivot.

### 3. Intrabar evaluation

Any condition read on a forming bar flips while the candle is alive. A sweep
that is true at 09:47 can be false at 09:50 on the same bar.

This build sets `calc_on_every_tick = false` and `process_orders_on_close =
true`, and every state transition is evaluated on a closed bar. Where a rule
genuinely needs the intrabar path — did price *touch* the gap — it is expressed
as a resting **limit order**, so the fill is an actual touch rather than an
assumption about what happened inside the bar.

### 4. Definition drift

The quiet one, and the one that costs the most in a real engagement. "Liquidity
sweep" has at least two defensible definitions:

- **Close back inside.** The bar trades through the pool and closes back on the
  origin side. Strict, later, fewer signals.
- **Wick through, confirmation deferred.** The wick arms the setup and the
  displacement of the MSS is the only confirmation. Earlier, more signals, more
  noise.

Neither is *correct*. Shipping one silently and calling it "the ICT rule" is
how a project ends in an argument about what was agreed. Here it is
`sweepConfirm`, an input with both options, and the same is true for the FVG
entry edge (proximal / distal / midpoint) and the target (opposing liquidity /
R multiple).

---

## Proving it: the A/B protocol

The `Look-ahead A/B demo` input group exists so the claim can be checked rather
than believed. It flips **one** thing: whether the daily pools come from the
previous day's closed bar or from the current day's finished extremes.

1. Load the script on **MNQ1! or MES1!, 5m**, with a fixed date range.
2. Leave `DEMO — leak the CURRENT day's high/low into history` **off**. Record
   net profit, profit factor, max drawdown, number of trades. The HUD reads
   `Honest build`.
3. Turn it **on**, change nothing else. The HUD reads `⚠ LOOK-AHEAD DEMO`.
4. Compare.

The leaked version sells the exact high of the day and buys the exact low,
because that is precisely what it was told. The gap between the two equity
curves is not an edge. It is the size of the lie — and it is roughly the size
of the gap between a backtest that sold a strategy and a live account that
disproved it.

Two honest caveats about what this demo does and does not show:

- It isolates **one** vector. A build can pass this A/B and still repaint
  through pivot handling or intrabar evaluation. The A/B is a demonstration of
  method, not a certificate.
- Neither curve is a claim about profitability. Every threshold in the script
  is a documented starting point, not a tuned value, and no parameter here was
  selected by searching the same data it is measured on.

## The verification I would run on someone else's code

Given a third-party ICT indicator and the claim "non-repainting", in order:

1. `grep` for `request.security` and read every call for the `[1]` offset and
   the `lookahead` argument together. Most claimed bugs live here, and so do
   most false accusations.
2. Check whether `calc_on_every_tick` is on, and whether any signal is
   evaluated without `barstate.isconfirmed`.
3. Find the swing/pivot source and confirm the level is used no earlier than
   its confirmation bar, not merely *drawn* there.
4. Compare a saved screenshot of the live chart against the same chart reloaded
   an hour later. Anything that moved, repainted. This step is the only one that
   needs no source access, and it is the one clients can run themselves.
