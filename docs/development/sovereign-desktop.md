# The Sovereign Desktop — dhancha's compositor seam (post-Wayland)

> Direction doc, opened 2026-07-06. dhancha is complete on the client side at
> **0.5.0**; the remaining frontier is the display seam to the compositor.
> This is a **design arc**, not a single dhancha bite — it spans dhancha
> (client), aethersafha (compositor), and a native protocol contract between
> them. The deep design decisions are the founder's; this doc frames them.

## Locked direction

- **Wayland is DEAD — not ported.** AGNOS went native. We do **not** port the
  Wayland wire protocol, `xdg-shell`, or `wl_shm`; we do **not** copy puka's
  `wl_*` client code (that is a host-Hyprland *dev crutch*, not the AGNOS
  path). aethersafha's ported-from-Rust Wayland-server work (its "Bite F", the
  ~3360-line `rust-old/src/wayland/*`) is **retired**, not implemented — it
  should be redefined as the native protocol below (aethersafha's call).
- **Own our own desktop — not a port of another desktop.** The desktop
  *paradigm* (compositor, shell, window management, interaction model) is
  AGNOS's own, designed from first principles — AI-native per the OS identity,
  **not** a GNOME/KDE/Wayland reimplementation. "Wayland-esque" describes only
  the loose shape (clients submit surfaces; a compositor composites and routes
  input) — sovereign in protocol, implementation, and paradigm.
- **Apps stay agnostic.** dhancha builds + proves on **Linux** today; "port" =
  Rust→Cyrius, never Linux→agnos. The compositor seam is the one agnos-facing
  boundary, alongside the platform backends (bhumi / agnodrm / vani). See the
  ecosystem note *"the OS-agnostic layer IS the swallow mechanism."*
- **Client support is the application's / toolkit's responsibility.** The
  compositor is not a server that arbitrary clients dial into over a borrowed
  protocol; **dhancha owns the client-side binding** that apps carry, and each
  app owns its connection. The compositor owns the desktop; the toolkit owns
  the client.

## Where dhancha stands (done, sovereign, agnostic)

dhancha 0.5.0 is a **complete client-side widget toolkit** — the sovereign
Qt/GTK-equivalent — built from scratch, no Linux-desktop lineage:

- **Widget tree** — retained-mode first-child/next-sibling nodes.
- **Layout engine** — flexbox-style `BOX_V`/`BOX_H` (flex grow, padding, gap,
  cross-axis align) + **intrinsic measure** (`dh_measure` / `dh_layout_fit`).
- **Draw** — `dh_surface_render` walks the tree into a **sadish** `SdSurface`
  (CPU BGRA pixels); text via **rekha**. The full draw stack proves on Linux.
- **Event model** — hit-test, focus + Tab traversal, per-widget fnptr handlers,
  two-phase capture/bubble propagation, hover enter/leave, click + keyboard
  activation, and a **drag-drop** state machine. Driven today by an in-memory
  `DhQueue` via `dh_run` — the seam a real input source will feed.

The two things that are still stubs — **present** (`dh_surface_present` is a
no-op that clears the dirty flag) and the **input source** (`dh_run` pumps an
in-memory queue, not a live transport) — are exactly the compositor seam.

## The seam to design (the native display protocol)

dhancha (client) ↔ aethersafha (compositor) over a **native, sovereign
protocol**. Greenfield: today aethersafha owns a native window model
(`compositor.cyr` window vec → `render.cyr` fill to the bhumi framebuffer) and
has **zero client IPC** — it creates a couple of windows internally and
composites them. There is no protocol yet to design *against*; we design it.

What the protocol must carry (the sovereign analogs of the concepts Wayland
also needs — but designed as our own, no `wl_`/`xdg_` inheritance):

1. **Transport / connection** — how a client attaches to the compositor
   (socket? a shared-memory command ring? — a sovereign choice, not
   `$WAYLAND_DISPLAY`/AF_UNIX-by-obligation).
2. **Surface lifecycle** — create a surface, configure (size/state from the
   compositor), commit, close. The compositor's `Window` heap struct is the
   natural server-side anchor; the client-facing handle is new.
3. **Buffer submission (present)** — a client hands the compositor a pixel
   buffer to composite. **CPU shared-memory first** (memfd/mmap, zero-copy):
   `dh_surface_render` → `SdSurface` BGRA → convert to the compositor's pixel
   format → submit. **No mabda / no GPU** in the first cut; GPU upload
   (dmabuf / mabda texture) is a later optimization.
4. **Input events** — compositor → focused client: pointer motion / button,
   keyboard key + modifiers, focus enter/leave. These map onto **existing
   `DhEvent`s** (`POINTER_MOVE` / `POINTER_BTN` carrying (button,state),
   `KEY` carrying a keysym) — the input model dhancha already built. A keycode
   → keysym map is needed (sovereign keymap, mine puka's `input/keymap.cyr`
   for the table, not the Wayland decode).

**Where the protocol contract lives** — the types + wire codec belong in a
**shared sovereign library** both aethersafha (server) and dhancha (client)
depend on, so the coupling is at the ABI, not the codebase (monolithic by
design). Naming + scaffolding a new repo is the founder's call — do not
scaffold unprompted.

## "Own the desktop" — the paradigm, not just plumbing

The protocol is the contract; the **desktop paradigm** is the product, and it
is the part that must not be a port. Open, founder-led design questions:

- What is the AGNOS window/interaction model? (Tiling? Spatial? Agent-driven
  surfaces? Something with no desktop precedent — AI-native.)
- What is the shell? aethersafha already has a themed desktop + panel + agent
  windows — is that the shell, or a placeholder?
- How do AI agents present surfaces? (aethersafha models an "agent" window
  flag + mehman foreign guests today — is the native protocol agent-first?)

dhancha's job is to be the **app toolkit** underneath whatever paradigm the
compositor defines; the widget/layout/event/draw layers are paradigm-neutral
and already done.

## Provable on Linux (the agnostic dividend)

aethersafha runs on **Linux** via its bhumi backend, so the whole chain —
**dhancha → native protocol → aethersafha → bhumi → screen** — is testable on
a dev box with **no agnos hardware**. Concretely:

- **Headless RUN tests** (the dhancha discipline): the protocol wire codec and
  the input-event → `DhEvent` mapping are pure functions — feed synthetic
  bytes, assert the decoded events (exactly how `event_test` already drives
  `dh_dispatch`). Most of the client side is unit-testable without a live
  compositor.
- **Live proof**: a small windowed demo app (dhancha's puka-equivalent —
  `programs/demo.cyr`) connects to a locally-running aethersafha and shows a
  laid-out, interactive widget tree. This is the end-to-end gate.

*(Verify bhumi's Linux/dev backend mode before committing to this as the CI
path; it may be a manual dev-desktop proof rather than headless CI.)*

## Cross-repo sequence (high level)

1. **Design** the native protocol + the desktop paradigm (design doc / ADR —
   founder-led). Retire aethersafha's Wayland Bite F; redefine as native.
2. **Shared contract lib** — protocol types + wire codec (sovereign repo, TBD),
   consumed by both sides.
3. **aethersafha (server)** — accept clients, surface registry mapped to its
   `Window` model, buffer submit → composite, input → client dispatch.
4. **dhancha (client)** — connect over the transport; `dh_surface_present`
   submits the rendered buffer; native input → `DhEvent`s into `dh_run`;
   replace the in-memory-queue loop with a transport-fd loop.
5. **Present** — CPU shared-buffer first (no mabda); GPU upload later.

Steps 3 and 4 co-design against the step-1/2 contract. dhancha's client side
(steps 4–5) is the part this repo owns; it is gated on the protocol contract
existing, which is gated on the paradigm/protocol design.

## Explicitly NOT doing

- Porting the Wayland wire protocol, `xdg-shell`, or `wl_shm`.
- Copying puka's `src/platform/wayland/*` client (host-Hyprland dev crutch).
- Cloning an existing desktop environment or its interaction model.
- Pulling in mabda/GPU for the first present path (CPU shared-buffer first).

## Open decisions (founder / architecture)

- The AGNOS desktop paradigm itself (the headline design work).
- Transport mechanism (socket vs shared-memory ring vs other).
- Buffer-sharing mechanism (memfd/mmap now; dmabuf/GPU later).
- Where the protocol contract lives (new sovereign lib — name + repo).
- Sequencing: protocol-design-first, or co-design compositor + client.
- Whether dhancha keeps any host-Linux backend for pre-compositor dev, or
  develops purely against a locally-run aethersafha.
