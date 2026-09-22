# Every `[deps.*]` carried a live `path = "../sibling"`, so `cyrius.lock` could never pin a dependency's commit — and a green build was not evidence the declared tag resolved

**Status:** 🟢 **ADDRESSED in 0.10.3 — all five `path` lines are dormant (commented out, one note
each), every dep resolves from its published git tag, and `cyrius.lock` carries the commit each tag
resolved to** (`commit d9f41f3… sadish 0.11.2`, `aa1a2d7… rupa 0.1.7`, `47aabba… rekha 0.9.0`,
`f8f9c97… kashi 1.0.10`, `cc4161f… setu 0.8.9`; `37 deps locked, 5 commit-pinned`). The lock is the
declared closure now, the state CI has always built from (`rm -rf lib && cyrius deps`), not the dev
box's full stdlib snapshot. Uncommenting one line for cross-repo work on an unpushed sibling is the
documented dev move, measured both ways: with `../sadish` live the lock reads `4 commit-pinned`, and
the tag-only resolve is byte-identical to the sibling checkout for all five (0.10.1 verified that
file by file). The manifest's `[deps]` header says why the lines stay dormant in a commit.
**Original status:** 🟡 OPEN — noted at 0.10.1, not a defect in any build, a gap in what a build proves.
**Filed:** 2026-09-21, by the 0.10.1 release.
**Affects:** dhancha 0.9.23 (the first `path` line) through 0.10.2.
**Severity:** **Low until a tag drifts, then silent.** It already happened once: setu's tag said
0.8.7 while `../setu` was 0.8.8 and every local build and test was green; CI, with no `../setu`,
resolved the tag and `dist/dhancha.cyr` would not compile (the `[deps.setu]` note in `cyrius.cyml`).

## What was found

`cyrius deps` prefers `path` to `tag`, and a `path` dep pins no commit: with a live `path` the lock
cannot carry that dep's `commit` line (0.10.0's changelog recorded the rekha line disappearing when
its `path` arrived). All five deps carried one by 0.10.0, each for a good reason at the time — a
floor that landed in a sibling minutes before the release that needed it, before the tag was pushed
— and each block carried a ⛔ note saying *"`path` WINS over `tag`, so a green build here is NOT
evidence the declared tag resolves — re-verify with this line disabled before any release that
moves the tag"*. 0.10.1 did that verification by hand (disable all five, `rm -rf lib`, resolve,
compare hashes, re-enable) and wrote the result into the changelog. A step that has to be done by
hand before every release, and whose omission is invisible, is a gap.

rekha removed its own `path` line at 0.3.11 for exactly this reason (`rekha/cyrius.cyml`,
`[deps.sadish]`: *"while it was here cyrius.lock could never carry a sadish `commit` line and a green
local build was not evidence the tag resolves"*), and its CI refuses a manifest with a live `path =`
line.

## What closed it

- `cyrius.cyml`: the five `path` lines are `# path = "../sibling"`, each under a one-line note;
  the `[deps]` header carries the rule and the history. The ⚠ paragraphs that justified each line
  stay as the record of why it exists.
- `cyrius.lock`: regenerated from a clean `lib/` — 37 entries (the declared stdlib closure plus the
  five dep bundles) and five `commit` lines. The full-snapshot lock of 0.10.0–0.10.2 (115–116
  entries) was the dev box's `lib sync --full` hygiene; CI never had it.
- `README.md` *Dependencies* says the same in two sentences.

## What did not change

The sibling checkouts on the dev box are at their tags (kashi one comment-only commit past its),
so a build against `path` and a build against `tag` are the same bytes today. Nothing in `src/` or
`dist/` moved for this.
