# dhancha

Version: 0.10.4

**dhancha** (ढाँचा — Hindi/Sanskrit: *framework / structure / scaffold*)
is a pure-Cyrius **client-side widget toolkit / desktop app framework**
for AGNOS — the Qt/GTK-equivalent layer. Desktop GUI apps build their
UIs on dhancha instead of hand-rolling raw GPU + a raw compositor connection (which
`puka`, the first windowed program, does today). dhancha is the
spiritual extraction of puka's windowing code.

It owns:

- a retained-mode **widget tree** (`WINDOW` / `BOX` / `LABEL` / `BUTTON` / `TEXTINPUT` / `LIST` /
  `CANVAS` / `PROGRESS` / `GRID`, plus MENU and SHEET composed from an overlay layer),
- **layout** (flexbox-style `BOX_V` / `BOX_H` — grow / shrink, padding, gap, cross-axis alignment —
  and intrinsic measure),
- an **event loop** and **input dispatch** — keyboard / pointer / scroll, focus + `Tab` traversal,
  hover, activation, capture + bubble propagation, and a drag-drop state machine,
- the **draw** — the tree into a sadish `SdSurface`: backgrounds and borders via sadish, text via
  kashi (the bitmap system font, CP437 cells for what the code page has) or rekha (a scalable face),
  one glyph per UTF-8 character either way, colours from rupa's theme tokens; with a per-frame arena
  so a rendered frame costs the global heap nothing, and
- the **client connection** — `dh_client_connect` / `_present` / `_next_event` / `_poll_event`
  over setu, the display-protocol contract shared with the aethersafha compositor.

It is the **CLIENT-side counterpart to `aethersafha`** (the
compositor/server) — analogous to how `cmdit` is the arg/CLI lib for
terminal apps. dhancha produces client surfaces that the aethersafha
compositor composites onto the screen over the native display protocol.

## Where things are

- **What shipped, and what each claim measured** — [`CHANGELOG.md`](CHANGELOG.md), newest first.
- **What is next** — [`docs/development/roadmap.md`](docs/development/roadmap.md): forward-facing
  work only, every item with its evidence in the tree.
- **The display-seam direction** — [`docs/development/sovereign-desktop.md`](docs/development/sovereign-desktop.md).
  Wayland is refused, not ported.
- **Filed issues** — [`docs/development/issues/`](docs/development/issues/) while open,
  [`issues/archived/`](docs/development/issues/archived/) once a release addresses them.
- **The gates** — `programs/*_test.cyr`, nineteen self-checking RUN suites; CI runs them all, plus
  lint / fmt / vet / distlib / sidecar.

## Place in the stack

```
  desktop apps  (build UIs on dhancha)
        │
     dhancha                        ← client-side widget toolkit (this repo)
        │  draws via
   sadish · rekha · kashi · rupa    ← 2D vector · scalable fonts · the system font · theme tokens
        │  presents a client surface over
   setu                             ← the native display protocol (contract + reference client)
        │  to
   aethersafha                      ← compositor / server (composites to screen)
```

The present path is complete: sadish's pixels go into a kernel-owned shared buffer that setu asks
for GPU-visible on hardware, and aethersafha composites it on the GPU. dhancha draws on the CPU by
design and makes no GPU call of its own.

## Consumers

- **Desktop GUI apps** — build their UIs on dhancha's widget tree,
  layout, event loop, and input dispatch instead of hand-rolling raw GPU
  + a raw compositor connection.
- **puka** — the first windowed program; its windowing code is the
  spiritual origin of dhancha, and it is the reference consumer as the
  toolkit fills in.
- **aethersafha** — the compositor on the other side of the native display
  protocol: dhancha emits the client surfaces it composites (dhancha is the
  client-side counterpart to aethersafha's server side).

## Dependencies

- **Cyrius stdlib** — `string`, `fmt`, `alloc`, `io`, `vec`, `str`,
  `syscalls`, `assert`, `bench`, `args`, plus `hashmap` (widget-id →
  handler routing), `fnptr` (event-callback dispatch), and `tagged`
  (tagged-value payloads). Resolved by `cyrius deps` into `lib/`.
- **sadish** (0.11.2) + **rekha** (0.9.0) — the draw path: sadish fills/strokes
  widget backgrounds/borders, rekha rasterizes text (via sadish). ⛔ Both floors
  are HARD: 0.10.0's scalable text installs `dh_falloc` through sadish 0.5.5's
  `sd_alloc_set` and blits with `sd_canvas_blit_at`, and rekha 0.3.10 is the
  version whose outline scratch follows that hook. The pins sit well above both
  floors since 0.10.1; nothing dhancha calls changed shape across sadish 0.6–0.11
  or rekha 0.4–0.9, and rekha 0.9.0 is cut against sadish 0.11.2 exactly, so the
  two move together.
- **rupa** (0.1.7) — the shared desktop theme tokens (`dh_theme_*`), the same
  source the compositor reads.
- **kashi** (1.0.10) — the VGA 8x16 system font the default (`font = 0`) text
  path blits; vendored as its freestanding core (see the manifest's ⛔).
- **setu** (0.8.9) — the display-protocol contract and reference client that
  `dh_client_connect` / `dh_setu_*` delegate to.
- **mabda** (GPU) — not a dependency, by design: dhancha draws on the CPU and
  the hardware path is the buffer's (setu) and the compositor's (aethersafha).

Every `[deps.*]` resolves from its published git tag, and `cyrius.lock` carries
the commit each tag resolved to. The `path = "../sibling"` overrides are kept
commented out in the manifest for cross-repo work on an unpushed sibling — ⛔
`path` wins over `tag` and pins no commit, so never commit one live.

The toolchain pin is `cyrius = "6.6.6"`.

## Quick Start

```bash
cyrius deps                                          # resolve the stdlib and every [deps.*] tag into lib/
cyrius build programs/smoke.cyr build/dhancha-smoke  # link-check
./build/dhancha-smoke                                # prints the banner
for t in programs/*_test.cyr; do cyrius build "$t" build/t && ./build/t || break; done   # the gates
```

## License

GPL-3.0-only.
