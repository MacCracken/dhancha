# Changelog

All notable changes to dhancha are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

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
