# Changelog

All notable changes to dhancha are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

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
