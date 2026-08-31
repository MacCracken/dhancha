# Changelog

All notable changes to dhancha are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [0.9.22] - 2026-08-31 — PROGRESS: a bar that can admit it does not know

### Added — `DhWidgetKind.PROGRESS` and `src/progress.cyr`

Three public functions, and that is the whole surface:

| | |
|---|---|
| `dh_progress_new(thickness)` | a bar `thickness` px tall, starting **indeterminate** |
| `dh_progress_set(bar, num, den)` | set fullness from an integer pair; **`den <= 0` = indeterminate** |
| `dh_progress_permille(bar)` | 0..1000, or -1 = indeterminate |

⭐ **Why a kind and not a BOX with a coloured child.** A bar *can* be built from existing parts — a
`BOX` with `bg = line` holding two flex children weighted `done` and `remaining` lands on exactly the
right pixel, because the flex distributor already hands its **last** share the exact remainder. That
was built out and rejected: the **app** would then be naming `dh_theme_accent()` itself, which crab's
ADR 0001 forbids outright, and every app that hand-rolls a bar picks its own track colour — the same
three-implementations-of-one-thing that made `LIST` a kind. A kind also costs one widget per bar
instead of three, and it is the only form that can express indeterminate at all.

⛔ **INDETERMINATE IS NOT A CONVENIENCE.** Callers really do have no denominator: crab's size array
carries **-2 (pending)** and **-1 (unstattable)** as distinct first-class values and refuses to
conflate them; a rename moves zero bytes; a recursive delete has no byte total at all, only a count
discovered while walking. A determinate-only bar forces those callers to invent a denominator — and
a bar reading 40 % because someone guessed is strictly worse than one that admits it does not know.
It paints in **`held`**, rupa's own token for "pending / sandbox / held", so the bar says *pending*
in the palette's vocabulary rather than in a colour dhancha invented. ⭐ That also puts `held` to
work: rupa published it and 0.9.20 bound it with nothing in the toolkit using it.

⚠ **Permille, not percent, and multiply before dividing.** Percent is one step per 3 px on a 300 px
bar, visibly steppy; and `num * (1000 / den)` written the other way round truncates to **zero** for
every value below `den`, so a bar would sit at 0 until the final chunk. `num` is clamped to `den`
first, so the product cannot overflow below nine petabytes.

⚠ **Out-of-range is CLAMPED and REPORTED, which is the opposite of `dh_list_select`.** Refusing is
right for a selection — naming row 12 of a 4-row list is a bug, and quietly handing back row 3 hides
it behind a plausible highlight. A transfer is not that case: a size stat taken before a copy can
legitimately disagree with the bytes written, and a bar frozen at its last value while the operation
is visibly running is worse than one pinned at full. So the **display never lies** and the **caller
is still told** (`DHANCHA_ERR_BAD_ARG`).

⚠ **The bar carries no text, deliberately.** One centred "68 %" straddling the fill edge cannot be
legible in a single ink — `on-accent` over the track and plain `ink` over the fill both fall near
**1.3:1** on the dark grounds. Put the percentage in a sibling label. A text set on a bar is still
drawn, on top of it, and inherits that problem.

### Changed — `DH_WIDGET_SIZE` 256 → 264 (`DH_W_FILL` at +256)

⛔ **This changes the per-widget footprint for every consumer.** A consumer with a **fixed-size**
arena sized against the old footprint now spills to the global allocator — and `dh_falloc` degrades
to a **leak**, never to a null, so it passes every functional test and surfaces only in an
`alloc_used()` convergence check. **crab is safe** (`arena_new_growable`, which chains); check any
other consumer before adopting. Cost to crab: ~236 widgets × 8 B ≈ **1,888 B** of additional arena
high-water mark per frame (+3.1 %) — rewound memory, not a leak.

⛔ **Seeded to -1, not 0**, for the same reason as `DH_W_SEL`: 0 is a *valid* fill, so a zeroed field
would make every bar claim a real 0 % rather than admit it had not been told. Under the frame arena
the memory is **recycled**, so an unseeded field carries a plausible stale permille, not an obvious
zero.

⚠ **The struct-layout header comment said "144 bytes, 18 u64 slots"** while the field list below it
already ran past +168 — it contradicted itself and understated the record by 112 bytes, which is
exactly how a reader sizing a new field picks a colliding offset. Corrected, along with the
per-frame figure in `widget.cyr` and `programs/arena_test.cyr` (248 B → 264 B, already one field
stale before this change).

### Verified — `programs/progress_test.cyr`, and ten mutations that must break it

The pixel checks count rather than probe, so no glyph bitmap has to be right for them to be exact.

1. clamp removed · 2. `w * (p / 1000)` instead of `w * p / 1000` · 3. `DH_W_FILL` seeded to 0 ·
4. indeterminate collapsed to an empty track · 5. `dh_theme_accent()` replaced with the mockup's
literal · 6. the draw arm moved after the text draw · 7. `dh_fill_rect_clip` → `sd_fill_rect` ·
8. `PROGRESS` made focusable · 9. `dh_falloc` → `alloc` · 10. `DH_WIDGET_SIZE` left at 256.
**Each one fails the suite.**

⭐ **The theme check is the 0.9.20 scar made a gate.** A hardcoded colour lifted from a mockup passes
every other pixel check, because they all read `dh_theme_*` too. Only re-rendering under a second
palette catches it, so the suite renders at `mudra-dark` and again at `shanta-light` and asserts the
fill followed — plus that the two palettes' accents actually differ, or the check proves nothing.

### ⛔ Not asserted, and said plainly rather than implied

- **Fill/track legibility.** A 3:1 floor on `accent`/`line` would **fail against the shipped
  palette** on both light grounds (~2.37 and ~1.88 by rupa's own approximation, which reads high).
  Asserting it here would be asserting a bug. The suite asserts token **identity** — the fill *is*
  `accent`, the track *is* `line` — and the number is recorded here as an upstream **rupa** ask.
- **"The bar carries no `on-accent` pixels."** Unwritable in either direction: `on_accent == bg` on
  both dark grounds and `on_accent == ink` on both light ones, so the scan finds background on dark
  and the sibling label's glyphs on light. Same class as the trap crab's `render_test.cyr` records.
- **Determinate vs indeterminate as a *contrast* claim.** `accent`/`held` measures 1.34–1.55: the two
  states differ by **hue**, not luminance, and rupa has no hue metric. Pixel identity carries it.
- **The `DH_WIDGET_SIZE` bump is only weakly covered.** With the size left at 256 the write lands on
  the next widget's `DH_W_ID` — silent corruption in a bump allocator with no guard pages, not a
  fault. The literal pin and the two-bar check catch the gross cases; neither is a memory-safety
  proof.
- **How it looks on a real display.** Nothing here blits to a framebuffer.

### Reported by / for

crab's M4 **transfer tray**. ⛔ **And the widget was never the real gate** — crab's roadmap records
the tray as "gated on a dhancha PROGRESS widget", which sends the work to the wrong repo. crab's
`crab_fs_copy` runs its whole read/write loop **synchronously inside the keypress branch**, so the
event loop draws no frames while it executes: a bar dropped into crab today renders once at 0 %,
never repaints, and vanishes. The real work is crab-side — stepping the copy off the idle tick that
already drives `crab_stat_batch`. This widget is necessary and not sufficient.

## [0.9.21] - 2026-08-30 — drag stops half-working under a frame arena

### Fixed — `DRAG_START` fired where `DRAG_END` never could

⛔⛔ **dhancha's drag API and its frame-arena API are mutually exclusive, and until now the failure
was silent and one-sided.** A drag is inherently multi-frame — press, one or more moves, release —
and its source is held as a raw widget pointer in `_dh_drag_src`. `dh_frame_begin` clears that
pointer on **every** frame, and it must: after `arena_reset` it addresses memory the arena is about
to hand out again. So with a frame arena installed, a drag begun on one frame had no source by the
next pointer event.

The symptom was the worst possible shape: **`DRAG_START` was delivered exactly as documented, and
`DRAG_MOVE` / `DRAG_DROP` / `DRAG_END` never arrived.** An app that set a "dragging" flag on START
was never told the drag ended, and the first event arriving correctly is what made it hard to
diagnose — a capability that half-works reads as an app bug, not a toolkit one.

⇒ **`dh_drag_progress` now refuses to begin a drag it cannot finish.** Under a frame arena, a press
on a draggable widget is simply a **click**: no `DRAG_START`, and `ACTIVATE` still synthesizes on
release — the behaviour the widget had before it was marked draggable. Nothing half-fires.

### Added — `dh_drag_available()`

The capability, made callable: `1` when drags can complete, `0` when a frame arena is installed.
⚠ **This is a capability question, not a policy one.** The 0.9.15 note on `dh_frame_begin` already
said cross-frame widget identity and a per-frame arena are exclusive *by construction*; drag **is**
cross-frame widget identity, so it was always on the wrong side of that line. This makes the
consequence observable up front instead of leaving it to be discovered at the drop that never came.
That note now names drag explicitly rather than leaving the reader to derive it.

⚠ **A fuller fix was considered and deliberately not built.** Drag could survive a rewind if identity
were an app-supplied opaque payload re-resolved against the current tree each frame, rather than a
pointer. That is a real API and it has **no consumer**: `dh_frame_begin` is a no-op without an arena,
so the defect only reaches apps using both — and dhancha's own position is that you pick one.
Building a speculative API to serve nobody is how a toolkit grows surface it cannot test.

### Reported by

crab **0.7.1**, the first client to want both APIs — and the client the frame arena was built for
(0.9.13–0.9.15). crab's M4 *drag between panes* is what surfaced it; crab already tracks its own
`(pane, row)` model rather than widget pointers, for this same reason, under a standing ruling from
2026-08-27 that it does not use `dh_dispatch` at all.

### Changed — toolchain pin 6.5.35 → 6.5.36

Matches crab, which moved at its own 0.7.1. `lib/` re-vendored by `cyrius lib sync`.

### Verified

All six `programs/*_test.cyr` pass on 6.5.36 — event (routing, hover, activate, tab, capture,
drag-drop, clip, refusal, motion), layout, list, draw, canvas, arena.
⭐ **The new behaviour is mutation-proven**: removing the guard, and making `dh_drag_available`
ignore the arena, each fail `event_test`.
⛔ **`cyrius distlib` REGENERATES `dist/dhancha.deps` WRONGLY AND IT BREAKS EVERY CONSUMER.** Raw
distlib emits `kashi_font_data` as a stdlib leaf; it is a **vendored** module, so `cyrius deps` then
fails in every downstream repo with *"dep dhancha requires 'kashi_font_data' but it is not in the
cyrius stdlib"*. crab's build broke exactly this way during this change. ⇒ **After any `distlib`,
run `sh scripts/sync-deps-sidecar.sh`** to restore the sidecar. The file's own header says so, and
setu carries the same defect in the other direction
(`docs/development/issues/2026-08-07-distlib-deps-sidecar-under-reports.md`).

⚠ The new sub-test pins `g_dmove` at **3, not 2** — sub-test Q above it drags over a non-target and
emits a `DRAG_MOVE` it never asserts, so the running total is already 3. Pinning the wrong baseline
there would have read as "the guard failed" when the guard was working.

## [0.9.20] - 2026-08-28 — real columns, and the selected row's text survives its own highlight

### Added — COLUMNS: a shared width spec so a header and its rows line up (`src/table.cyr`)

dhancha could already draw a row of cells — a `BOX_H` of fixed-width labels lays out fine on its own.
What it could **not** do is guarantee that the header and every row agree on where column 2 starts.
Rows laid out independently drift the moment one cell's content changes, and a table whose columns
drift is worse than no columns: the reader trusts the alignment to say which value belongs to which
heading.

- **`dh_cols_new(n)` / `dh_cols_set_width` / `dh_cols_width` / `dh_cols_count` /
  `dh_cols_fixed_total`** — one spec, read by the header and by every row. Up to 4 columns.
- **`dh_table_row(spec)` / `dh_table_cell(row, spec, i, text)` / `dh_table_header(spec)`** — rows are
  ordinary `BOX_H` widgets and cells ordinary labels; `table.cyr` owns no pixels, like `list.cyr`.
- **A 0-width column takes the remainder** (flex 1), so a caller need not know the container's pixel
  width at build time — which for a resizable pane it does not.
- **The header is drawn in the theme's secondary ink.** ⛔ A header that looks like a row is a bug,
  not a style choice: the first line of a file list reading as a file is the misreading a header
  exists to prevent.
- ⛔ **Fixed widths, not content-measured.** Measuring the widest cell needs every row's text before
  the first row can be placed — a second full pass over a directory that may hold thousands of
  entries — and it makes column geometry depend on what is on screen, so scrolling would shift the
  columns. Auto-sizing is a real feature; it is not this one.

### Added — per-widget text colour

- **`DH_W_FG` (offset 248, `DH_WIDGET_SIZE` 248 → 256)**, `dh_widget_set_fg` / `dh_widget_fg`.
  `-1` inherits, which is what every widget did unconditionally before. Appended, so every existing
  offset is unchanged.
- ⛔ **It does NOT outrank the selection highlight.** A cell's colour is a *preference*; on-accent on
  a selected row is a *legibility guarantee*. Without that precedence a table could set a column's
  colour and make the selected row unreadable again — the exact defect fixed below.

### Fixed — the highlight was erasing the row it highlighted

`dh_draw_list_selection` fills the focused selection with `accent`, and every row's label was then
drawn in the theme's primary `ink`. On MUDRA dark that is `0xE7E9EF` on `0x00E5FF` — a contrast ratio
of **1.27:1**. The one row the operator is looking at was the one row that could not be read.

- **`dh_theme_on_accent()`** — binds rupa 0.1.5's new `on-accent` token (the ink to use *on* an
  accent fill).
- **`dh_draw_widget_ink` / `dh_draw_text_ink`** carry a text colour down the widget tree; the focused
  list's selected row gets `on-accent`, and it propagates to that row's own children (a row is not
  always a leaf).
- ⚠ **`dh_draw_widget` and `dh_draw_text` keep their existing signatures** and default to
  `dh_theme_ink()`. puka calls `dh_draw_widget` directly, as do dhancha's test programs.
- ⚠ **Gated on focus, exactly as the fill is.** An unfocused list fills its selection with `line`
  instead, where normal ink reads fine — so the two cannot drift apart.

### Fixed — the scalable-font path ignored the theme entirely

`dh_draw_text`'s rekha branch (`font != 0`) blitted **hardcoded `sd_rgb(255, 255, 255)`**. White
glyphs on MUDRA light's `0xFBF8F0` paper is white-on-white: that path was unreadable on both light
grounds and had been since it was written. The bitmap path (`font == 0`, what everything actually
ships with) always honoured the theme, which is why nothing caught it. It now uses the same `ink`.

### Added — the theme binding is complete

- **`dh_theme_faint()`** and **`dh_theme_held()`**. rupa published both and dhancha never bound
  them, so a consumer wanting either had to reach past the toolkit into rupa directly — the coupling
  `theme.cyr` exists to prevent.

### Changed

- **`[deps.rupa]` 0.1.4 → 0.1.5**.

### Testing

- `programs/list_test.cyr` gains a pixel test: the focused+selected row's glyphs are `on-accent` and
  **none** are plain ink, while an unselected row in the same render is the reverse. ⛔ That control
  row is what makes the assertion mean something — a build that drew *every* label in on-accent
  would pass without it.
- ⚠ The oracle **counts** glyph pixels rather than probing one coordinate, so it does not depend on
  the exact bitmap of a CP437 glyph.
- ⚠ Expected values are bare `0xRRGGBB`: glyph pixels really are stored with alpha `0xFF`, but
  `sd_surface_pixel_at` rebuilds its answer from b/g/r and drops the alpha byte.
- Mutation-proven five ways: the swap reverted (2 failed checks), applied to every row (4), not
  gated on focus (2), `dh_draw_text` ignoring its ink parameter (2), and `on_accent` bound to rupa's
  `ink` token (4).
- **`programs/table_test.cyr`** — a new RUN suite. ⭐ The claim under test is **alignment**, so the
  checks are on laid-out **bounds**, not on the spec's accessors: a spec that stored the right
  numbers and laid out wrongly would pass an accessor-only test and still be useless. Two rows of
  very different content length must put SIZE and MODIFIED at the same x as the header.
- Mutation-proven four ways: cells ignoring the spec width (5 failed checks), the header not muted
  (1), a cell's own fg defeating the selection highlight (2), and out-of-range width writes accepted
  (1).
- All 12 RUN suites pass; puka builds unchanged against the new toolkit.

## [0.9.18] - 2026-08-27 — `POINTER_SCROLL`: the wheel reaches apps

### Added

- **`POINTER_SCROLL` (`DhEventKind` 14)** — `a` = signed wheel delta, **positive = wheel-up** (away
  from the user). Mapped from setu's `SETU_INPUT_PTR_SCROLL` (kind 12, setu 0.8.8).

⛔ **NOT a `POINTER_BTN` with a magic button code.** X11 spends buttons 4/5 on wheel detents; copying
that would make `POINTER_BTN`'s `a` mean two different things and turn a scroll into a click in every
consumer that range-checks buttons rather than enumerating them.

⚠ **No position, deliberately.** setu carries id + delta only. The pointer is wherever the last
`POINTER_MOVE` left it, and `dh_dispatch` already tracks that in `_dh_ptr_x`/`_dh_ptr_y` — the same
reason `POINTER_BTN` carries no coordinates either.

⚠ **The whole chain has to be present for this to fire**: agnos >= 1.56.49 reads the wheel byte,
bhumi >= 1.4.3 carries it, setu >= 0.8.8 defines the kind, and the compositor must send it.

### Changed

- **`[deps.setu]` gains `path = "../setu"`.** Without it dhancha can only build against a published
  setu tag, so a new protocol kind cannot be mapped until it is pushed. ⛔ `path` WINS over `tag`, so
  a green local build is not evidence the declared graph resolves — re-verify against setu's VERSION
  at every cut.

### Testing

`poll_test` gains 7 checks: the kind maps, the sign survives (a wheel-down arriving as a large
positive would scroll a list to its end), it is not confusable with `POINTER_BTN`, and it does not
displace input already queued ahead of it.

## [0.9.17] - 2026-08-27 — `dh_surface_resize`: the entry point `WINDOW_CONFIGURE` has waited five releases for

### Added — `dh_surface_resize(surf, w, h)`

`SETU_CONFIGURE` has reached apps as `WINDOW_CONFIGURE` since **0.9.12**, and **no client could act on
it**: a `DhSurface`'s `w`/`h` were fixed at construction with no way to change them. A client that
ignores the ask does not crash — it **freezes at its old extent** while the compositor clamps or
refuses every blit. This is the missing half.

Returns `DHANCHA_OK`, or `DHANCHA_ERR_NO_SURFACE` / the new `DHANCHA_ERR_BAD_ARG` for a degenerate
extent. ⚠ Resizing to the size the surface already has is a **no-op** — a compositor is free to
re-send a CONFIGURE with the current size, and treating that as a change would discard a render target
and a pixel buffer for nothing, every time.

### Fixed — a latent buffer overflow that 0.9.13 created and nothing could reach until now

`dh_surface_pixels` allocates `w*h*4` **on first ask** and caches it (0.9.13). With `w`/`h` immutable
that was safe by construction. The moment a surface can grow, a cached buffer sized for the **old**
extent would be handed to a caller writing the **new** one — a silent heap overflow into whatever the
bump allocator placed after it, in an allocator with no `free()`.

⇒ `dh_surface_resize` **drops the pixel cache**; the next `dh_surface_pixels` re-allocates at the new
size. The old buffer is abandoned rather than freed — there is nothing to free it with — so a resize
costs one screenful once, which is the right trade against handing back a short one.

⚠ **This is the shape of hazard a lazy cache creates: it was correct when written and became wrong
because something else became possible.** It was found by asking what resize would touch, not by a
failure.

### Changed — 0.9.14's dimension check is now LIVE

`dh_surface_render` has compared its cached target's dimensions against the surface's since 0.9.14.
That branch was **unreachable**, documented as such, and *measured* as such — deleting it did not fail
`draw_test`. Its ⚠ said "when `dh_surface_resize` lands, it makes this branch live — add the test
then." It landed, and `draw_test` now pins it.

⇒ The render target is deliberately **not** dropped by `dh_surface_resize`: the existing check is the
mechanism, rather than a second invalidation site that can drift from it.

### Added — `DHANCHA_ERR_BAD_ARG` (6)

A well-formed call carrying an out-of-range value. Folding a degenerate extent into `_NO_SURFACE`
would tell a caller to check its handle when the handle was fine.

### Testing

`draw_test` gains 20 checks: the resize itself, the target re-made at the new extent **and still
reused after it**, the dropped pixel cache, every refusal, and the same-extent no-op.

⭐ **Mutation-verified, four ways**, each failing: dropping the pixel-cache invalidation (1 check);
deleting the now-live dimension check (3); removing the same-extent no-op (1); accepting a degenerate
size (5).

## [0.9.16] - 2026-08-27 — an idle poll allocates nothing

### Fixed — `dh_setu_poll_event` allocated 80 B before it knew whether anything was pending

It opened with `setu_msg_new()` — an 80-byte `alloc` — and only then asked the client whether a frame
was there. `lib/alloc.cyr` has **no `free()`**, so an idle desktop grew the heap once per wakeup
forever, and a client repainting without input grew it at the repaint rate: **~4.8 KB/s at 60 Hz**,
permanently.

⭐ **This was the last unbounded per-cycle allocation in the render/input loop.** 0.9.13–0.9.15 took a
rendered frame to zero bytes; this is the other half. With both, the loop allocates **nothing** in
steady state — which is what makes a self-repainting element (an idle animation, a transfer progress
bar, an index counter) affordable rather than a slow leak.

The message is pure scratch: `setu_client_poll_input` fills it and `dh_setu_map_input` reads it to
build a **separate** `DhEvent`, so one per process is enough. `dh_setu_msg_scratch()` hands out that
buffer, zeroed. `dh_setu_read_event` uses it too.

⚠ **NOT on the frame arena, and that is the trap worth naming.** Routing it through `dh_falloc` would
be wrong: polling happens in the event loop, `dh_frame_begin` rewinds the arena inside the caller's
render, and the scratch would be freed out from under a loop still using it. Per-frame and per-poll
are different lifetimes.

⚠ An **event** still allocates — `dh_event_new` is 56 B per real event, which is per-input, not
per-cycle. Only the idle path is free.

### Testing

`poll_test` gains 8 checks: 200 idle polls moving the global heap by **exactly 0 bytes**, a
non-vacuity arm proving a client with frames still costs something, and the CONFIGURE→CLOSE sequence
across the shared buffer.

⚠ **Warm-up is load-bearing in that measurement** — the scratch and the client's inbuf are both
allocated on first use, so measuring from a cold client reports their one-time cost as a per-poll one.

⛔ **The zeroing of the reused scratch is UNEXERCISED DEFENCE, and the test says so rather than
implying coverage.** Removing it leaves `poll_test` green: `dh_setu_map_input` maps `SETU_CLOSE` with
a literal `a = 0`, and every kind that reads an arg has it guaranteed by `setu_decode`'s argc-vs-kind
check. Measured, not assumed. It is kept because it removes a dependency on that validation staying
correct, for ten stores.

## [0.9.15] - 2026-08-27 — a per-frame arena: an immediate-mode toolkit that stops leaking a tree a frame

### Added — `dh_frame_arena_set` / `dh_frame_begin` / `dh_falloc`

dhancha is immediate-mode: an app rebuilds its widget tree every frame. The allocator underneath
(`lib/alloc.cyr`) is a bump allocator with **no `free()`**, and `alloc_reset` rewinds the *whole*
arena — taking the caller's long-lived objects with it — so until now **every frame's tree was
retained for the life of the process**. Measured against crab at 380x220 with 114 entries per pane:
**~236 widgets x 248 B = 58,528 B per frame**, permanently, plus layout's measure scratch.

An app now supplies an arena and dhancha routes its per-frame allocations onto it:

```
var a = arena_new_growable(262144);
dh_frame_arena_set(a);
while (running) {
    dh_frame_begin();          # rewind the arena AND drop dhancha's retained widget pointers
    ... build the tree, lay out, render ...
}
```

⭐ **The gate is CONVERGENCE, not "it uses an arena".** `arena_reset` on a GROW arena rewinds to the
first chunk and **keeps the chain**, so a render loop settles at its high-water mark and then costs
the global allocator **nothing at all**. `programs/arena_test.cyr` asserts exactly that: 20 identical
frames, then 10 larger ones, must each move `alloc_used()` by **0**.

⛔ **WHAT IS ROUTED IS A SAFETY BOUNDARY, NOT A COMPLETENESS ONE.** dhancha calls `alloc()` at 19
sites and only the per-frame ones moved — `dh_widget_new` and layout's measure scratch, both of which
die with the frame by construction. **Not** routed: `dh_surface_new` and its pixels, the setu client's
shared buffer, event records and queues, textinput text buffers, canvas surfaces, error records.
Every one of those outlives a frame and a caller holds pointers to them across resets. Routing
`dh_surface_new` in particular would free a consumer's session surface out from under it on the first
rewind — and the bump allocator would then hand that memory back out, so it would corrupt silently
rather than fault.

⛔ **`dh_frame_begin` DOES TWO THINGS AND THEY CANNOT BE SEPARATED.** `_dh_focus`, `_dh_hover`,
`_dh_press` and `_dh_drag_src` are raw widget pointers held across calls. After a rewind they address
memory the arena is about to hand out again, so `dh_focus_within` would walk recycled parent links and
answer confidently about a widget that no longer exists — with no fault to say so. ⇒ **Do not call
`arena_reset` on a frame arena directly.**
⚠ It follows that an app using a frame arena **must re-establish focus every frame**. Cross-frame
widget identity and a per-frame arena are mutually exclusive by construction, not by policy.

⚠ **Exhaustion degrades to a leak, never to a null deref.** `dh_falloc` falls back to the global
allocator when the arena refuses. The stdlib's own arena notes diagnose why that matters: a 0 from
`arena_alloc` is indistinguishable from a valid pointer and faults several layers away, so adopting
the feature would otherwise make the failure *worse and quieter*. Prefer `arena_new_growable`, which
chains a chunk and never reaches the fallback.

⭐ **Opt-in, and the default is byte-for-byte the old behaviour.** With no arena set, `dh_falloc` is
`alloc` and `dh_frame_begin` is a no-op.

### Testing

New `programs/arena_test.cyr` — 20 checks across six groups: the unchanged default, widgets actually
landing in the arena, `dh_frame_begin` doing both halves, global-heap convergence over 20 frames and
again over 10 larger ones, and unsetting restoring the old behaviour exactly.

⭐ **Mutation-verified, five ways**, each failing: `dh_widget_new` back on `alloc`; `dh_frame_begin`
skipping `dh_reset_input`; `dh_frame_begin` skipping `arena_reset`; layout scratch back on `alloc`;
`dh_falloc` ignoring the arena.

⚠ The suite asserts the **pre-arena baseline too** — two arena-less frames must each cost the global
heap something. Without that, every saving below it would be measured against nothing.

## [0.9.14] - 2026-08-26 — `dh_surface_render` reuses its render target

### Changed — the returned `SdSurface` is owned by the `DhSurface` and reused across calls

⛔ **THIS IS A CONTRACT CHANGE.** Until 0.9.13 every `dh_surface_render` allocated a brand-new
`sd_surface_new(w, h)`, so two renders of the same `DhSurface` produced two independent images. They
are now **the same object**: a caller holding an earlier render sees it overwritten by the next one.
A caller that genuinely wants two images at once must render into **two `DhSurface`s** — which is
strictly clearer than the old behaviour, where the aliasing question was invisible because the answer
was always "no", and always cost a full-size buffer to say so.

⚠ **Why it is worth a contract change.** `lib/alloc.cyr` is a bump allocator with **no `free()`**, so
the per-call allocation was never transient — it was retained for the life of the process, every
frame, forever. Measured against crab at 380x220: **334,432 B per frame**, which after 0.9.13's fix
was **81 % of the entire frame cost**. An immediate-mode toolkit that cannot repaint without leaking a
screenful cannot host an animation, a progress bar or a clock.

`DhSurface` grows **40 → 48 bytes** for the cached target (`DH_S_SDS` at +40). Reuse is
pixel-identical because `sd_clear` fills every pixel of the surface before the tree is drawn; if that
ever stops being true, reuse stops being safe, and `draw_test` now says so.

### Fixed — `dh_surface_render` dereferenced a failed allocation

The old body passed `sd_surface_new`'s result straight into `sd_clear` with no check, so an OOM there
dereferenced 0 instead of returning it. It now returns 0, which the documented contract already
promised for a bad surface.

### Testing

`draw_test` gains 10 checks pinning both halves of the new contract: the **identity** (a second render
returns the same object; two `DhSurface`s still return two) and the **equivalence** (a frame rendered,
overwritten, and rendered again is pixel-identical, with no residue).

⛔ **The first draft of the residue check could not fail, and mutation testing is what caught it.**
Every tree it rendered painted its root across the full surface, so deleting `sd_clear` outright left
the suite green. The check now renders a root that paints nothing (`bg = -1`) above a short child and
reads a pixel below it, where residue from the previous frame is the only thing that could appear.
Mutation-verified: reverting to per-call allocation fails, deleting `sd_clear` fails, and sharing one
target across all surfaces fails.

⚠ **One branch is unexercised and is documented as such**: the cached target's dimension check cannot
be reached, because there is no `dh_surface_resize` and a `DhSurface`'s `w`/`h` are immutable after
construction. Deleting it does **not** fail the suite — measured, not assumed. It is kept so that the
entry point `WINDOW_CONFIGURE` implies cannot land without the cache already being correct.

## [0.9.13] - 2026-08-26 — a DhSurface no longer carries a pixel buffer nothing reads

### Fixed — `dh_surface_new` allocated a full-size pixel buffer that was never written and never read

`dh_surface_new` did `alloc(40)` for the struct and then `alloc(w * h * 4)` for a pixel buffer stored
at `DH_S_PIXELS`. **`dh_surface_render` ignores that buffer entirely** — it allocates its own
`sd_surface_new(w, h)` and draws the widget tree into that. Nothing in dhancha, none of its
`programs/`, and no consumer in the stack ever wrote or read the field.

⛔ **The allocator underneath has no `free()`.** `lib/alloc.cyr` is a chunk-based bump allocator;
`alloc_reset` rewinds the *whole* arena, so a consumer that builds a surface per frame cannot reclaim
any of it. Every byte a render touches is retained for the life of the process.

⭐ **Measured, against crab 0.5.0 at 380x220 with 114 entries per pane** (the real iron count for `/`,
via a host probe on the production `crab_render`):

| | per frame |
|---|---:|
| before | **746,440 B** |
| after | **412,040 B** |
| saved | **334,400 B — 44.8 %** |

`dh_surface_new(380, 220)` itself goes from **334,440 B to 40 B**.

⚠ **THE FIELD IS DEFERRED, NOT DELETED, AND THAT IS NOT FUSSINESS.** `surface.cyr` ships in
`dist/dhancha.cyr`, so an external caller may hold `dh_surface_pixels`. The accessor now allocates on
first call and caches, so such a caller still receives an owned `w*h*4` buffer and cannot tell the
difference; every caller that does not ask — which is all of dhancha, all of its programs, puka and
crab — pays nothing.

⛔ **And it is NOT the one-line change it looks like.** Simply storing 0 breaks `event_test` S):
`dh_surface_present` returns `DHANCHA_ERR_NO_SURFACE` when pixels are 0 and `DHANCHA_ERR_UNSUPPORTED`
otherwise, so a permanently-zero field silently downgrades 0.9.5's "refuse loudly and diagnosably"
contract to "bad surface". **Mutation-verified**: the naive variant fails `event_test` with exit 1.

⚠ **This is the first of three steps.** The other pixel buffer — `dh_surface_render`'s per-call
`sd_surface_new`, another 334,432 B per frame — and the ~236 widget records per frame are still
unreclaimed. Reusing the render target needs a caller that keeps its `DhSurface` alive across frames,
and crab builds a fresh one inside `crab_render`; that is a separate change on both sides. See crab
`docs/architecture/001-every-frame-allocates-and-nothing-is-freed.md`.

### Testing

All 10 `programs/*_test.cyr` suites pass unchanged (`event_test` included). Mutation-tested: replacing
the deferred accessor with the plain `load64` one fails `event_test`.

## [0.9.12] - 2026-08-18 — WINDOW_CONFIGURE: the compositor may ask a client to resize

### Added — `SETU_CONFIGURE` reaches apps as a `DhEvent`

`SETU_CONFIGURE` (S->C: id, w, h, state) has been in setu's protocol since it was written, with a
constructor and **zero senders and zero handlers on either side**. aethersafha 0.16.8 now sends it;
this maps it to `WINDOW_CONFIGURE` (`a` = width, `b` = height) so a client can act on it.

⛔ **It is an ASK, not a fact.** The compositor cannot resize a client's buffer — the buffer is the
client's, and on agnos a `#86` slot it owns. Until the client re-attaches at the requested size the
compositor clamps or refuses its blit, so a client that ignores this **freezes at its old extent**.
Before 0.16.8 there was neither ask nor clamp: the blit read `win_w x win_h` from a smaller slot and
took a page fault on iron.

⚠ Like `WINDOW_CLOSE`, it survives the non-input drain in `dh_setu_poll_event` — a poll that consumed
it silently would leave the client frozen with no way to learn why.

### Testing

`poll_test` 26 checks. Mutation-tested: unmapping `SETU_CONFIGURE` fails 5.

## [0.9.11] - 2026-08-17 — the toolkit offers both event shapes

### Added — `dh_client_poll_event`, the NON-BLOCKING half

⛔ **Until now `dh_client_next_event` was the only event API and it BLOCKS.** That suits an app whose
display changes only when input arrives. It stalls one that redraws on its own — a file manager
repainting a selection, a terminal draining a pty — so every such consumer reached past the toolkit
to setu. That is precisely how crab ended up with a hand-rolled input loop while using dhancha for
pixels. Operator ruling: *a toolkit should present both styles for downstream users.*

⚠ **It wraps `setu_client_poll_input`, not `setu_poll_input` — a bug fix, not a detail.** setu's own
header says the latter "decodes only the FIRST frame of each recv; coalesced or split frames are
dropped ... kept for API compat", and a dropped tail **loses key-RELEASE events, which sticks keys**.
Any consumer moving off a hand-rolled `setu_poll_input` loop gains that fix by adopting.

⚠ A non-input frame is consumed and the poll CONTINUES rather than returning 0 — returning 0 there
reads as "nothing pending" and stops a caller's drain loop early, stranding buffered frames behind it.

### Fixed — the client layer was missing from `src/lib.cyr`

`setu_client.cyr`, `setu_input.cyr` and `dh_client.cyr` were in `[lib] modules` (the dist fold) and
**not** in the build include chain, so nothing built from `src/lib.cyr` could call `dh_client_*`.
dhancha could not test its own connection surface, and only consumers vendoring `dist/dhancha.cyr`
had it. Same shape as the 0.9.4 defect with the halves reversed.

### Testing

`programs/poll_test.cyr` (14 checks) — no socket needed: `setu_client_poll_input` drains the client's
reassembly buffer before any syscall, so seeding that buffer exercises the real path on a host.
Mutation-tested: stopping the drain on a non-input frame, and wrapping the frame-dropping poll.

⚠ Event accessors in the test are guarded — an unguarded deref made both mutants SIGSEGV before the
suite could report *which* check failed. A crash is a signal; a failure count is a diagnosis.

## [0.9.10] - 2026-08-17 — toolchain pin to 6.5.27

### Changed — `cyrius = "6.5.21"` -> **6.5.27**

Stack-wide sweep so every repo in the desktop stack declares one toolchain. Pins had drifted across
three lines (6.5.5 / 6.5.20 / 6.5.21) while the installed wrapper was 6.5.27, so every build ran with
a drift warning and the declared graph did not describe what was actually compiled.

⚠ **Measured byte-identical**: 6.5.21 and 6.5.27 produce the same artifact for this repo, so the bump carries no codegen risk here. Recorded because a pin that is assumed to be cosmetic is how a real change gets waved through later.

⚠ The vendored `lib/` was re-synced to the 6.5.27 bundled set, which clears the
`./lib/ shadows version-pinned` warning. Tests re-run green after both changes.

## [0.9.9] - 2026-08-17 — CANVAS: the slot an app paints itself

### Added — the "text surface" the roadmap asked for, generalised

M7-A named two containers a desktop needs: *"a file manager needs a LIST/scroll container; a terminal
needs a text surface"*. 0.9.7 shipped the first and not the second, and puka's port is where that
showed: **not every surface decomposes into widgets.** An 80×24 terminal is 1920 cells, each with a
foreground, background, attributes, a wide-glyph spacer flag and a possible cursor. As widgets that
costs more memory than the terminal's scrollback and discards the **dirty-row scanner** that makes a
keystroke echo repaint sixteen scanlines instead of the screen.

`CANVAS` is the answer every real toolkit reaches: give the app a rectangle and let it paint.
`dh_canvas_new(draw_fn, userdata)`; the callback receives `(wgt, sds, clx, cly, clw, clh)`.

⛔ **AND IT IS DELIBERATELY NARROW.** A canvas gets its box and its clip and nothing else — no access
to siblings, no way to widen the clip it was handed. An escape hatch that can reach outside its
rectangle is not a hatch but a hole; the whole value of putting app-drawn content in the tree is that
layout, clipping and hit-testing keep applying to it.

⚠ A canvas entirely outside its clip is **not called at all**, so an expensive renderer is skipped
rather than asked to draw nothing.

### Added — `dh_surface_wrap`: draw into memory the caller owns

An SdSurface header over caller-owned pixels. Nothing is copied, nothing is freed.

⭐ **This is the difference between a port and a regression.** An app with its own present buffer —
puka gets one from `win_present_begin` — would otherwise render the tree into a dhancha-allocated
surface and copy the whole thing across every frame. Adopting the toolkit would have made the app
measurably slower.

⚠ It uses sadish's own `SD_SURFACE_*` constants rather than hardcoded offsets, so a layout change
there is a rebuild here and not a silent mis-write into a caller's framebuffer.

### Fixed — ⛔ raw-pixel painters used WIDTH where they meant STRIDE

`sd_surface_new` makes width == stride, so every painter that conflated them was accidentally correct
— until `dh_surface_wrap` introduced surfaces where they differ. The bitmap-font blit and the RGB24
canvas blit now use `sd_surface_stride`.

⚠ **This one does not fail loudly.** A width-as-stride bug over a padded buffer shears the image
progressively, one row of slip per scanline. It is covered by a wrap test using a 40px image in a
64px-per-row buffer, and by a mutation.

### Added — `dh_canvas_blit_rgb24`, which sets the alpha byte

⛔⛔ **THE ALPHA IS THE POINT OF PUTTING THIS IN THE TOOLKIT.** An RGB24 source has no alpha, so the
obvious pack — `(r << 16) | (g << 8) | b` — leaves byte 3 at **zero**. Harmless while nothing reads
it; under agnos's `gpu_shader_op #92` op 0x01 (premultiplied src-over,
`out = src + dst * (1 - src_a)`) alpha 0 collapses the blend to `out = src + dst`, and the content
renders as an **additive over-bright ghost** rather than opaquely — much harder to notice than a black
rectangle. puka's present path had exactly this defect. This stack has now paid that bill three times
(kashi glyph text, the coverage-blend colour, and puka), which is why the conversion lives here.

⚠ **`sd_surface_pixel_at` documents itself as returning "0x00RRGGBB (alpha dropped)"**, so no test
written against it can ever catch a missing alpha byte. That is sadish's readback contract, not a bug
— but it does mean the one defect that only manifests on iron is invisible to the readback every other
test in this repo uses. `canvas_test` reads the raw 32-bit word instead.

### Testing

`programs/canvas_test.cyr` (37 checks). Mutation-tested: dropping the alpha byte, handing the callback
the un-intersected clip, a blit that ignores its clip, and width-used-as-stride are each caught.

## [0.9.8] - 2026-08-17 — the LIST paints its own selection

### Fixed — ⛔ 0.9.7 OWNED THE SELECTION AND DREW NONE OF IT

`DH_W_SEL` drove scrolling and nothing else. A consumer adopting LIST still had to set a background on
the selected row itself — which is the hand-rolling the container was added to end. **crab found this
the moment it ported**: its panes shed the truncation and the scroll arithmetic and kept
`if (selected) { bg = accent }` completely untouched. A container that owns the selection must own how
the selection LOOKS, or it has taken only half the job.

`dh_draw_list_selection` now paints the selected row: **accent when the list holds focus, the muted
line colour when it does not**.

⛔ **THAT DISTINCTION IS THE POINT, NOT DECORATION.** Two panes both showing an accented row cannot
answer *"which one do my arrow keys drive"* — the single question a two-pane file manager exists to
make obvious. crab, puka and aethersafha's launcher had each written this rule separately.

⚠ Drawn BEFORE the rows, so a row that keeps its own background paints over the highlight. That is
deliberate — a row wanting to stay opaque when selected simply keeps its background — but it is also
the trap for a porting consumer, and crab carries a mutation check for it.

⚠ `dh_focus_within` matches the node **or any descendant**: a list containing a focused row is still
the active list, and a highlight dropping to muted the moment focus entered a row would flicker on
every interaction.

### Added — a LIST takes keyboard focus

`dh_widget_focusable` now accepts `LIST`. Arrow keys have to reach it and Tab has to be able to land on
it; a scrollable list the keyboard cannot enter is a mouse-only control, which is the same class of
failure as painted-but-inert seen from the input side.

### Testing

`list_test` grew to 65 checks. Mutation-tested: always-accent (loses the active-pane distinction),
an unclipped selection (paints past the viewport), and a `dh_focus_within` that ignores descendants
are each caught.

## [0.9.7] - 2026-08-17 — a scrolling list, an editable text field, and a painter that clips

### Fixed — ⛔ THE PAINTER IGNORED BOUNDS THE HIT-TEST HAS ALWAYS ENFORCED

`dh_hit_test` rejects a point outside a widget's rect at **every** level of its recursion, so a child
drawn outside its parent was already un-clickable. `dh_draw_widget` did not clip, so it was still
**visible**. That combination is worse than either half alone: pixels the operator can see that do not
respond, which reads as a working control until it is used.

`dh_draw_widget` now takes a clip box and intersects it with each node's own rect on the way down.
Rect fills clip arithmetically (exact — an axis-aligned rect clipped by one is another rect), borders
draw as four clipped edges rather than through `sd_rect`, the bitmap-font path folds the clip into its
per-pixel bound (so clipping costs nothing per pixel), and the scalable path pushes sadish's clip stack
(v0.4.0) so a glyph straddling the edge is **cut, not dropped**.

⛔ **A SUBTREE WITH NO VISIBLE PIXELS IS NOT VISITED AT ALL.** Culling before recursion is what makes a
10 000-row list cost its viewport rather than its content.

⚠ Applied to **every** container, not just the new LIST. Gating it on the scrolling kind would have left
the same draw/input disagreement everywhere else, which is how it survived this long.

⚠ `dh_widget_contains` is now the single definition of "inside", shared by the hit-test, the clip, and
`dh_list_index_at`. Three subtly different half-open conventions is how a 1px seam ends up dead.

### Added — LIST: the container three apps had already hand-rolled

crab's pane, puka's buffer, and aethersafha's launcher panel were three implementations of "rows, a
highlight, an offset", each with its own bottom-edge bug. `src/list.cyr` is one.

⭐ The scroll offset is applied **at layout time**, not at draw time, so a scrolled row's `x/y` are its
real on-screen coordinates. That is what lets hit-testing keep working with no knowledge of scrolling:
the pointer is compared against the same bounds the painter used. A draw-time-only offset would have
moved the pixels and left the clickable regions behind.

⛔ **`dh_list_max_scroll` clamps at 0.** Content shorter than the viewport yields a *negative* maximum,
and a scroll clamped to a negative maximum scrolls the content up off the top of a list that is not even
full. Every hand-rolled version in the stack had this.

⭐⭐ **`dh_list_scroll_to_sel` does nothing when the row is already visible.** The tempting
`scroll_to(sel * row_h)` snaps the selection to the top on every keypress even when it had not moved
out of view. The rule is: already visible → do not scroll; otherwise move by the least amount that
fits it.

⚠ Out-of-range `dh_list_select` is **refused**, while `dh_list_move_sel` **clamps** — a caller selecting
row 12 of a 4-row list has a bug and should not be handed row 3, but an operator holding Down does not.

⚠ Rows must be fixed-height: a flex row grows to fill the viewport, so a list of them never scrolls.
`dh_list_add` pins `flex = 0` rather than documenting it and hoping.

### Added — TEXTINPUT now actually takes input

Before this release `TEXTINPUT` was a kind and a focusability rule and nothing else. It could be built,
focused, Tab-cycled to and drawn — and could not be typed into, because no buffer existed for a
character to go into.

⭐ The buffer is `DH_W_TEXT`, the **same field a LABEL draws from**, so an edited field renders with no
new painting code and there is one answer to "what does this widget say" rather than a display string
and an edit string that drift.

⛔ **UTF-8 throughout, with a byte-offset caret that steps whole characters.** A caret that steps one
byte lands inside a multi-byte sequence, and the backspace after it leaves a half-encoded byte the
operator cannot delete — the caret can no longer find a boundary either side of it. Surrogates are
refused (they are not scalar values); a full buffer refuses a whole codepoint rather than writing part
of one.

⛔ **The edit step runs AFTER `dh_propagate` and only if the key was not already consumed.** A window
binding Ctrl+S would otherwise get its shortcut *and* type "s" into the focused field: the app sees a
working shortcut, the operator sees junk accumulating, and nothing connects the two.

⚠ The caret draws only on the **focused** field — a caret in every text box says every box is taking
input, which is the one thing the operator uses it to find out.

⚠ `DH_KEY_*` gained the named keys, at **puka's `InputKey` values verbatim** (0x110000+). This enum's
header already promised it is puka's sym space; a toolkit numbering Left as 3 while its input source
sends 0x110003 does not fail loudly, it silently edits text on arrow keys.

### Fixed — `[lib]` module list omitted the new modules

⛔ Same defect the client layer hit in 0.9.4, caught before release this time: `src/list.cyr` and
`src/textinput.cyr` were absent from `[lib] modules`, so `dist/dhancha.cyr` shipped none of the new
API and consumers would have linked against a bundle without it. `cyrius distlib` reports this as an
**undefined-function warning** (`dh_text_char_index`, from surface.cyr's caret call) rather than as an
ordering or omission error, which is a confusing way to learn it.

### Testing

`programs/list_test.cyr` (55 checks) and `programs/textinput_test.cyr` (80 checks), both mutation-tested
— byte-stepping caret, off-by-one capacity, accepted surrogates, swallowed Tab, dropped clip clamp,
negative max scroll, snap-to-top scrolling, unapplied scroll offset, caret drawn unfocused, and the
dispatcher not calling the editor are each caught by at least one check.

⚠ Two of those tests exist because the first version of the suite passed a mutation: the guard that an
app handler beats the field, and the end-to-end path proving the **dispatcher** calls the editor at all.
Every other check calls `dh_text_key` directly and would pass with the field unwired.

## [0.9.6] - 2026-08-17 — per-widget motion, and every override goes through the guards

### Added — a widget carries a motion ASK; rupa decides what it gets

⭐ Implements the consumer half of the operator's ruling: *"compositor grants the motion, apps can
override per-widget, but lets guard against any impossible or highly destructive behaviors."* The
vocabulary lives in **rupa 0.1.3** (`RU_MO_INSTANT/QUICK/CALM/BUSY`); a widget stores a role plus an
optional duration/easing override, and `dh_widget_motion_ms/_ease/_at` resolve it.

⛔ **THE ASK IS STORED, NOT THE ANSWER, AND THAT IS THE ACCESSIBILITY DECISION.** Resolving at set-time
would freeze the reduced-motion switch into every widget at the moment it was created — flipping the
switch later would leave already-built UI still moving. Resolving on READ means every widget follows it
immediately, which is the behaviour someone who turned it on actually needs.

⛔ **NO PATH RETURNS THE RAW ASK.** There is deliberately no `dh_widget_motion_want_ms` — a convenience
getter would be used by accident and would route around the flash-band floor and the reduced-motion
override. `dh_widget_motion_ms` is the only door to a number a caller can animate with, and it calls
rupa's clamp.

⚠ **The easing default is `-1`, not 0.** `RU_EASE_LINEAR` IS 0, so a zeroed field would silently mean
"this widget overrides the easing to linear" and every widget in the tree would lose its role's curve.

⚠ **`dh_widget_motion_at` routes by ROLE, not by the caller's choice**: periodic roles cycle, the rest
run once. A consumer picking `progress` for a BUSY widget would get a spinner that completes and stops,
which reads as the work having finished.

### Changed — `[deps.rupa]` 0.1.2 -> 0.1.3, and it gains a `path` override

Every other dep in this stack carries `git` + `path`; rupa did not, so a local rupa change could not be
built against until pushed — how a burn ends up testing last week's tokens. ⛔ `path` WINS over `tag`.

**Verification** — `event_test` sub-test **T**: defaults, an honoured override, a 10 Hz BUSY ask floored
out of the seizure band, a BUSY widget cycling rather than completing, and reduced-motion overriding a
widget that asked for 400 ms.

## [0.9.5] - 2026-08-17 — two GUI basics that were quietly wrong

### Fixed — `dh_surface_present` returned DHANCHA_OK without presenting anything

⛔⛔ **A SUCCESS CODE FOR WORK IT NEVER DID.** It validated the surface, cleared the dirty flag and
returned `DHANCHA_OK` while sending nothing to the compositor — so an app reaching for the
obviously-named entry point got a blank window and a success code to explain it away.

⭐ **It cannot present, and that is structural, not missing code:** presenting needs a CONNECTION and
the signature takes only a surface. The real entry point is `dh_client_present(c, surf, font)`. It now
returns **`DHANCHA_ERR_UNSUPPORTED`** — refusing diagnosably instead of lying — and no longer clears the
dirty flag, which had been asserting "this has been shown" of a surface that never reached a screen.
⚠ Refused rather than deleted: `surface.cyr` is in the published dist, so an external caller may exist.
Today it gets silence; now it gets a code it can act on.
⚠ Nothing in this repo called it, including dhancha's own programs — which is why the lie survived.

### Fixed — `dh_hit_test` ignored clipping: widgets answered clicks where they are not drawn

⛔ The walk recursed into children **even when the point was outside the parent**, so a child laid out
past its container — anything scrolled, overflowing or oversized — received input at coordinates where
`dh_surface_render` does not draw it. Render clipped; input did not; the two disagreed about where a
widget was. The failure looks like a dead click on the widget you CAN see.
⇒ A point outside a subtree's root is outside every descendant, so the walk now returns immediately —
the correctness fix and a free pruning. Sibling order was already correct (last painted wins).
⚠ This assumes children are clipped to parents, which is what box/flex containment means. A future
overlay/popup that must escape its parent needs its OWN root, not a relaxation here.

### Verification — and why the existing suite could not see either fix

⛔ **All five run-tests passed BEFORE and AFTER both changes**, because every existing sub-test places
children inside their parent and none calls `dh_surface_present`. A behaviour change that leaves the
suite green is one the suite cannot see. `event_test` gains sub-tests **R** (clipping, with a
non-vacuity check that the inside child is still reachable) and **S** (present refuses, and leaves the
surface dirty). **Mutation-tested: reverting both fixes fails event_test with 3 failures.**

## [0.9.4] - 2026-08-16 — the toolkit can reach the compositor on agnos again

### Fixed — `[deps.setu]` 0.7.4 -> **0.8.5**: dhancha had NO agnos transport at all

dhancha delegates its whole client path to setu — `dh_client_connect(path)` → `setu_client_connect(path)`,
`dh_setu_connect` → `setu_connect` — which is the right design and is why this is a dep bump rather than
new code. It pinned **setu 0.7.4**, and puka's changelog records what that means: *"0.7.4 predates the
channel-band cutover and has no agnos arm at all, so `setu_client_connect` returned 0 on agnos with none
of setu's own refusal messages — a silent failure."* Before the bump dhancha's vendored setu had **zero**
`CYRIUS_TARGET_AGNOS` arms; after it, **28**, plus the `AGNOS_CHAN` + `CH_CAPS` floor check.

⛔⛔ **SCOPE, CORRECTED IN DRAFT — THIS FIXES dhancha's OWN AGNOS PROGRAMS, NOT ITS CONSUMERS.**
The first version of this entry said the toolkit "was disconnected from the sovereign desktop" full stop.
**That is wrong, and the measurement that corrects it is one grep:** a consumer resolves its OWN
`[deps.setu]`, so `crab/lib/setu.cyr` is 0.8.5 with 28 agnos arms regardless of what dhancha pins — which
is exactly why crab has been presenting on iron all along. What 0.7.4 actually broke is **this repo's own
`programs/setu_*_probe.cyr` and demos on agnos**, i.e. the means of testing the toolkit against a real
compositor — and it is a landmine for any future consumer that trusts dhancha's declared graph.
⚠ Worth stating because the difference is the whole diagnosis: "the toolkit cannot reach the compositor"
and "the toolkit's own test programs cannot" are different defects with different urgency.

⚠ **HOW IT REGRESSED — a substrate moved and the library was left behind.** 0.7.0 shipped *"a real
dhancha app on the sovereign desktop"* over setu's TCP transport (0.6.3 adapted to it). That transport
was then retired as the wrong primitive for local display IPC and replaced by the agnos channel band
`#97`, where the compositor mints a channel and endows an end at spawn. setu came across; dhancha did
not, and the toolkit's silence looked exactly like "not wired up yet".

⛔ **THE CONSEQUENCE WAS AN ARCHITECTURE INVERSION.** With no working client, every windowed app on
agnos hand-rolled its own — `crab/src/main.cyr` carries 3 `CYRIUS_TARGET_AGNOS` arms and calls
`setu_client_connect` itself, using **no** `dh_client` and **no** `dh_surface_present`. It used dhancha
as a *drawing library only*. That is precisely what this repo's README says it exists to prevent.

### Changed — cyrius pin 6.5.5 -> **6.5.21**, matching agnos and aethersafha

One language version across the desktop stack. ⚠ `cycc` is already **6.5.23** locally; the pin tracks
the burn stack rather than the newest compiler, and moving the whole stack is a separate sweep.

Verified: `--agnos` build OK, and all six run-tests green — `draw_test`, `event_test`, `layout_test`,
`text_test`, `theme_test`, `font_render_test`.

## [0.9.3] - 2026-08-02

### Changed — cyrius pin 6.4.71 -> 6.5.5; sadish 0.5.1, rupa 0.1.2, rekha 0.3.4, kashi 1.0.4, setu 0.7.2

Part of the whole-desktop-stack toolchain catch-up cut on this date, so the next burn runs binaries
built by ONE compiler. ⚠ The pin was documentation, not enforcement — `cyrius build` uses the
INSTALLED `cycc` and only warns on drift.

⭐ Worth knowing for a toolkit: **6.5.0 added file-scoped `private` / per-item `public`**, the first
real answer to this ecosystem's duplicate-`fn`-silently-shadows hazard. dhancha does not use it yet;
it is the obvious next hardening for a library whose symbols share one flat namespace with every
consumer's.

### Fixed — ⛔ every dhancha app inherited a setu connect that could not succeed on agnos before `net_src_for`

> ⛔ **SUPERSEDED 2026-08-03 — the transport this "fix" repairs is RETIRED as the wrong primitive.**
> TCP-on-loopback was the WRONG PRIMITIVE for a local display protocol — nothing to route, nothing to
> checksum, no window to negotiate, no business owning a port — and bumping to setu 0.7.2 bought one
> more accommodation (the sixth) on it. That architectural ruling, not a failure, is why it is gone.
> The desktop transport is now the agnos socket (`anu`) — agnos
> `docs/development/planning/ipc.md` §9/§10; do not restore a TCP dial on the strength of this entry.
>
> ⚠ **But do not overstate the retraction.** "Verified downstream — crab, a dhancha app, now composites
> as a live window on agnos" was a REAL, un-rigged observation on agnos 1.56.34+ (which added
> `net_src_for`, the kernel-side half of this fix): on 2026-08-02 the honest harness
> `agnos/scripts/harness/aethersafha-clients-test.py` — which hard-exits if the kernel carries any
> selftest hook — reached **`connected: 2, presented: 2`** with `crab` as one of the two clients. Scope:
> QEMU at `-smp 1`, never shown on iron, `-smp 4` fault-kills. It does not carry forward as a *current*
> capability claim, because the transport under it is retired — not because it did not happen.

setu **0.7.2** (not 0.7.1). `dh_client_connect` forwards straight to `setu_client_connect`, so **any**
app built on this toolkit carried the defect: setu dialled `127.0.0.1` while agnos puts `net_ip` in
the outbound SYN's *source*, so the SYN-ACK came back on a 4-tuple the client's own conn could not
match and `sock_connect` #47 returned -1 instantly.

⚠ **A toolkit propagates a transport bug to every consumer at once**, which is why this is worth
calling out in a cut whose headline is a toolchain refresh: pinning 0.7.1 here would have left every
dhancha app broken on a real boot even after setu itself was fixed. Verified downstream — crab, a
dhancha app, now composites as a live window on agnos.

### Verification

Host + `--agnos` builds green; **6 RUN tests** pass (`draw`, `event`, `font_render`, `layout`,
`text`, `theme`); `distlib` regenerated.

⚠ This version was **never published** (upstream stopped at 0.9.2), so the setu bump is folded into
it rather than minted as 0.9.4 — there is no released entry to correct.

## [0.9.2] - 2026-07-23

### Fixed — system-font text would have rendered WRONG under GPU blending

`src/surface.cyr` wrote glyph pixels as `store32(..., ink)` where `ink` is a rupa theme token —
`0x00RRGGBB`, **alpha byte zero**. Every other painter in the stack writes byte 3 = 255, so this one path
produced glyph pixels that were transparent-by-accident.

Harmless until now because nothing downstream read byte 3. Under agnos's `gpu_shader_op` **#92** op 0x01
(premultiplied src-over, which **does** read it) every character rendered with the system font would have
rendered wrong. Now `ink | 0xFF000000`.

> ⛔ **Corrected 2026-08-02** — this entry originally said the text *"would simply have disappeared."* That
> is wrong. The kernel shader is `out = src + dst*(1 - src_a)`, so alpha 0 gives `out = src + dst` — an
> **additive over-bright ghost**, not a disappearance, and harder to notice than missing text. The fix
> (`ink | 0xFF000000`) was correct and is unchanged; only the predicted symptom was wrong.

## [0.9.1] - 2026-07-23

### Changed — setu 0.6.0: client buffers are GPU-visible on agnos

Picks up `setu` **0.6.0**, whose `setu_buf_create` now asks for `shm_create_gpu` **#86** before falling back
to `shm_create` **#71**.

⚠ **Why this matters beyond a version number.** `#71` allocates **system RAM**, which the agnos GPU cannot
reach at all — bus-master is off by design and the engines see only the framebuffer aperture. The kernel
rejects a `#71` slot at both GPU entry points (`gpu_blit_shm` #87: `src_mc == 0 ⇒ the GPU cannot read it`;
`gpu_shader_op` #92: `GPO_E_BADSLOT`). Every shared surface in the desktop was allocated that way, so the
whole iron-proven ring-3 GPU band had **no reachable consumer**. Buffers from this release are eligible for
a hardware blit.

No API change and no call-site change here — the buffer id behaves identically, and `#86` falls back to
`#71` automatically on a machine with no GPU carveout (every QEMU boot).

### Changed — cyrius pin → 6.4.71

## [0.9.0] - 2026-07-12 — widgets follow the shared desktop theme (rupa)

The toolkit now draws with the sovereign desktop theme instead of hardcoded colours. A
widget tree rendered by dhancha matches the aethersafha compositor chrome, because both
read the same source — **rupa** (रूप, "form / appearance"), the shared theme-token core.
Switch the whole desktop's look with `rupa_theme_set_active_name("shanta-dark")` and every
dhancha surface re-colours to match. Two themes, each dark + light: MUDRA (the seal, the
default) and SHANTA (stillness).

### Added

- **`[deps.rupa]`** (`0.1.0`) + **`src/theme.cyr`** — the `dh_theme_*` helpers, each packing
  a rupa `0xRRGGBB` token (of the single active theme) into a sadish colour via `sd_rgb`:
  `dh_theme_bg` / `_panel` / `_widget` / `_line` / `_ink` / `_mute` / `_accent` / `_alert`.
  Apps set a widget background with `dh_widget_set_bg(w, dh_theme_panel())`, or just let the
  renderer use the theme automatically.
- **`programs/theme_test.cyr`** — a RUN test proving `dh_theme_*` track the active rupa theme
  and re-colour when it switches (MUDRA · Carbon → SHANTA · First Light → back).

### Changed

- **`dh_surface_render` draws with the theme.** The desktop backdrop (was `sd_rgb(32,32,40)`)
  → `dh_theme_bg()`; the button border (was black) → `dh_theme_line()`; and default
  (kashi-bitmap) text (was white) → `dh_theme_ink()`. Explicit per-widget `dh_widget_set_bg`
  colours are untouched — only the previously-hardcoded chrome now follows the theme. This
  makes the light themes legible (dark ink on paper). `draw_test`'s border assertion updated
  to `dh_theme_line()`; `text_test` unchanged (it checks text differs from bg).

## [0.8.0] - 2026-07-10 — text renders in the kashi SYSTEM font (bitmap blit), not a hand-rolled font

`dh_draw_text` now draws its default text with **kashi** — the AGNOS system console font
(full CP437, VGA 8×16, lowercase and all) — blitted directly onto the sadish surface, the
same font the compositor chrome uses. This replaces the interim in-app blocky font: apps
just set text and get the system font, with no font baking. **rekha stays** for scalable
TrueType (pass a non-zero `RekhaFont`); kashi is the default (`font = 0`).

### Added

- **`[deps.kashi]`** (1.0.2) — the system font's glyph data + accessors. dhancha's dist
  *references* `kashi_glyph_row` / `kashi_font_init` (a consumer supplies them, same as
  sadish/rekha); `src/lib.cyr` includes the module.

### Changed

- **`dh_draw_text(sds, font, text, x, y, h)` — `font == 0` now blits kashi** (per glyph, 16
  rows × 8 cols → white pixels into the sadish buffer, 9 px monospace advance, glyph centred
  in the box). `font != 0` keeps the existing rekha outline path unchanged. This is the right
  path for bitmap text — no bitmap→SFNT→outline round-trip.
- **`programs/setu_widget_client.cyr` migrated to `font = 0`** (kashi) and now types lowercase
  into its text field (kashi has the full set).

### Removed

- **`programs/blockfont.cyr`** — the interim hand-authored in-app font (both the outline and
  the later 5×7-bitmap versions). Superseded by kashi; no program uses it.

## [0.7.0] - 2026-07-10 — a real dhancha app on the sovereign desktop (setu 0.4.0 client + in-memory font)

dhancha becomes a **client that presents a widget UI over setu and is composited on agnos** —
the whole draw stack (widget tree → box layout → sadish 2D vector + rekha text → BGRA buffer →
setu → aethersafha) runs on the sovereign kernel, reacting to focus and keyboard input routed
back over the wire. Text labels are drawn from a font baked entirely in memory — no font files.

> ⛔ **RETRACTED 2026-08-03 — "composited on agnos" *as evidenced in this arc* is a FALSE GREEN.** Every
> agnos run in this arc went through the `AETHERSAFHA_SETU_SELFTEST` kernel hook, which assigned
> `net_ip = 0x7F000001` before launching the compositor; that accidental src == dst match is the only
> reason setu's loopback TCP handshake closed here. Before `net_src_for` (agnos 1.56.34) an ordinary
> boot could not complete it, so *this arc's* proof is rigged and withdrawn. The hook and its smoke are
> deleted, and TCP-on-loopback is retired as the desktop transport — retired as the **wrong primitive**
> for local display IPC, not as something that never worked: after `net_src_for`, on 2026-08-02, the
> hook-scanning harness `aethersafha-clients-test.py` reached `connected: 2, presented: 2` with the real
> dhancha app `crab` as one of the two clients (QEMU `-smp 1`; never on iron, `-smp 4` fault-kills). See
> agnos `docs/development/planning/ipc.md` §9/§10. The draw stack (layout → sadish → rekha → BGRA) is
> unaffected; this arc's transport claim is not.

### Added

- **`programs/setu_widget_client.cyr`** — a real dhancha widget client: a window with a titled
  bar + two labelled buttons (FILE / OPEN / HALT), rendered by the draw stack and presented over
  setu's shared-buffer path (`setu_buf_*` + CREATE_SURFACE → ATTACH-by-buf → COMMIT). On agnos it
  stays live, re-rendering the tree when the compositor forwards `SETU_INPUT_FOCUS` (title/border
  reflects focus) or `SETU_INPUT_KEY` (a button lights up). sadish's `SdSurface` is BGRA-packed —
  exactly setu's pixel format — so the rendered surface feeds `setu_buf_write` with no conversion.
- **`programs/blockfont.cyr`** — a tiny hand-authored blocky **rekha font baked in memory** (an
  SFNT with head/maxp/loca/glyf/cmap-format-4), so widgets carry real text with zero font assets
  on disk. Single- **and multi-contour** glyphs (holes wind opposite the outer for nonzero fill),
  covering the letters used by the demo labels. Extend `bf_letter` for a fuller alphabet.
- **`programs/font_render_test.cyr`** — a host harness that bakes the font and dumps a rendered
  string to a buffer for eyeballing (the fast iteration loop for authoring glyphs).

### Changed

- **setu dep 0.3.0 → 0.4.0** — the current shared-buffer present + `SETU_INPUT_*` input channel.
  dhancha's own 0.3.0-era `dh_setu_*` delegation is bypassed in favour of setu's direct client API.
- **cyrius pin 6.4.25 → 6.4.34** — setu 0.4.0's `setu_buf_*` needs the kernel `sys_shm_*` wrappers
  introduced in cyrius 6.4.34.
- **`src/lib.cyr` gained `result` + `net`** — the setu client transport needs cross-platform TCP
  sockets (`tcp_socket` / `sock_*` / `INADDR_LOOPBACK`); dhancha had never networked before.

## [0.6.3] - 2026-07-08 — adapt to setu 0.3.0 (cross-platform TCP transport)

> ⛔ **RETRACTED 2026-08-03 — "cross-platform on Linux and agnos" was not yet true on agnos when this
> was written.** The TCP transport adopted here could not complete a compositor↔client handshake on an
> ordinary agnos boot **until `net_src_for` (agnos 1.56.34)**: before that, every outbound segment
> claimed `net_ip` as its source, so a SYN to 127.0.0.1 was answered on a 4-tuple the client's own conn
> could not match. It later did complete un-rigged (2026-08-02, hook-scanning harness, `connected: 2,
> presented: 2`, QEMU `-smp 1`) — but it is RETIRED as the desktop transport anyway, because TCP is the
> **wrong primitive** for local display IPC. The replacement is the agnos socket (`anu`) — agnos
> `docs/development/planning/ipc.md` §9/§10. setu's Linux arm survives because Linux is a different
> target, not an agnos fallback.

setu 0.3.0 replaced its Linux-only AF_UNIX client with a cross-platform **TCP**
transport (item 3b), dropping the `sockaddr_un` builder. dhancha's thin client
layer adapts — one dead forwarder removed, the pin bumped. No behavior change for
dhancha's own client API (`dh_setu_connect` / `dh_setu_*` / `setu_client_*` all
forward unchanged).

### Changed

- **`[deps.setu]` → 0.3.0** — the shared reference client transport is now TCP
  over loopback:7700 (`net.cyr`), cross-platform on Linux and agnos.

### Removed

- **`dh_setu_sockaddr`** (`src/setu_client.cyr`) — a forwarder to setu's
  `setu_cl_sockaddr`, which no longer exists under the TCP transport (there is no
  socket path to marshal; the address is the implicit loopback endpoint). It had
  no callers.

## [0.6.2] - 2026-07-08 — draw-stack pin alignment

Dep-hygiene release — no code change. Aligns the draw-stack deps with the
toolchain-alignment cuts and drops their dev path-overrides.

### Changed

- **`[deps.sadish]` → tag `0.4.1`, `[deps.rekha]` → tag `0.3.1`** (both bumped to
  the toolchain-aligned 6.4.25 cuts), and the dev `path = "../sibling"` overrides
  dropped — tag-only, reproducible, matching the `[deps.setu]` cleanup in 0.6.1.
  Push order: sadish → rekha → dhancha. dhancha's own cyrius pin was already
  `6.4.25`.

## [0.6.1] - 2026-07-08 — setu client dedup (delegate to setu's promoted client)

### Changed

- **The setu client transport is no longer duplicated — it is setu's.** setu
  **0.2.0** promoted the reference client (`setu_connect` / `setu_send` /
  `setu_read_msg` + the persistent `setu_client_*`) into the protocol lib so
  dhancha and puka share ONE implementation. dhancha's `src/setu_client.cyr` is
  now thin forwarders + one-shot (connect→do→close) convenience wrappers over
  setu's primitives — **zero re-implemented framing** (no raw `SYS_SOCKET`/
  `SYS_CONNECT` left in the client files). `DhClient` (`src/dh_client.cyr`)
  delegates straight to `setu_client_*` — a `*DhClient` **is** a `*SetuClient`
  (identical `{fd, sid}` layout) — keeping only `dh_surface_render`
  (widgets → pixels) + the DhEvent map. All `dh_*` signatures are unchanged.
- **`[deps.setu]` pinned to tag `0.2.0`** (was a dev `path` override + a stale
  `0.1.0` tag).

### Verified

- All 6 setu programs build; end-to-end `setu_demo_client` → aethersafha
  **blit-verified** (widget tree composited). No behavioral change on the
  success path; `dh_client_present` now surfaces `setu_client_present`'s error
  codes.

## [0.6.0] - 2026-07-08 — native display protocol (setu client binding)

### Added

- **The setu client binding — dhancha speaks the native display protocol.** The
  full client side of the sovereign dhancha ↔ aethersafha wire (`setu`), built
  incrementally and proven end-to-end on Linux against the real compositor
  (keyboard + pointer, zero Wayland):
  - **`src/setu_client.cyr`** — the setu transport: `dh_setu_connect` (AF_UNIX),
    `dh_setu_send` (encode + write), `dh_setu_recv` / `dh_setu_read_msg`
    (length-from-header framing for a message *stream*), `dh_setu_read_exact`,
    plus the lifecycle + present helpers (`dh_setu_create_surface`,
    `dh_setu_send_buffer`, `dh_setu_send_pixels`).
  - **`src/setu_input.cyr`** — the "compositor-fd input source": maps setu
    `INPUT_KEY` / `POINTER_MOVE` / `POINTER_BTN` / `FOCUS` frames 1:1 into
    `DhEvent`s (`dh_setu_map_input` / `dh_setu_read_event`).
  - **`src/dh_client.cyr`** — the app-facing **`DhClient`** binding
    (`dh_client_connect` / `dh_client_present` / `dh_client_next_event` /
    `dh_client_close`): connect once, present a rendered widget tree, and pump
    input off the same connection ("each app owns its connection").
  - Adds a dependency on the new **[`setu`](https://github.com/MacCracken/setu)
    0.1.0** contract lib (typed messages + wire codec).
  - Proven: a real dhancha widget tree (sadish-rasterized, rekha TrueType text)
    rendered and presented over setu and composited by the **real** aethersafha
    compositor, with input events flowing back as `DhEvent`s. The setu modules
    are opt-in (included alongside the toolkit); folding them into the core
    distribution is a follow-up packaging decision.

### Changed

- **The v0.6+ compositor seam is the native display protocol — Wayland refused.**
  The remaining client stubs (`dh_surface_present`, the `dh_run` input source)
  bind to aethersafha's **native, first-principles display protocol**, not a
  Wayland client. The "Wayland socket / commit" framing in the deferred lists of
  earlier releases below is superseded; the direction lives in
  [`docs/development/sovereign-desktop.md`](docs/development/sovereign-desktop.md)
  (and the ecosystem pivot in `agnosticos/docs/design-patterns.md`). Source
  comments swept to match; no code change.

## [0.5.0] - 2026-07-06

The layout engine — `BOX_V` / `BOX_H` become real flex containers, plus
intrinsic measure (natural content size). `layout_test` now covers stacking,
padding, gap, flex, alignment, measure, and fit.

### Added
- **Flex grow** — `dh_widget_set_flex(w, weight)`. Fixed children (weight 0)
  take their preferred main-axis size; children with weight > 0 split the
  container's leftover main-axis space in proportion to their weights. The
  last flex child gets the exact remainder, so integer rounding never loses
  or overshoots a pixel.
- **Padding** — `dh_widget_set_padding(w, px)` uniformly insets a container's
  content box before its children are arranged.
- **Spacing** — `dh_widget_set_gap(w, px)` separates consecutive children in
  `BOX_H` / `BOX_V`.
- **Cross-axis alignment** — `dh_widget_set_align(w, DhAlign)` positions a
  child on the cross axis: `ALIGN_STRETCH` (default) fills it, `ALIGN_START` /
  `ALIGN_CENTER` / `ALIGN_END` use the child's preferred cross size.
- **Intrinsic measure** — `dh_measure(w, out)` computes a widget's natural
  content size bottom-up (leaf → its pref; container → children combined per
  mode + padding + gaps), with `dh_measure_w` / `dh_measure_h` shorthands.
  `dh_layout_fit(root)` measures then lays out at the natural size (a
  shrink-to-fit window). Inside a flex box, a fixed child with no preferred
  main size now auto-sizes to its measured content, so nested containers fit.
- `BOX_V` / `BOX_H` are now flexbox-style rows/columns; `FLEX` is an alias of
  `BOX_V`; `NONE` remains absolute overlay (now padding-aware). The arranger
  is factored into `dh_layout_box` (flex) + `dh_layout_none` (overlay).
- Test: `layout_test` — padding + gap insets, flex 50/50, weighted flex (1:2
  with exact remainder), mixed fixed + flex, cross-axis align in both `BOX_V`
  (cross = width) and `BOX_H` (cross = height), and measure / `dh_layout_fit`
  (leaf, BOX_V/BOX_H, a nested tree, and auto-sizing a pref-less container).

### Deferred
- Per-edge padding + margins; flex-wrap; the compositor-fd input source
  (decode `wl_pointer` / `wl_keyboard` wire bytes + block on the Wayland
  socket — cross-repo, needs aethersafha); the present path (mabda GPU upload
  + the aethersafha Wayland commit).

## [0.4.0] - 2026-07-06

Finishes the event model — the two items 0.3.0 deferred: capture-phase
propagation and a drag-drop state machine. `event_test` now runs sub-tests
A–Q (adds capture + drag coverage).

### Added
- **Capture-phase routing** — dispatch now runs a full two-phase propagation
  (`dh_propagate`): a **capture** pass root→target invoking each node's
  capture handler (`dh_widget_set_capture_handler`), then a **bubble** pass
  target→root. The first handler in *either* pass to return 1 consumes the
  event and stops all propagation, so a capture handler can intercept an event
  before the target's bubble handler sees it. `DhEvent` gains `DH_E_PHASE`
  (`dh_event_phase` → `DH_PHASE_CAPTURE` / `DH_PHASE_BUBBLE`). Existing bubble
  handlers are unaffected (the capture pass is a no-op with none registered).
- **Drag-drop state machine** — widgets opt in via flags
  (`dh_widget_set_draggable` / `dh_widget_set_drop_target`). A press on a
  draggable widget that then travels past `DH_DRAG_THRESHOLD` (Manhattan)
  emits `DRAG_START`→`DRAG_MOVE…` to the source; the release emits `DRAG_DROP`
  to the widget under the pointer *iff* it is a drop target, then always
  `DRAG_END` to the source. Drag events carry the source in `DH_E_SOURCE`
  (`dh_event_source`). A drag suppresses the click `ACTIVATE`; a press+release
  that never crosses the threshold still clicks.
- Event kinds `DRAG_START` / `DRAG_MOVE` / `DRAG_DROP` / `DRAG_END`; widget
  flags `DH_FLAG_DRAGGABLE` / `DH_FLAG_DROP_TARGET`; `dh_reset_input` also
  clears the in-progress drag.
- Test: `event_test` sub-tests M–Q — capture runs-then-bubble with correct
  phases, capture-consume suppresses bubble, a full press→drag→drop cycle
  (with source identity + no spurious click), click-on-draggable (no drag),
  and drop over a non-target (END, no DROP).

### Deferred

The client-side toolkit is feature-rich; these are the remaining v0.5+ items
(referenced from the in-code `TODO (see CHANGELOG)` markers):

- **Compositor-fd input source** — decode `wl_pointer` / `wl_keyboard` wire
  bytes into `DhEvent`s and block on the Wayland socket (the `dh_run` loop
  currently pumps an in-memory `DhQueue`). Cross-repo (needs aethersafha's
  wire); the `DhQueue` is the seam it will feed.
- **Present path** — `dh_surface_present`: mabda GPU upload + the aethersafha
  Wayland commit. The CPU draw shipped in 0.2.0 (`dh_surface_render`).
- **Flex layout** — flex grow/shrink, an intrinsic measure pass,
  padding/spacing, cross-axis alignment (box stacking ships today).
- **Hit-test refinement** — z-order, clipping, and input-transparency (the
  current hit-test is a plain contains-point walk).

## [0.3.0] - 2026-07-06

Event dispatch — the widget tree becomes interactive: hit-testing, keyboard
focus + Tab traversal, per-widget handlers, bubble propagation, hover
enter/leave, click + keyboard activation, and an event-loop pump. Adds
`event_test` (12 sub-tests, A–L). The toolkit draws AND responds now; the
compositor-fd input source is the remaining seam.

### Added
- **Per-widget event handlers** — `DhWidget` gains a handler fnptr + userdata
  slot (`dh_widget_set_handler(w, &fn)` where `fn(wgt, ev) -> consumed`, plus
  `dh_widget_set_userdata` / `dh_widget_userdata`). Dispatch invokes through
  the stdlib `fncall2` (null-checked).
- **Bubble routing** (`dh_bubble`) — an event resolves a target, then bubbles
  up the parent chain invoking each node's handler until one returns 1
  (consumes); `DhEvent` gains a `dh_event_consumed` accessor.
- **Pointer-position tracking** — `POINTER_MOVE` updates a tracked cursor
  position; `POINTER_BTN` / `DRAG` hit-test against it, matching the Wayland
  wire model where a button event carries `(button, state)` — not coordinates.
  `dh_pointer_x` / `dh_pointer_y` expose it.
- **Hover enter/leave** — as the pointer crosses widget boundaries the toolkit
  synthesizes `POINTER_ENTER` / `POINTER_LEAVE` to the widget entered / left
  (`dh_update_hover`, `dh_hover_get`).
- **Click activation** — a `POINTER_BTN` press records the press target and
  takes keyboard focus if it is focusable (`dh_widget_focusable`:
  BUTTON / TEXTINPUT); a release on the *same* widget synthesizes an `ACTIVATE`
  (a click). KEY / FOCUS events route to the focused widget.
- **Keyboard activation** — Enter / Return / Space on a focused `BUTTON`
  synthesizes `ACTIVATE`, so buttons fire from the keyboard too.
- **Tab focus traversal** — `Tab` / `Shift-Tab` cycle keyboard focus among the
  focusable widgets in pre-order, wrapping (`dh_focus_advance`); the key is
  toolkit-consumed and does not route to a widget.
- **Event queue + loop** — `DhQueue`, a fixed-capacity ring of events
  (`dh_queue_new` / `_push` / `_pop` / `_count` / `_empty`), and
  `dh_run(root, q)` which pops + dispatches until the queue drains or a handler
  calls `dh_quit`. The ring is the seam the future compositor-fd translator feeds.
  Null events are rejected at push and `dh_run` returns `-1` (not an overloaded
  error code) on null args — both hardened after an adversarial review pass.
- **`dh_reset_input`** — clears transient input state (focus / hover / press)
  when an app swaps the widget tree, so a stale pointer can't route into a
  torn-down tree.
- New event kinds `POINTER_ENTER` / `POINTER_LEAVE` / `ACTIVATE`, and `DhKey`
  constants (`DH_KEY_TAB` / `_ENTER` / `_RETURN` / `_SPACE`).
- Test: `event_test` — hit-test, click-to-focus, keyboard-to-focus, bubble,
  consume-stops-bubble, queue drain / wrap-around / full-drop, quit-mid-drain,
  hover enter/leave (per-widget targets), pointer + keyboard activation, and
  Tab forward / backward / wrap (three focusables, so the two directions
  provably diverge).

### Fixed
- **Focus-slot segfault** — `_dh_focus` was a `var X[1]` module-array read via
  `load64` (out-of-bounds; module-global `var X[N]` sizing is non-uniform in
  Cyrius). Switched to a scalar — the same latent trap fixed for the widget-id
  counter in 0.2.0, here it would have fired the first time focus was set.

### Deferred
- Capture-phase routing (bubble-only for now); a drag-drop state machine; the
  compositor-fd input source (decode wl_pointer / wl_keyboard wire bytes into
  DhEvents + block on the socket); mabda GPU upload + aethersafha commit.

## [0.2.0] - 2026-07-05

The toolkit draws: real box layout, and a render path that draws the widget
tree via sadish (fills/strokes) + rekha (text). The full draw stack —
dhancha → rekha → sadish → pixels — is validated as a unit. 3 RUN tests.

### Added
- **Draw-path deps wired** — `[deps.sadish]` (0.4.0) + `[deps.rekha]` (0.3.0),
  local path overrides. dhancha now consumes the whole draw stack.
- **Widget style + layout fields** — `DhWidget` gains a layout mode, background
  color, text, and preferred size, with setters (`dh_widget_set_layout` /
  `_set_bg` / `_set_text` / `_set_pref`).
- **Box layout** (`dh_layout_at` / `dh_layout_apply`) — real `BOX_V` / `BOX_H`
  stacking (top→bottom / left→right by preferred size), replacing the skeleton.
- **Widget draw** (`dh_surface_render`, `dh_draw_widget`, `dh_draw_text`) —
  renders the widget tree into a sadish `SdSurface`: backgrounds via
  `sd_fill_rect`, button borders via `sd_rect`, and `LABEL`/`BUTTON` text via
  rekha (`rekha_char_to_sdpath` → coverage → blit). Fixed-advance text for now.
- Tests: `layout_test` (box stacking), `draw_test` (bg + border pixels),
  `text_test` (full stack: dhancha → rekha → sadish glyph coverage).

### Fixed
- **Widget-id counter segfault** — `_dh_next_widget_id` was a `var X[1]`
  module-array read via `load64` (out-of-bounds; module-global `var X[N]`
  sizing is non-uniform in Cyrius). Switched to a scalar. Latent since the
  scaffold — surfaced the first time `dh_widget_new` ran.

### Deferred
- Event dispatch (hit-test + pointer/keyboard routing) → v0.3; flex layout,
  intrinsic measure, padding/spacing; real hmtx text advances; mabda GPU upload
  + aethersafha Wayland commit (CPU draw first).

## [0.1.0] - 2026-07-05

### Added
- Repo scaffolded: pure-Cyrius client-side widget toolkit / desktop app
  framework (Qt/GTK-equivalent) — buildable, link-checkable skeleton
  with the widget-tree / layout / event-loop+input-dispatch / surface
  module surfaces (`src/error.cyr`, `src/widget.cyr`, `src/layout.cyr`,
  `src/event.cyr`, `src/surface.cyr`), the `src/lib.cyr` include chain,
  and `programs/smoke.cyr` link-check. `cyrius = "6.4.7"`, GPL-3.0-only.
  Draw/present cross-deps (sadish + rekha + mabda) are deferred to v0.2.
