# dhancha — Roadmap

> **Last updated:** 2026-09-21, at **0.10.3**.
>
> This file tracks **forward-facing work only**. A finished item leaves. What already shipped is in
> [`CHANGELOG.md`](../../CHANGELOG.md), release by release, with the measurement each claim rests
> on; what the toolkit *is* today is [`README.md`](../../README.md). The direction the display seam
> follows is [`sovereign-desktop.md`](sovereign-desktop.md), opened 2026-07-06 and still the locked
> direction (Wayland is refused, not ported).
>
> Every item below carries a `path:line` or a grep, because an item nobody can check is a wish.
> Filed issues live in [`issues/`](issues/) while open and in [`issues/archived/`](issues/archived/)
> once a release addresses them; a filing that leaves a standing ask behind carries that ask here.

## Where the toolkit stands

The client side is done and proven on Linux: widget tree, layout, event model, draw through sadish +
rekha + kashi + rupa, the frame arena, and the setu connection — `dh_client_connect` /
`dh_client_present` / `dh_client_next_event` / `dh_client_poll_event` / `dh_client_close`
(`src/dh_client.cyr`) over setu's reference client. A consumer (crab) renders a real face with a
frame costing the global heap **0 B** (0.10.0), a label is one bounded flatten operation (0.10.2),
and every dependency resolves from a published tag with its commit in `cyrius.lock` (0.10.3).

What remains is **scheduled** (a booked bite with a consumer waiting), **pinned** (real, evidenced,
unscheduled), **blocked on a sibling**, or an explicit **non-goal**.

---

## Scheduled

### The compositor-connected loop

`dh_run` (`src/event.cyr:793`) pumps an in-memory `DhQueue` until it drains; its own header says
what replaces that: *"the compositor-connected loop (block on the native protocol fd, decode wire
events into DhEvents, run layout + present when dirty) replaces the empty-check branch with an fd
wait; the dispatch core is the same"* (`src/event.cyr:790-792`), and `src/event.cyr:17-20` still
calls the fd source *"a later bite"* and names the queue as the seam it will feed. The two halves it
needs already exist: `dh_client_next_event` / `dh_client_poll_event` decode a setu frame into a
`DhEvent` (`src/setu_input.cyr`, `dh_setu_read_event` / `dh_setu_poll_event`), and
`dh_client_present` renders and submits (`src/dh_client.cyr`). What does not exist is the loop that
joins them for a **retained-mode** app: wait on `dh_client_fd(c)`, drain with `dh_client_poll_event`
into `dh_dispatch`, re-layout and present when the surface is dirty (`DH_S_DIRTY`,
`src/surface.cyr`), and honour `dh_quit`. crab does not need it — an immediate-mode app rebuilds
its tree per frame and drives `dh_client_poll_event` itself, which is the loop `dh_client.cyr`'s
header prescribes for *"an app that redraws on its own"* — so this ships when a retained-mode
consumer asks, and its gate is the discipline `sovereign-desktop.md` names: synthetic frames in,
decoded events and one present out, headless (`programs/poll_test.cyr` is the shape).

### TREE, and COLUMNS as a browser — designed with caller-owned state

The standing ask of the archived filing
[`issues/archived/2026-08-31-immediate-mode-consumers-and-the-retained-tree-assumption.md`](issues/archived/2026-08-31-immediate-mode-consumers-and-the-retained-tree-assumption.md):
a widget that is *stateful by nature* must keep that state where a per-frame-arena consumer can hold
it — an opaque record the caller owns across frames, dhancha supplying geometry and painting (the
shape `dh_cols_*` uses, `src/table.cyr:29`), or a toolkit-held mode an arena'd caller can decline.
0.9.24's keys made retained state *recoverable* (`dh_widget_set_key`, `src/widget.cyr`); they do not
make the widget the right place to *keep* it. Of the three widgets the filing named: **GRID** shipped
at 0.9.25 (`src/grid.cyr`); **COLUMNS** shipped at 0.9.20 as the shared width spec a header and its
rows agree on (`src/table.cyr` — a table, not a miller browser); a menu **bar** is `dh_list_new_h`
(0.9.26, `src/list.cyr`). Still absent: **TREE** (an expanded/collapsed set — the most retained
state of the three) and a **miller-columns browser** (a selection and a scroll offset *per column*).
`grep -cE 'fn dh_tree_' src/*.cyr` → 0. Neither is booked until a consumer names the state it
needs; when one does, the design rule above is the whole of the requirement.

---

## Pinned — real, evidenced, unscheduled

- **The draw reads bytes, not scalars.** Both branches of `dh_draw_text_ink` take
  `load8(text + i)` as the codepoint (`src/surface.cyr:476`, and the advance pre-pass at `:419`),
  so a UTF-8 multibyte character draws as its two to four bytes — Latin-1 glyphs under a face, CP437
  cells under kashi. The canvas-slack measurement is honest about it (*"cp 32..255, what `load8` can
  reach"*, `src/surface.cyr`). `TEXTINPUT` is UTF-8 throughout for editing and the caret
  (`src/textinput.cyr:14`, `dh_text_utf8_len`, `dh_text_char_index`) — the buffer is right, the draw
  of it is not. Closing it is a decoder in the two loops and a `rekha_char_to_glyph` that takes the
  scalar; the kashi arm stays byte-wise by construction (CP437 is a byte font). Unscheduled because
  nothing in the stack draws a non-ASCII label yet; it reopens the day one does.
- **No kerning.** `dh_text_advance` (`src/surface.cyr:218`) is `rekha_char_advance_px` per
  codepoint. rekha publishes pair kerning — `rekha_kern_pair_px`, `rekha_gpos_kern_pair`
  (`lib/rekha.cyr`, 0.6.5 / 0.6.6) — and nothing here reads it. A run's width and its caret would
  both move; `text_test`'s `hmtx` expectations and `text_arena_test` group G are the gates that
  would say so.
- **The one global cost that survives warm-up.** sadish's per-row accumulator is re-allocated from
  the GLOBAL heap, never the hook, at exactly `w*8` whenever a canvas WIDER than any before arrives
  (`text_arena_test` B2: 6,176 B once for a 772 px run, 0 on the repeat). Documented at
  `src/surface.cyr` (*"THE ONE GLOBAL COST THAT SURVIVES WARM-UP"*) and sadish's, not ours; a
  zero-heap gate warms up at the widest run it will draw.
- **A fixed arena that cannot hold a frame spills, silently.** `dh_falloc` (`src/widget.cyr:305`)
  falls back to `alloc()` when the arena refuses, by design (a 0 faults several layers away);
  `text_arena_test` B3 pins the identity `spill + arena_used == the frame`. The advice stands:
  `arena_new_growable`. A loud mode (refuse, or count the spills) is a fair ask nobody has made.
- **One budget per label is a trade with a measured edge.** A single label longer than ~1,889
  glyphs at a 32 px em (~649 at 208 px — sadish's figures) degrades to chords under the default
  `SD_FLATTEN_BUDGET_DEFAULT`; the suite's widest legitimate run spends 1.5 % of it
  (`programs/text_budget_test.cyr`, group B). A consumer past the edge raises
  `sd_flatten_budget_set` or scopes a frame itself. `dh_text_degraded_runs()` is the witness.

---

## Blocked on a sibling

- **GPU present — mabda.** The present path is CPU shared-buffer by decision
  (`sovereign-desktop.md` §*The seam*, item 3; `cyrius.cyml`: *"mabda (GPU) + … are a later bite —
  CPU draw first"*). `dh_client_present` sends pixels (`setu_client_present`); a GPU upload
  (a mabda texture / dmabuf-shaped handle over setu) needs setu to carry it and aethersafha to
  composite it before dhancha has anything to call. `mabda` appears in `src/` only in two comments,
  never a call, by design.
- **A proportional face on the target.** crab's other blocker on a real face is agnos-owned
  (`agnos/docs/development/issues/2026-09-13-no-proportional-face-on-the-target.md`, named in the
  archived 2026-09-13 filing's *Related*). dhancha's side closed at 0.10.0; a face arriving is what
  makes it observable.

---

## Out of scope — committed

- **Wayland.** Not ported, not shimmed: no `wl_*`, no `xdg-shell`, no `wl_shm`, no copy of puka's
  host-Hyprland client. `sovereign-desktop.md` §*Locked direction* and §*Explicitly NOT doing*.
- **The desktop paradigm.** Window model, shell, how agents present surfaces — the compositor's and
  the founder's; dhancha is the toolkit underneath whatever it is (`sovereign-desktop.md` §*"Own
  the desktop"*).
- **Text shaping.** GSUB, complex scripts, BiDi — a shaping library's job; rekha is the glyph-data
  provider and dhancha draws runs. (Pair kerning is the one positioning feature rekha already
  answers, and it is pinned above, not here.)
- **Naming a colour.** Widgets and chrome draw with rupa's tokens (`dh_theme_*`, `src/theme.cyr`);
  a widget that would make an app paint its own highlight is refused a kind on that ground
  (CHANGELOG 0.9.23, 0.9.25).

## What would legitimately reopen an item here

- A retained-mode consumer that wants `dh_run` on the transport fd → *The compositor-connected loop*.
- A consumer naming the state a TREE or a miller browser must keep → the design rule above.
- A non-ASCII label anywhere in the stack → *The draw reads bytes, not scalars*.
- setu carrying a GPU buffer handle → *GPU present*.

## Where to look for…

- **what shipped and what it measured** — `CHANGELOG.md`, newest first; every figure has its probe.
- **the display-seam direction** — `docs/development/sovereign-desktop.md`.
- **filed issues** — `docs/development/issues/` (open) and `issues/archived/` (addressed, with the
  release that did).
- **the dependency floors and why each is hard** — `cyrius.cyml`, the `[deps.*]` blocks.
- **the gates** — `programs/*_test.cyr`, every one a self-checking RUN test; `.github/workflows/ci.yml`
  runs them all, plus lint / fmt / vet / distlib / sidecar.
