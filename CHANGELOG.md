# Changelog

All notable changes to tui-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-09-18

Layout, the cell buffer with its damage model, and the seven widgets,
over ansi-nv's style model and textwrap-nv's wrapping.

- `tuiarea` solves a layout in the order it promises.  A run of
  `TuiRatio` constraints divides the axis exactly: each takes the cells
  between the running total before it and the running total after it,
  so three thirds of a hundred columns are 33, 33 and 34 rather than
  three 33s and a lost column.  The cells the rounding leaves over go to
  the last piece that can grow, which is the last `TuiFill` or, where
  there is none, the last `TuiMinimum`.
- `tuibuf.diff` walks the new buffer's area in screen order and answers
  a patch for every cell that differs and for every cell the old buffer
  does not cover.  `patch_bytes` emits a cursor move only where the
  cursor is not already, `seqwrite.sgr_change` only where the style
  differs, and a space for a cell that went blank.  A patch whose cell
  is the continuation half of a wide character emits nothing: the half
  before it drew both cells.
- Text arrives as bytes and is drawn as codepoints.  `buffer_set_str`
  decodes UTF-8, measures each codepoint under the caller's rule, writes
  a continuation cell after a wide one, leaves out a wide character that
  would straddle the right edge, and answers the column it stopped at.
  A byte that begins no sequence draws U+FFFD rather than shifting every
  cell after it.
- The seven widgets clip through `tuiarea.rect_intersection` and draw
  nothing into an empty area, so a constraint there was no room for
  needs no special case in a caller's render loop.  `render_block` and
  `render_scrollbar` take no width rule: the first measures its title
  and its footer with textwrap-nv's `widths.unicode_width()`, and the
  second draws nothing but single-column glyphs.
- The dependency on ansi-nv moves to `^0.1.0` and the dependency on
  textwrap-nv to `^0.1.0`, which brings unicode-nv into the closure as
  textwrap-nv's own.  The toolchain floor moves to `>= 0.9.1`, the
  release the package was built, tested and measured on.
- Every call that answers an altered buffer copies the cells before it
  writes one.  A list element store reaches every holder of that list on
  this toolchain, so writing into the cells the caller handed in would
  change the caller's buffer, and the diff of a frame against the one
  before it would be empty.
- `tests/edges_tests.nv` is the fourth suite, 27 tests over the answers
  at the edge of each promise.  Every line under `src/` is executed by
  the four suites; `bash tests/coverage.sh` merges the per-suite
  coverage and prints the number.
- No published signature changed.  `TuiBlock`'s `TuiQuadrantBorder` is
  documented as a line along the outer edge of the cells it is drawn in
  rather than as a border that costs no cell: every border kind a block
  draws costs one cell on each side it is drawn on, and `block_inner`
  answers the same rectangle for all seven.

## 0.0.2 — 2026-09-16

README rewritten to the package README style guide
(docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-10

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `tuiarea` — a zero-based rectangle and a constraint solver over one
  axis: fixed, percentage, ratio, minimum, maximum and weighted fill.
  The order the constraints are satisfied in is documented as a
  promise, and the leftover cell goes to the last piece that can grow —
  which is what stops a screen from shifting by one as a window is
  resized.
- `tuibuf` — the cell buffer and the damage model. `diff` returns
  patches in screen order; `patch_bytes` builds them into a `[u8]` the
  caller owns and reports the style the terminal was left in, because
  the next frame has to start from it and nothing can ask a terminal
  what state it is in. A wide character is two cells and the second is
  a continuation.
- `tuiwidget` — block, paragraph, list, table, gauge, tabs and
  scrollbar, each a value and a render function rather than an object.
  The hit tests come with them: `list_item_at`, `tab_at`,
  `scrollbar_position_at` and `table_columns`, because the arithmetic
  belongs with the widget that drew the thing being hit.

### Known

- **Both dependencies are path dependencies in this tree** and become
  `ansi-nv = "^0.0.1"` and `textwrap-nv = "^0.0.1"` at publish. A path
  dependency is refused by `novo pkg publish`.
- **The paragraph widget's wrapping is only as good as the width
  rule.** `widths.WrapWidth` is a parameter throughout, and until
  unicode-nv lands `monospace_width()` measures a CJK ideograph as one
  column.
- **No `@tier(embedded)` claim.** A cell buffer for an 80x24 screen is
  1920 structs and the widget surface is `Str` throughout; a device
  with a display has a framebuffer rather than a terminal.
- **No trait for a widget**, and none is planned: `dyn Trait` is
  charged the union of every impl in the program, so one consumer with
  an effectful widget would cost this whole surface its `core` budget.
