# dhancha

Version: 0.4.0

**dhancha** (ढाँचा — Hindi/Sanskrit: *framework / structure / scaffold*)
is a pure-Cyrius **client-side widget toolkit / desktop app framework**
for AGNOS — the Qt/GTK-equivalent layer. Desktop GUI apps build their
UIs on dhancha instead of hand-rolling raw GPU + raw Wayland (which
`puka`, the first windowed program, does today). dhancha is the
spiritual extraction of puka's windowing code.

It owns:

- a retained-mode **widget tree** (window / box / label / button / text
  input),
- **layout** (box + flex measure/arrange),
- an **event loop**, and
- **input dispatch** — keyboard / pointer, focus, and drag-drop.

It is the **CLIENT-side counterpart to `aethersafha`** (the
compositor/server) — analogous to how `cmdit` is the arg/CLI lib for
terminal apps. dhancha produces Wayland client surfaces that the
aethersafha compositor composites onto the screen.

## Scope

- **v0.1.0 — scaffold.** A buildable, link-checkable pure-Cyrius
  skeleton — **not** the functional toolkit yet. Real types and real
  function signatures with small/stub bodies, so the include chain
  (stdlib + domain modules) compiles clean and `cyrius distlib` can
  bundle it. The domain modules present their public API surface:
  - `src/error.cyr` — `DhanchaErr` model (`DHANCHA_OK` / `_ERR_OOM` /
    `_ERR_BAD_WIDGET` / `_ERR_NO_SURFACE` / `_ERR_LAYOUT` /
    `_ERR_UNSUPPORTED` / `_ERR_OTHER`), a 16-byte record, and
    `dhancha_err_name`.
  - `src/widget.cyr` — `DhWidget` (first-child / next-sibling tree),
    `DhWidgetKind { WINDOW / BOX / LABEL / BUTTON / TEXTINPUT }`,
    `dh_widget_new` / `dh_widget_add_child` + bounds accessors.
  - `src/layout.cyr` — `DhRect`, `DhLayout { NONE / BOX_H / BOX_V /
    FLEX }`, and a `dh_layout_apply` tree-walk skeleton.
  - `src/event.cyr` — the **input-dispatch** module: `DhEventKind`,
    `DhEvent`, `dh_event_new`, hit-testing, `dh_dispatch` (route to the
    focused / hit widget), and the `dh_run` event-loop skeleton.
  - `src/surface.cyr` — `DhSurface` (client window + RGBA8 pixel
    buffer), `dh_surface_new` / `dh_surface_present` skeleton.
  - `programs/smoke.cyr` — the link-check entry.
- **v0.2.0 — the toolkit draws (shipped).** sadish + rekha wired; real box
  layout (`dh_layout_at` — `BOX_V` / `BOX_H` stacking); a render path
  (`dh_surface_render`) that draws the widget tree into a sadish `SdSurface` —
  backgrounds/borders via sadish, `LABEL` / `BUTTON` text via rekha. The full
  draw stack (dhancha → rekha → sadish → pixels) is RUN-tested.
- **v0.3.0 — the toolkit responds (shipped).** Event dispatch: hit-testing,
  keyboard focus + `Tab` traversal, per-widget handlers (fnptr) with bubble
  propagation, hover enter/leave, click + keyboard activation, and an
  event-loop pump over a `DhQueue` (`dh_run` / `dh_quit`). Buttons hover, click,
  and keyboard-activate; the whole path is RUN-tested (`event_test`).
- **v0.4.0 — the event model completes (shipped).** Capture-phase propagation
  (`dh_propagate` — capture root→target then bubble, opt-in capture handlers)
  and a drag-drop state machine (draggable / drop-target flags, a move
  threshold, `DRAG_START` / `MOVE` / `DROP` / `END`); a drag suppresses the
  click. RUN-tested (`event_test`, sub-tests A–Q).
- **v0.5+ — next.** The compositor-fd input source (decode `wl_pointer` /
  `wl_keyboard` wire bytes into events + block on the Wayland socket), flex
  layout + intrinsic measure, real hmtx text advances, and the present path
  (mabda GPU upload + the aethersafha Wayland commit).

## Place in the stack

```
  desktop apps  (build UIs on dhancha)
        │
     dhancha            ← client-side widget toolkit (this repo)
        │  draws via
   sadish · rekha · mabda   ← 2D vector · fonts · GPU  (DEFERRED cross-deps)
        │  presents a Wayland client surface to
   aethersafha         ← compositor / server (composites to screen)
```

The draw/present path — **sadish** (2D vector), **rekha** (fonts), and
**mabda** (GPU) — is a **deferred cross-dependency**: not wired at the
scaffold. As the draw/present code lands (v0.2), add to `cyrius.cyml`:

```toml
[deps.sadish] path = "../sadish"
[deps.rekha]  path = "../rekha"
[deps.mabda]  path = "../mabda"
```

## Consumers

- **Desktop GUI apps** — build their UIs on dhancha's widget tree,
  layout, event loop, and input dispatch instead of hand-rolling raw GPU
  + raw Wayland.
- **puka** — the first windowed program; its windowing code is the
  spiritual origin of dhancha, and it is the reference consumer as the
  toolkit fills in.
- **aethersafha** — the compositor on the other side of the Wayland
  wire: dhancha emits the client surfaces it composites (dhancha is the
  client-side counterpart to aethersafha's server side).

## Dependencies

- **Cyrius stdlib** — `string`, `fmt`, `alloc`, `io`, `vec`, `str`,
  `syscalls`, `assert`, `bench`, `args`, plus `hashmap` (widget-id →
  handler routing), `fnptr` (event-callback dispatch), and `tagged`
  (tagged-value payloads). Resolved by `cyrius deps` into `lib/`.
- **sadish** (0.4.0) + **rekha** (0.3.0) — the draw path: sadish fills/strokes
  widget backgrounds/borders, rekha rasterizes text (via sadish). Wired as
  `[deps.*]` (local `../` path overrides for dev; git tags as published pins).
- **mabda** (GPU) — still deferred; the present path (GPU upload + Wayland
  commit) is a later bite. The CPU draw is complete.

The toolchain pin is `cyrius = "6.4.7"`.

## Quick Start

```bash
cyrius deps                                          # resolve stdlib into lib/
cyrius build programs/smoke.cyr build/dhancha-smoke  # link-check
./build/dhancha-smoke                                # prints the banner
```

## License

GPL-3.0-only.
