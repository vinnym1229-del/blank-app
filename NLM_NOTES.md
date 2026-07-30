# NLM — v2 build notes

`NLM.pine` = your original NLM script (untouched, lines 1–914) + a rebuilt model layer below it.

## Verification status — read this first

**I could not compile this.** There is no Pine compiler in this environment, and TradingView's
editor needs a logged-in account, which I do not have and did not attempt to use. "Verified"
below means verified by reading plus scripted checks over all 2,802 lines — not by a green
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
- 30s is **off by default** (`Enable the 30s feed`). It cannot work above a 30s chart, and a
  seconds request errors outright on some symbols. When on, it fires only if no 1m gap is open
  and within 15 bars of a sweep.
- Style: black outline, dark grey fill, dashed CE midline and the label both green (bullish) or
  red (bearish), label inside the box at centre-right reading `1m FVG`. Inverted → white fill,
  `1m IFVG`, midline and text flip to the new direction. Fill transparency rises 6 points per
  timeframe step, so 4H is the faintest.
- All gaps extend to exactly 20 bars past the current bar, recomputed every bar.
- Daily wipe: 30s/1m/5m always; 15m/1H/4H only if price ran through them unrespected.
- HTF relevance: anything further than 25 × 5m-ATR from price is dropped.
- First-and-last-of-leg filter: if a move prints five 1m gaps you keep #1 and #5. A gap is exempt
  once price has **touched** it, or it has been respected, inverted or promoted to a BPR — the
  same protection applies to the overlap prune and the 18-gap ceiling.

**Overlap priority — one deliberate deviation.** You said a higher timeframe trumps an
overlapping lower one. Applied literally that would delete every 1m gap sitting inside a 5m gap,
and the confirmation leg of your own model ("1m IFVG out of the 5m or 15m gap") depends on that
1m gap existing. So pruning runs **within a tier**: 30s vs 1m, 5m vs 15m, 15m/1H/4H. Both of your
examples (1H beats 15m, 1m beats 30s) behave exactly as you described. `Prune overlaps across
every timeframe` in the FVG group switches on the literal version if you disagree.

**Breaker blocks.** BOS → CHoCH → CISD. The leg anchors freeze at the BOS and never update. The
CHoCH close *is* the breaker, so nothing is ever labelled CHoCH. The box spans swing low to swing
high using **bodies only**, anchored at the swing that was broken and width-capped by
`Max breaker width (bars)` (120) so it cannot span the screen. A dotted CISD line is drawn and
stops the bar a close takes it, which confirms the breaker. Light grey box, `1m BB` at
centre-right.

**BSL / SSL.** 5m wick highs and lows only. Black line broken in the middle with `BSL` / `SSL`
centred in the gap, tagged `BSL+VAH` / `SSL+POC` when it lines up with a Daily or Weekly profile
level. On the sweep it becomes `$$$` and the line stops dead at the sweep candle. Swept rays clear
10 minutes later, unswept rays expire after 5 days, and anything left dies on the day roll.

**SMT.** Wick to wick between the two actual pivots, `SMT` centred on the line, solid or dotted.
Two engines: HTF (15m source) and LTF (always the chart's own timeframe). Suppressed when the
comparison symbol is already trading through its own last pivot.

**Stops.** Wick first — the swept level itself when one matches the direction, otherwise the
6-bar extreme, plus a buffer. If that is wider than **1.0 × 5m ATR** it falls back to the wick of
the LTF confirmation object facing the trade (the 1m IFVG's or breaker's own wick). The checklist
names which is in use, and says `recent wick` rather than `swept wick` when no sweep is involved.

**Targets.** TP1 = the first real wick at or beyond `TP1 minimum R` (1.0 by default, tunable);
TP2 and TP3 are the next real levels beyond that. Candidates are **chart-timeframe and 5m** swing
wicks, session and intraday highs/lows, Daily/Weekly VAH-VAL-POC, and HTF gap CE lines. Only TP1
is gated on snapping to a real level — TP2/TP3 fall back to R-multiples when nothing is left in
front of price, and the checklist reports `2/3 TPs on wicks`. TP4 is off by default
(50/40/10 → 50/40/5/5).

**Sizing.** `Auto (charted symbol)` uses `syminfo.pointvalue` so it works on stocks; otherwise
MNQ / NQ / MES / ES, defaulting to **MNQ with $600 risk** so you can chart NQ and execute micros.
Always rounds **down**, and shows `2-3` when the exact figure is close to the next contract.
75% size below B+ (grade and percentage both configurable), 50% on Monday and Friday.

**Gates**, any one of which forces No Trade: the 09:35–15:00 session plus an Asia window
20:00–23:00; news blackouts (08:30 / 10:00 / 14:00 built-in, each individually switchable, plus
two custom slots that ship **off**); a 0–100 chop score; minimum RR 2.0 measured to a target you
choose; confirmation present; TP1 anchored on a real level; and the sweep → MTF gap →
confirmation **order**. The `Blocked by` checklist row names whichever one is failing.

**Continuation vs reversal** is classified and tracked separately in the stats table, alongside
per-grade Full / 3TP / 2TP / 1TP / Loss buckets, binary win% (any profit = win) and average % of
target captured.

## Four calls I made because you didn't answer

All four are settings, so change them without touching code:
1. Stop fallback trigger — **ATR multiple** (`stopMaxAtr`, now 1.0 × 5m ATR).
2. Chop — **combined score** of ATR compression, body overlap and range containment (`chopThreshold`, 60).
3. News — **built-in recurring plus two custom windows**.
4. HTF gap filter — **distance + respected + first/last combined** (`htfDistAtr`, `htfNeedRespect`, `gapFirstLast`).

## Why nonsensical trades were firing

`Condition`, `Pullback` and `Confirmation` were three **independent time windows**
that only had to OVERLAP. A session sweep 18 bars ago, an unrelated MTF gap touch
20 bars ago, and a 1m IFVG 3 bars ago pointing whichever way it liked would all be
"fresh" at the same moment, and the model fired. Nothing required them to be the
same event unfolding, or even to happen in order.

Your model is a **sequence**: sweep -> price pulls into the MTF gap -> a LTF gap
inverts coming back out of it. That order is now enforced —
`sweep bar <= MTF gap bar <= confirmation bar`. The checklist reports
`out of sequence` when the legs are present but jumbled, so you can see it rather
than wonder. `Enforce sweep -> MTF gap -> confirmation ORDER` in the Model group
turns it off if it proves too strict.

## Styling audit against your spec

Audited the code line by line against every point in your styling brief rather than
assuming earlier rounds had covered it. One item genuinely was not done.

**Profile levels were still banners.** `VAH Weekly`, `POC Daily` and the rest drew as
solid coloured bubbles — the exact "massive banner" style you said should be reserved
for the overnight session highs and lows. They now draw like `$$$` and `BSL`: a dashed
line broken near the right edge with the name sitting in the break, no bubble, text in
the level's own colour so Daily and Weekly stay distinguishable.

A check now enforces this: any bubble-style label that is not a session high/low fails
the build. The six that remain are exactly the ones you wanted kept — Asia, London,
PDH/PDL, PWH/PWL and the intraday IDH/IDL.

Everything else in the brief was already in place and verified by reading the code:
FVG label centre-right inside the box reading `1m FVG`, black outline with dark grey
fill, CE midline and text green or red by direction, white fill and `1m IFVG` once
inverted, light grey `1m BB`, transparency rising with timeframe, BSL/SSL black with
the label centred in the line break turning to `$$$` and freezing at the sweep candle,
`BSL+VAH` / `SSL+POC` tagging, SMT wick to wick with the label on the line, and no
CHoCH label anywhere since the CHoCH close *is* the breaker.

**Plan levels now name themselves fully:** `ENTRY 27721.50`, `STOP LOSS 27601.25`,
`TP1 / BE 50% 27812.00`, and each target appends `HIT` the moment it is reached, on top
of turning solid and thick.

## Follow-up review of the sequence gate

The gate as first written was nearly unsatisfiable. `mtfGapBar` was stamped with
`bar_index` on **every** bar price sat inside the MTF gap, so while price was still
in the gap it always equalled "now" — and `mtfGapBar <= confirmation bar` can never
hold when the confirmation landed a bar or two earlier. It only passed in the
narrow case where price had already left the gap.

`mtfGapBar` now records the bar the current **visit** into the gap began, which is
the "pulled in" event the model actually means. A separate `mtfLastInBar` tracks how
recently price was inside, which is what the Pullback freshness window wants. Two
different questions that were sharing one variable.

## Why TP1 was not on the near wick

The stop was too wide, so 1R was too far, so the first anchor at or beyond 1R
skipped straight past the wick you expected. Both of your notes — TP1 belonging
where the final target sat on trade one, and TP1 belonging at the recent wick high
on trade two — are the same symptom.

The wick search dropped from 10 bars to 6, and "too large" fell from 1.5x to 1.0x
the 5m ATR, so the confirmation-candle stop takes over sooner. A tighter stop pulls
1R in and TP1 lands on the near wick. `TP1 minimum R` is now an input if you want
TP1 to snap even closer than 1R.

## Entry banner

The entry candle carries a filled banner: `LONG B | 19 MNQ` on the first line, the
setup and the reason on the second — `Continuation | sweep > in MTF gap > IFVG` —
so the chart itself answers "why was this taken". On resolution the outcome is
appended and the banner fills solid: green for a win, black for breakeven, red for
a loss. Target lines are green dashed and turn solid and thick as each one is hit;
the stop line is red, the entry line black.

## The historical-buffer error — root cause

```
Error on bar 24317: The requested historical offset (4474) is beyond the
historical buffer's limit (4473).   at f_gapLine():966 / f_layoutLiq():1256
```

**A past `bar_index` coordinate is capped by `max_bars_back`. A `bar_time`
coordinate is not.** When drawings moved to `xloc.bar_index` to kill the scroll
drift, every long-lived object became a ticking clock: a BSL ray created 24,000
bars ago still handed its start index to `line.set_xy1`, and Pine refused. Cutting
`max_bars_back` from 5000 to 500 did not cause this — it just made it fire sooner.

The fix is a split by direction, not a blanket choice:

- **Anchors reaching into the past** — gap and breaker box left edges, ray starts,
  trade plan starts — are back on `xloc.bar_time`. Timestamps have no buffer limit,
  so a ray can be a week old and still draw.
- **Anchors reaching only forward** — the session and profile labels at
  `bar_index + 14` — stay on `xloc.bar_index`. A *future* offset has no buffer
  limit and no projection, so these keep the drift fix. That was the real drift all
  along: labels parked on a future timestamp, not the boxes.

A regression check now fails the build if any `xloc.bar_index` site appears outside
the forward-looking label anchors.

Two things came out of the revert for free: gap boxes anchor on the **open time of
the oldest of the three candles**, which is exactly "start where the first candle of
the gap starts" with no timeframe arithmetic to get wrong; and unswept BSL/SSL rays
now expire after 5 days, since a 24,000-bar-old untouched ray is not information.

## "Works for a few seconds then goes away" — the memory/time limit

That symptom is not a compile error. The script compiles, starts drawing, and is then killed by
TradingView's runtime limit. The cause was `max_bars_back = 5000` in the `indicator()` call:
that forces Pine to allocate a **5000-bar history buffer for every series variable in the
script**, and this script has hundreds of them.

It was set to 5000 when the CISD walk-back indexed series with runtime offsets. That code is
gone, so the deepest lookback anything now needs is `chopAtrRef` at its 300 maximum plus the ATR
window — about 320 bars, and only 64 at default settings. `max_bars_back` is now **500**, a 10x
cut in allocated history, with a check that verifies it still covers the deepest possible
lookback.

Three other allocations came down with it: the lower-timeframe request 20000 to 10000 chart bars,
the profile sample cap 10000 to 5000, and the live drawing budget (gaps 24 to 18, unswept rays 12
to 10, breakers 6 to 4, retained plans 3 to 2). Every drawing is retained memory.

## TP1 / TP2 placement

The anchor pool only held **5m** pivot wicks. The wicks you circled are **1m** swings, which is
why the targets missed them. The chart's own swing wicks now feed the pool alongside the 5m ones,
so a target can land on the 1m wick that price actually respected.

## The 1m runtime error

You did not send the message text, so this is elimination rather than diagnosis. The single
highest-risk construct in the script was the CISD walk-back: `close[i]` / `open[i]` with `i`
computed at runtime inside a `while` loop. Indexing a series with a runtime-computed offset is
the classic trigger for *"Pine cannot determine the referencing length of a series"*, which is
thrown at **runtime** and pulls the indicator off the chart — matching "gives an error and goes
off my chart".

It is gone. The CISD level is now carried forward bar by bar: on every bar the script remembers
the open of the first candle of the consecutive same-direction streak running into it. Identical
definition, no dynamic indexing anywhere in the script, and faster. A scan confirms zero
series-with-computed-offset accesses remain.

Also hardened: a seconds timeframe is now only ever requested when the chart itself is at or
below it. Even with the 30s feed switched on by mistake on a 1m chart, the script asks for the
chart's own timeframe, which always resolves.

**If the error survives this, send me the exact message.** It appears under the indicator name on
the chart, or in the Pine Editor console. One line of it and I can name the cause instead of
narrowing.

## 30s charts were silently dead

`request.security_lower_tf` returns an **empty array** when the requested timeframe is above the
chart's. The base engine asks for 1m data, so on a 30s chart it got nothing back and every
session level, PDH/PDL, and the whole volume profile silently vanished — no error, just absence.
The calculation timeframe is now clamped to the chart, so 30s and 1m both work.

## The reason no trade ever printed

`input.session("0000-0000")` was used as an "off" sentinel for the two custom news windows.
TradingView does **not** read that as an empty session — midnight to midnight is the **full
24-hour day**. Both custom windows therefore matched every bar, `inNews` was permanently true,
`sessionOk` permanently false, and every gate downstream of it failed. That is why the panel said
`Blocked by: news window` at 01:01, and why the whole model was inert.

Every news window now has its own explicit on/off switch; the two custom windows ship off. No
session string is ever used as a sentinel again.

## The 30s feed is off by default

`request.security` to a seconds resolution errors outright on some symbols and account plans, and
it cannot return sub-chart resolution anyway — on a 1m chart it was pure cost for no data. It now
requests the chart's own timeframe when disabled, so nothing is asked for that cannot be served.
Turn it on only on a 30s or lower chart.

## Touched gaps are never pruned

A 5m FVG that price has already tapped is the setup, not clutter. Three separate mechanisms could
delete one: the first-and-last-of-leg filter, the higher-timeframe overlap prune, and the hard
ceiling. All three now treat a gap as protected once it has been touched, respected, inverted or
turned into a BPR, and the ceiling drops untouched gaps first.

## Gap alignment + why nothing was printing

**Boxes started too far right.** The left edge anchored at the bar where the *middle* candle
CLOSED. On a 1m gap that is only one bar off, but on a 15m gap viewed on a 1m chart it lands 15
bars right of where the pattern begins, and on a 1H gap, 60 bars. Boxes now anchor at the OPEN of
the oldest candle of the three, so the box starts exactly where the gap-forming sequence starts,
on every timeframe.

**No trades were printing because of a gate I added the round before.** Requiring TP1, TP2 *and*
TP3 to all snap to a real level sounds right, but near a session extreme there are simply not
three levels left in front of price, so the gate could never pass. Only **TP1** is gated now —
the one that actually matters. TP2 and TP3 still prefer real levels and fall back to R-multiples
when none remain; the checklist reports `2/3 TPs on wicks` so you can see it.

**The checklist now has a `Blocked by` row** naming the first failing gate: `outside session`,
`news window`, `chop 71`, `no direction yet`, `no IFVG / breaker`, `RR 1.4 < 2.0`, `TP1 not on a
wick`, `score 54% < 60`, `below B-`, `trade already open`, or `cooldown`. No more guessing.

Confirmation freshness went 3 to 5 bars, and gaps within 8 ticks of an existing gap on the same
timeframe now merge instead of stacking as near-duplicates.

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
