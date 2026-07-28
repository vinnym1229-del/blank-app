# NLM — v2 build notes

`NLM.pine` = your original NLM script (untouched, lines 1–914) + a rebuilt model layer below it.

## Verification status — read this first

**I could not compile this.** There is no Pine compiler in this environment, and TradingView's
editor needs a logged-in account, which I do not have and did not attempt to use. "Verified"
below means verified by reading plus scripted checks over all 2,641 lines — not by a green
compile and not by running it on a chart.

Scripted and clean: no dangling `if`/`for`/`while`/function bodies; no `continue`; no function
used before its definition; balanced brackets on every line; every `f_*` call site matches its
definition's parameter count; every `Type.new(...)` uses named arguments and every named argument
is a real field of that type; every `obj.field` access resolves against the right UDT under
scope-aware resolution; no global `var` reassigned from inside a function; every array loop
guarded against the empty-array `-1 to 0` trap.

**First thing to do on the chart:** turn on `Panels → Show diagnostics`. The "Feeds live" row
tells you which of the six gap timeframes are actually running.

## Changes to base defaults (three, nothing else)

- `NY reset hour` 0 → **18**, so the trading day rolls at the 18:00 New York futures reopen.
- `Show POC/VAH/VAL (legacy single profile)` and `Show volume profile bars` → **false**. The new
  Daily/Weekly profile engine draws those levels with proper `VAH Daily` / `VAH Weekly` naming,
  and turning the histogram off frees the box budget (it alone can eat ~480 of TradingView's 500).

Asia, London, PDH/PDL, PWH/PWL and the live intraday high/low still come from the base engine,
still carry their dates, still go dashed-and-faded on the sweep, and still roll on the new day.

## What was built to your spec

**Gaps.** 30s / 1m / 5m / 15m / 1H / 4H. No daily, weekly or monthly.
- **Inversion is a body close on the gap's own timeframe.** A 15m gap only inverts when a 15m
  candle *closes* through it — a 1m close through it does nothing. Each timeframe's inversion
  pass runs when that timeframe's candle completes, against that candle's close.
- Re-inversion **deletes** the gap outright (chop), never fades it.
- BPR needs an FVG forming inside an IFVG **of the same timeframe**; the parent IFVG is removed
  and the survivor is grey and labelled `1m BPR`.
- 30s only fires when there is no open 1m gap **and** within 15 bars of a sweep.
- Style: black outline, dark grey fill, dashed CE midline and the label both green (bullish) or
  red (bearish), label inside the box at centre-right reading `1m FVG`. Inverted → white fill,
  `1m IFVG`, midline and text flip to the new direction. Fill transparency rises 6 points per
  timeframe step, so 4H is the faintest.
- All gaps extend to exactly 20 bars past the current bar, recomputed every bar.
- Daily wipe: 30s/1m/5m always; 15m/1H/4H only if price ran through them unrespected.
- HTF relevance: anything further than 25 × 5m-ATR from price is dropped.
- First-and-last-of-leg filter: if a move prints five 1m gaps you keep #1 and #5. Respected and
  inverted gaps are exempt.

**Overlap priority — one deliberate deviation.** You said a higher timeframe trumps an
overlapping lower one. Applied literally that would delete every 1m gap sitting inside a 5m gap,
and the confirmation leg of your own model ("1m IFVG out of the 5m or 15m gap") depends on that
1m gap existing. So pruning runs **within a tier**: 30s vs 1m, 5m vs 15m, 15m/1H/4H. Both of your
examples (1H beats 15m, 1m beats 30s) behave exactly as you described. `Prune overlaps across
every timeframe` in the FVG group switches on the literal version if you disagree.

**Breaker blocks.** BOS → CHoCH → CISD. The leg anchors freeze at the BOS and never update. The
CHoCH close *is* the breaker, so nothing is ever labelled CHoCH. The box spans swing low to swing
high using **bodies only**. A dotted CISD line is drawn and stops the bar a close takes it, which
confirms the breaker. Light grey box, `1m BB` label at centre-right.

**BSL / SSL.** 5m wick highs and lows only. Black line broken in the middle with `BSL` / `SSL`
centred in the gap, tagged `BSL+VAH` / `SSL+POC` when it lines up with a Daily or Weekly profile
level. On the sweep it becomes `$$$` and the line stops dead at the sweep candle. Swept rays clear
10 minutes later; anything left dies on the day roll.

**SMT.** Wick to wick between the two actual pivots, `SMT` centred on the line, solid or dotted.
Two engines: HTF (15m source) and LTF (always the chart's own timeframe). Suppressed when the
comparison symbol is already trading through its own last pivot.

**Stops.** Wick first — swept wick plus buffer. If that stop is wider than 1.5 × 5m ATR it falls
back to the wick of the LTF confirmation object (the 1m IFVG's or breaker's own wick). The
checklist shows which one is in use.

**Targets.** TP1 = the first real wick at or beyond 1R; TP2 and TP3 are the next real levels
beyond that. Candidates are 5m swing wicks, session and intraday highs/lows, Daily/Weekly
VAH-VAL-POC, and HTF gap CE lines — never a floating R-multiple unless nothing qualifies, and the
checklist says `TPs unanchored` when that happens. TP4 is off by default (50/40/10 → 50/40/5/5).

**Sizing.** `Auto (charted symbol)` uses `syminfo.pointvalue` so it works on stocks; otherwise
MNQ / NQ / MES / ES, defaulting to **MNQ with $600 risk** so you can chart NQ and execute micros.
Always rounds **down**, and shows `2-3` when the exact figure is close to the next contract.
75% size below B+ (grade and percentage both configurable), 50% on Monday and Friday.

**Gates.** 09:35–15:00 plus an Asia window 20:00–23:00, news blackouts (08:30 / 10:00 / 14:00
built-in plus two custom slots), a 0–100 chop score, and minimum RR 2.0. Any gate failing = No Trade.

**Continuation vs reversal** is classified and tracked separately in the stats table, alongside
per-grade Full / 3TP / 2TP / 1TP / Loss buckets, binary win% (any profit = win) and average % of
target captured.

## Four calls I made because you didn't answer

All four are settings, so change them without touching code:
1. Stop fallback trigger — **ATR multiple** (`stopMaxAtr`, 1.5 × 5m ATR).
2. Chop — **combined score** of ATR compression, body overlap and range containment (`chopThreshold`, 60).
3. News — **built-in recurring plus two custom windows**.
4. HTF gap filter — **distance + respected + first/last combined** (`htfDistAtr`, `htfNeedRespect`, `gapFirstLast`).

## Deep-review pass — six real bugs found and fixed

These are logic bugs, not lint findings. Each one changed behaviour.

1. **The stop ignored the swept level.** It used only `lowest(low, 10)` / `highest(high, 10)`,
   so if the sweep happened more than 10 bars ago the stop landed on an arbitrary recent extreme
   while the panel still claimed "swept wick". It now anchors on the actual swept price when a
   matching sweep exists, and the panel says "recent wick" honestly when one does not.
2. **The confirmation-wick fallback was direction-blind.** It picked the *nearest* inverted LTF
   gap or confirmed breaker regardless of which way it pointed, so a long could anchor its stop
   under a bearish object. It now requires the object to face the trade's direction.
3. **A bare sweep with no confirmation could reach grade B and be traded.** Sweep alone sets a
   direction, and Condition + Condition+ + Pullback + Pullback+ + RR sums to 75% = B, which
   passes a B- minimum. Confirmation is now a hard gate (`Confirmation is a hard gate`, on).
4. **Plans could be built on floating R-multiples.** When no real wick sat beyond 1R the target
   fell back to a bare multiple — exactly the "floating in air" case you called out. Targets must
   now sit on a real wick or level (`Targets must sit on a real wick / level`, on).
5. **Nothing capped the total gap count.** Overrunning `max_boxes_count` makes TradingView
   silently drop the oldest drawings while the arrays still hold their ids, so gaps vanish for no
   visible reason. Hard ceiling of 24, dropping the lowest-timeframe oldest gap first.
6. **`g.bull == not wantHigh` would have misparsed.** `not` binds *looser* than `==` in Pine.
   Rewritten as `g.bull != wantHigh`.

Also added: RR is now measured to a target you choose (TP1/TP2/TP3, default TP2) instead of being
hard-wired to TP2, and an `NLM setup` alertcondition fires when a plan is drawn.

## Scroll drift — round 2

Round 1 moved the *boxes and rays* to `bar_index`. It missed the **labels**: all 7 label sites
(session levels, profile levels, intraday H/L, and the Daily/Weekly profile) anchored at
`f_labelX()` = `time + 14 x chartMs` — a future timestamp. TradingView projects future
timestamps using the visible bar spacing, so they slid every time the chart was scrolled or
zoomed. They now use `f_labelXi()` = `bar_index + 14` with `xloc.bar_index`.

## Breaker boxes were screen-wide

The box anchored at `math.min(originBar, peakBar)` — the *older* of the two swings. On a
trending 1m chart the swing low behind a BOS can be 700 bars back, so the box covered the whole
screen. It now anchors at the swing that was actually broken (`math.max`) and is width-capped by
`Max breaker width (bars)`, default 120. The box's price range is unchanged: still the full
swing-low body to swing-high body.

## Runtime

Was near the limit. Six changes, no behaviour lost:
- Layout is deferred to the last bar. Drawings persist, so only the final bar's positions are
  ever visible; repositioning every object on every historical bar was pure waste.
- The O(gaps squared) overlap pass runs only when the gap set actually changed, not every bar.
- The sweep-watcher sync (O(levels x watchlist)) runs only when a session level appears. The live
  intraday high/low left the watchlist — it moves every bar and the BSL/SSL rays cover those wicks.
- BSL/SSL session dedup re-checks only when a session level appears.
- The four take-profit anchor searches run only when a plan is actually possible.
- Two `request.security` calls folded away (13 to 11): the ATR reference rides on the MTF-a feed,
  and the comparison symbol's live close rides on its own pivot feed.
- `Lower-TF bars requested` default 100000 to **20000**. This is the single biggest lever in the
  whole script — raise it only if the profile looks short on history.

## Scroll drift — fixed

Every drawing that extends past the last bar (gaps, breaker boxes, BSL/SSL rays, the whole trade
plan) now anchors on `xloc.bar_index`, not `xloc.bar_time`. A future *timestamp* has no bar to
sit on, so TradingView projects where it would land, and that projection shifts when you scroll
or zoom — which is why the gaps slid around. A bar index is exact and cannot drift. The three-
candle buffers now carry the chart bar index of each completed higher-timeframe candle, so a 4H
gap still anchors to its real displacement candle rather than an estimate.

Session levels, the Daily/Weekly profile lines and the SMT line stay on `bar_time`: the first two
use `extend.right`, which TradingView resolves natively, and the SMT line runs between two past
pivots, so none of them project into the future.

## Entry label

Prints on the entry bar as `Long (A+) - 3` (below the bar for longs, above for shorts). On
resolution the outcome is appended and the text recolours: faded green if it closed beyond
breakeven, **black if it reached TP1 and then gave it back at breakeven**, faded red on a
straight loss.

## Known limits

- **30s gaps only work on a 30s or lower chart.** Pine cannot get true sub-chart resolution from
  `request.security`, so on a 1m chart the 30s feed reports itself unavailable rather than
  returning fake data. The diagnostics panel shows this.
- Box budget is the tight resource. If gaps or plans stop drawing, that is what ran out.
- The CISD walk-back uses dynamic history indexing capped at 50 candles.
