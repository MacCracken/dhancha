# `dh_text_degraded_runs` counted every run drawn after an enclosing operation degraded, not the run that degraded it

**Status:** 🟢 **ADDRESSED in 0.10.3 — a run is counted when the operation it drew in CROSSED into
degraded during it, once.** `dh_draw_text_ink` reads the depth `sd_flatten_op_begin` returns; at
depth > 1 its scope is nested in a consumer's operation, so it reads the standing verdict at entry
and counts only a 0 → 1 transition across its own glyph loop. At depth 1 the begin cleared the flag,
so the count is per run, as before. MEASURED (`programs/text_budget_test.cyr` group E, 115 checks):
inside one consumer scope, `b6` (degrades) then `"A"` then `b6` counts **1** (0.10.2: 3); `"A"` then
`b6` then `"A"` counts **1**, at the second run; outside any scope `b6`, `"A"`, `b6` counts **2**.
Mutations: 0.10.2's count (never reading the standing verdict) fails E; never counting when nested
fails E; a second begin fails the depth checks. The delta is 0 exactly when nothing degraded, as it
was — what changed is that the non-zero count now means something.
**Original status:** 🟡 OPEN — documented at 0.10.2 as *"the documented over-count"* and pinned by
its own gate at the inflated value.
**Filed:** 2026-09-21, by the 0.10.2 release.
**Affects:** dhancha 0.10.2.
**Severity:** **Low.** Only a consumer that wraps a whole frame in `sd_flatten_op_begin` / `_end`
(none does today: swept `crab`, `puka`, `agnos` for the call — 0 hits outside sadish and its vendored
copies) saw the count inflate, and only the exact figure was wrong; the zero / non-zero answer a
gate asks was right.

## What was found

sadish's degraded flag belongs to an OPERATION and is cleared only by a begin at nesting depth 0.
0.10.2's `dh_draw_text_ink` opened its per-run scope, drew, closed it, and counted if the flag read
1 — correct when its scope is the outermost, because then every run is its own operation. Nested in
a consumer's scope the flag is sticky until the consumer closes it, so after the run that spent the
consumer's budget every later run in that operation read 1 and was counted too. 0.10.2 said so in
the accessor's contract and in the changelog, and `text_budget_test` E asserted **2** for two runs
of which one degraded.

## What closed it

`src/surface.cyr`, the scope inside `dh_draw_text_ink`:

```
var depth = sd_flatten_op_begin();
var was = 0;
if (depth > 1) { was = sd_flatten_degraded(); }
…
sd_flatten_op_end();
if (was == 0) { if (sd_flatten_degraded() != 0) { _dh_text_degraded_runs = … + 1; } }
```

The accessor's contract now says *exactly* what a count is: the run in which an operation crossed.
With no consumer scope that is every degraded run; with one, the run that spent the last of the
consumer's budget, once per operation.

## What it does not do

It does not tell a consumer which of the LATER runs in an already-degraded operation would have
degraded on their own — inside one operation the budget is the operation's, and "on its own" is not
a question sadish can answer without a per-fill verdict it does not publish. A consumer that wants
per-label verdicts draws without an enclosing scope; that is the default.
