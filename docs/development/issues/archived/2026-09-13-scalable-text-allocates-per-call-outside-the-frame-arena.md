# `dh_draw_text_ink`'s scalable branch allocates a full-surface canvas per call, outside the frame arena

**Status:** 🟢 **ADDRESSED in 0.10.0 — the scalable branch now costs the global heap exactly what the
bitmap branch costs per frame: nothing.** Three moves, one per repo. **sadish 0.5.5** put every
per-call `alloc(` site (23 of its 32) behind `sd_alloc(n)` with `sd_alloc_set(fp)` as the hook
(returns the previous one), made the other 9 — its fixed-capacity fill/stroke scratch — process-
lifetime on the global heap (589,896 B once, instead of 327,824 B per fill; plus w*8 for a per-row
accumulator whenever a canvas wider than any before arrives), and added `sd_canvas_blit_at`. **rekha 0.3.10** draws its outline scratch from
that same seam, so the 136 B per simple glyph (70,856 B per composite) it used to leak under any hook
follows the paths. **dhancha 0.10.0** installs `dh_falloc` as sadish's hook for the duration of one
`dh_draw_text_ink`, restoring the PREVIOUS hook after, and sizes the canvas to the clip ∩ surface ∩
run instead of the surface — the clip stack is gone because the canvas bounds ARE the clip, pixel for
pixel (asserted). MEASURED, `programs/text_arena_test.cyr`, the filing's own shape at 380x220 with a
proportional face that rasterises: one 10-character label **2,513,248 B per call → 38,184 B** (the
same 38,184 B on a 760x440 surface); a frame of 12 labels **26,645,208 B → 417,056 B** with no arena;
under a frame arena **26,640,472 B per frame on the global heap → 0 B over twenty frames**, the whole
frame reclaimed at `dh_frame_begin`. What remains on the arena is the path, not the canvas: 4,328 B
per 3-point glyph, of which 4,144 B is `sd_path_new`'s opening capacity — a sadish/rekha question, and
not a leak. ⚠ **crab's hand-off, MEASURED against crab 0.8.10 rather than predicted:** (1) crab pins
sadish 0.5.4 / rekha 0.3.7 TAG-ONLY, and against those with dhancha 0.10.0 by path its build is
REFUSED — `refusing to emit binary with 2 reachable undefined function(s)` (`sd_alloc_set`,
`sd_canvas_blit_at`) — so crab must move `[deps.sadish]` to >= 0.5.5 and `[deps.rekha]` to >= 0.3.10
once the tags exist (CHANGELOG 0.10.0, dependencies). (2) With those pins, the assertion that trips
is NOT `scost > 0` but `arena_capacity_total(farena) == cap0` (got 468,040, expected 16,384): crab's
8 KiB one-bitmap-frame arena chains ~452 KB of chunks from the global heap on the FIRST face frame
(glyph paths at ~4.3 KB each; `scost` 452,520 B), and the second face frame costs 0 B. ⇒ The
flip to `scost == 0` needs one warm-up face frame (or an arena sized to hold one) and `cap0`
re-baselined after it; a warm-up must also cover the WIDEST run crab will draw, since sadish's
per-row accumulator re-grows on the global heap (w*8, once) for any wider canvas. (3) Fonts are
opened outside a `dh_draw_text_ink` call (crab's fixture already is): `rekha_font_open` follows the
scoped hook and would die at the arena's first reset. ⛔ The gate that missed this is closed on
dhancha's side too:
`arena_test` still renders `font = 0`; `text_arena_test` renders the branch that was not running.
**Original status:** 🔴 **OPEN — a real defect in the `font != 0` path. MEASURED**, not read.
**Filed:** 2026-09-13, by **crab**.
**Affects:** dhancha **0.9.29** (and every version since the scalable path was written).
**Severity:** **High for any consumer that adopts a scalable face and has an allocation discipline.**
Latent for everyone today only because **nothing in the stack passes a non-zero font** — dhancha's own
`programs/setu_demo_client.cyr` is host-only, and every other caller passes `0`.

## What was found

`dh_draw_text_ink`'s scalable branch opens with:

```
var cv = sd_canvas_new(sd_surface_width(sds), sd_surface_height(sds));
```

A **full-surface canvas, per call** — that is per LABEL, per FRAME — plus `rekha_char_to_sdpath`
allocating a sadish path **per glyph**. All of it comes from the global bump allocator, which has no
`free()`, and **none of it goes through dhancha's own per-frame arena** (`dh_falloc`) — the one every
other draw path in the toolkit already uses, and the one that exists for precisely this.

The bitmap branch (`font == 0`) returns before reaching it, which is why nothing has noticed.

⚠ **The clipping the canvas is there for is right, and is not what is in question.** Its own comment
earns it — *"a glyph straddling the viewport edge is CUT, not dropped… Dropping the whole glyph was
the tempting shortcut and it reads as text that vanishes a row early while scrolling."* The defect is
**where the memory comes from**, not what it is for.

## Why crab is filing it

crab spent **four releases** (dhancha 0.9.13–0.9.15, crab 0.6.0) driving per-frame allocation from
**746,440 bytes to zero**, and it is a milestone of its own (crab M1.5). crab's headline assertion is
*"a rendered frame costs the global heap ZERO bytes."*

A crab pane at 760×300 draws on the order of a dozen labels per frame. At a full-surface canvas each,
one frame would allocate megabytes that are never returned, and a session would exhaust the heap
rather than merely slow down. ⇒ **crab cannot pass a real face until this is fixed**, and it is one of
two blockers on crab's `0.9.0 · A real face`.

## ⛔⛆ And the gate that should have caught it was measuring the other branch

crab's zero-allocation test renders **twenty frames and asserts the heap cost is exactly 0** — and
every one of those renders passes `font = 0`. The scalable path is a different function body, so the
gate was proving a branch that was not running. *A gate that covers one state proves one state.*

⭐ **crab 0.8.10 closed that blind spot from its own side** and is what makes this filing a measurement
rather than a code read. crab's suite now builds a **synthetic proportional face** in memory
(head/maxp/hhea/hmtx/cmap — no `glyf`, because a width question never rasterises), renders one frame
with it, and asserts the heap cost is non-zero.

⚠ **That assertion carries its own expiry, stated at the assertion**: it documents a defect, and **the
day this is fixed it will FAIL and must be inverted.** That is deliberate — it is how a blocker in
another repo gets a gate rather than a sentence in a document that goes stale. So dhancha will get a
signal from crab's suite when the fix lands.

## What would close it

Route the canvas and the per-glyph paths through `dh_falloc`, so the scalable path costs what the
bitmap path costs: nothing that survives the frame.

⚠ **crab will not work around it.** A consumer-side workaround means reaching into the toolkit's draw
path, which is dhancha's job by construction — the same reasoning as the retained-tree filing of
2026-08-31.

## Reproducing

crab's suite, `tests/crab.tcyr`, allocation group — `t_prop_face` builds the face, then one
`crab_render` with it against `alloc_used()` before and after. Nothing else in the stack exercises
the branch.

## Related

- `agnos/docs/development/issues/2026-09-13-no-proportional-face-on-the-target.md` — the **other**
  blocker on the same crab roadmap item, owned by agnos. They are independent: a face arriving
  without this fix ships a leak; this fix landing without a face changes nothing observable.
- `2026-08-31-immediate-mode-consumers-and-the-retained-tree-assumption.md` — the previous crab
  filing here, and the same shape: a dhancha feature that assumes something crab's arena discipline
  cannot give it.
