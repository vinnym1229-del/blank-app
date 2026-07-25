# NLM — build notes

`NLM.pine` is the original NLM indicator with the model stack added on top. Everything above
the `NLM MODEL EXTENSION` banner is the untouched foundation (sessions, PDH/PDL, PWH/PWL,
Asia/London, NY intraday, daily/weekly volume profile). No line of it was rewritten; the new
features read from `levels`, `asiaHigh/Low`, `londonHigh/Low`, `intradayHigh/Low` and
`activeProfilePoc/Vah/Val` instead of re-deriving session state.

## What is verified vs. what needs the TradingView compiler

I have no Pine compiler in this environment, so "verified" below means verified by reading the
code and by scripted static checks over the whole file, not by a successful compile.

### Verified mechanically (scripted checks over all 2361 lines)

- **Pitfall 6 — dangling blocks.** Every `if` / `else` / `else if` / `for` / `while` and every
  `=>` function header is followed by a more-indented line. Zero empty bodies.
- **Pitfall 5 — `continue`.** Not used anywhere; loops that need to skip use boolean flags
  (`skip`/`drop`/`removeDrawing`) or `break`.
- **Pitfall 9 — definition order.** Every `f_*` call site occurs at a line after that
  function's definition; no call references an undefined function.
- **Pitfall 4 — globals in functions.** No `var`-declared global is reassigned (`:=`) inside any
  function body. Helpers either mutate objects/arrays passed in as parameters, or return values
  that the top level assigns.
- **Pitfall 7 — constructor field order.** Every `Gap.new`, `Bar3.new`, `Liq.new`, `Brk.new`,
  `Trade.new`, `PivSeq.new` uses **named** arguments, so field order cannot silently mismatch.
  (The base script's `Level.new` / `ProfileDrawing.new` / `ProfileBox.new` calls are positional
  and untouched.)
- **Loop-range guards.** Every `for i = array.size(x) - 1 to 0` and every
  `for i = 0 to array.size(x) - 1` is preceded by a size guard, so no loop ever runs the
  `-1 to 0` ascending range on an empty array.
- **Pitfall 3 — `var` inside a function returned to the caller.** The persistent collections
  (`gaps`, `liqs`, `brks`, `trades`, the sweep-watch arrays, the stats arrays) are declared as
  true globals and passed into helpers as parameters.

### Verified by reading, not by execution

- **Pitfall 1 — `request.security` at or below the chart timeframe.** `f_resolve()` compares
  `timeframe.in_seconds(tf)` with `chartSeconds`: equal → the chart's native `time/open/high/
  low/close` are used and the security result is discarded; lower → the feed is reported
  unavailable (`ok = false`) and that timeframe simply detects nothing rather than returning
  degraded data. The diagnostics panel row "Feeds ok (1..6)" shows which timeframes are live.
  The LTF SMT engine uses the chart's own series for the charted symbol for the same reason;
  the comparison symbol necessarily goes through `request.security` because it is a different
  symbol (that is not the degradation case).
- **Pitfall 2 — offsets inside `request.security`.** No `[n]` is taken inside any security
  expression list for the 3-candle FVG pattern, and no `[n]` is taken on a security *result*
  to reconstruct consecutive HTF candles. Instead each timeframe keeps a rolling `Bar3` buffer
  of the last three genuinely consecutive **completed** candles of that timeframe, appended when
  the feed's bar time changes. One deliberate exception: `f_pivFeed()` fetches
  `time[len]` alongside `ta.pivothigh/pivotlow(len, len)`. That is a single scalar resolved in
  the same context on the same bar as the pivot — it *is* the pivot bar's own timestamp — and it
  is the only construct that gives an exact anchor. It is not a multi-candle pattern, and the
  result is never offset outside the call.
- **Pitfall 8 — things that grow.** Recomputed every bar while active: FVG boxes/CE lines/labels
  (`f_layoutGap`), breaker boxes/midlines/labels (`f_layoutBrk`), BSL/SSL two-segment rays and
  the centred label gap (`f_layoutLiq`, gap midpoint recomputed from the live right edge), and
  trade entry/stop/TP lines plus both zone boxes (`f_updateTrade`). Each freezes at the bar it
  resolves — swept rays freeze at `sweptTime`, TP lines freeze at the bar they are hit, the whole
  plan freezes at the resolution bar.

### Cannot verify without the compiler

- Type-qualifier acceptance of a few builtins (`table.new` position, `input.timeframe` passed to
  a `simple string` parameter). Tables are created with literal positions and repositioned via
  `table.set_position()` so the input qualifier never has to satisfy a `const` requirement.
- The exact drawing-object budget. Boxes are the tight resource: the base volume profile can use
  up to ~480 of the 500 boxes on its own. With profiles on, drop `Historical profiles to keep`
  or `Profile rows` before complaining that gaps/breakers/plans are missing boxes.
- Runtime `max_bars_back` behaviour for the dynamic history access in `f_cisdLevel`
  (`close[i]` / `open[i]` with a series `i`). The walk is capped at `bbCisdLookback`
  (default 50, max 200) and the indicator declares `max_bars_back = 5000`.

## Deliberate interpretation calls

- **CISD confirmation direction.** After a CHoCH down the streak that broke structure is
  bearish, so the CISD level is the open of the oldest candle in that bearish streak and the
  breaker confirms when price **closes back above** it (mirror for CHoCH up). That is the literal
  reading of "closes back through that CISD level".
- **BPR.** When a new FVG forms fully inside an opposite-direction IFVG of the same timeframe,
  the parent IFVG is deleted and the new gap is drawn grey and labelled `{tf} BPR` — the BPR
  supersedes the two gaps rather than stacking a third box over them.
- **Breaker box price span.** `max(open, close)` at the peak anchor bar and `min(open, close)` at
  the origin anchor bar, captured when the pivot was confirmed rather than re-read later with a
  dynamic offset.
- **Trade resolution timing.** TPs and stops are not evaluated on the entry bar, and the stop
  check on any bar uses the stop value from *before* a breakeven move made on that same bar.
  Without this a stop anchored below a wick that was swept on the entry bar resolves instantly
  as a loss.

## Diagnostics panel

`Panels → Show breaker/leg diagnostics` (on by default, bottom-left) prints the live internal
state of the leg tracker so a visual mismatch can be checked against numbers instead of guessed
from a screenshot: current bias, each frozen anchor (`origin low`, `peak high`, `origin high`,
`trough low`) with price and bars-ago, the rolling swing high/low that feed BOS detection, the
breaker count and whether the gate is open, per-timeframe gap counts, which of the six gap feeds
are live, unswept/swept BSL-SSL counts, the last sweep (side, price, bars ago), SMT freshness on
both engines, and the chart timeframe the `BB` labels derive from.
