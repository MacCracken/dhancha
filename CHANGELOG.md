# Changelog

All notable changes to dhancha are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [0.7.0] - 2026-07-10 — a real dhancha app on the sovereign desktop (setu 0.4.0 client + in-memory font)

dhancha becomes a **client that presents a widget UI over setu and is composited on agnos** —
the whole draw stack (widget tree → box layout → sadish 2D vector + rekha text → BGRA buffer →
setu → aethersafha) runs on the sovereign kernel, reacting to focus and keyboard input routed
back over the wire. Text labels are drawn from a font baked entirely in memory — no font files.

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
