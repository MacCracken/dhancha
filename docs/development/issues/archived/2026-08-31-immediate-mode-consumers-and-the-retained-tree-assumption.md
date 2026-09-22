# Four dhancha features now assume a retained widget tree, and crab can use none of them — please read this before GRID, COLUMNS and TREE

**Status:** 🟢 **ADDRESSED in 0.9.24 — the general fix landed, and the design guidance stands.**
The four existing cases are fixed at their cause: `dh_widget_set_key` gives a widget an identity that
survives a frame-arena rewind, `dh_surface_set_root` re-binds focus/hover/press/drag from it, drag
works under an arena, and `dh_text_attach` hands the edit buffer to the app. **The ask for GRID,
COLUMNS and TREE is unchanged and is why this file stays here**: keys make retained state
*recoverable*, they do not make a widget the right place to *keep* it. A tree's expanded set is still
app state, and should still be caller-owned by construction.
**Original status:** 🟡 OPEN — a design note, filed *before* the widgets it is about are written.
**Filed:** 2026-08-31, by **crab**.
**Affects:** dhancha **0.9.23**. Nothing is broken; four features are simply unreachable for one
consumer, and the next three are about to be designed.
**Severity:** **Low today, high if the pattern repeats a fifth time**, because M5 and M6 need exactly
the widgets most likely to repeat it.

## The pattern

crab rebuilds its **entire widget tree every frame** and renders after every handled event.
`crab_render` opens with `dh_frame_begin()`, which rewinds the frame arena and clears dhancha's
retained pointers. That is not a crab quirk — it is dhancha 0.9.15's own stated rule:

> *cross-frame widget identity and a per-frame arena are mutually exclusive by construction*

Every feature that holds a **widget pointer across frames**, or stores per-widget state in
`dh_falloc`'d memory, is therefore unreachable from an immediate-mode app. So far:

| feature | what it retains | crab's answer |
|---|---|---|
| `dh_dispatch` | a press as `_dh_press` / `_dh_hover` — a widget pointer matched on release | tracks **pane index + row index**, its own model (operator ruling 2026-08-27) |
| drag | `_dh_drag_src`, same shape | own drag state; 0.9.21 fixed the *symptom* by refusing to start under an arena |
| `TEXTINPUT` | `dh_text_new` allocates its buffer **globally** so it survives frames, but the widget holding it is `dh_falloc`'d and dies at `dh_frame_begin` — so an immediate-mode app must call it per frame and leaks a buffer per frame into an allocator with no `free()` | owns the edit buffer, length and caret (0.7.5) |
| `dh_list` selection | the selection lives on the widget, destroyed each frame | `sel_l` / `sel_r` are app state |

**Four features, one assumption.** Each was worked around correctly in crab, and each workaround is
the right answer *for crab* — the point of filing this is not to ask for them back.

## The ask

M5 and M6 need **GRID**, **COLUMNS** (miller) and **TREE**, plus a menu **bar**. Verified absent from
`dist/dhancha.cyr` on 2026-08-31 (`dh_grid_new`, `dh_columns_new`, `dh_tree_new`: 0 hits each) — so
they are still design, which is the cheap moment for this note.

All three are **stateful by nature**, which is exactly the shape that has produced the mismatch four
times:

- a **grid** wants a selected cell,
- **columns** wants a selection per column *and* a scroll offset per column,
- a **tree** wants an expanded/collapsed set — the most retained state of the three, and the one that
  would hurt most to rebuild from the app side without help.

⇒ **Please design the state as caller-owned from the start.** The shape that works for both kinds of
consumer is the one `dh_cols_*` already uses and that crab's panes rely on: dhancha supplies
**geometry and painting**, the caller supplies **an opaque state record it owns across frames**. Two
concrete forms, either fine:

1. `dh_tree_draw(spec, state_ptr, ...)` — state is the caller's allocation, dhancha only reads and
   writes through the pointer; or
2. an explicit "the toolkit holds this" mode that an arena'd caller can decline, so the retained path
   still exists for retained apps.

⚠ What does **not** work is a per-widget buffer allocated inside the widget, however it is allocated:
globally it leaks once per frame, on the arena it dies mid-use. `TEXTINPUT` is the worked example.

## What crab is not asking for

- Not a rewrite. `dh_dispatch`, drag and `TEXTINPUT` are correct for a retained consumer and crab has
  working answers for all three.
- Not a change to 0.9.15's rule. The per-frame arena is the reason a crab frame costs the global heap
  **zero bytes**, which took four dhancha releases and four crab ones to achieve. The rule is right.

The request is only that the **next three** widgets be designed knowing that at least one first-party
consumer cannot hold a widget pointer between frames — before that is discovered a fifth time, in the
widget with the most state.

## Reference

crab's ruling and its reasoning: `crab/docs/development/roadmap.md` § *Rules that outlive their
milestone*, and `crab/docs/development/handoff.md` § *Pointer input — and the ruling that shaped it*.
