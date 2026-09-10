# Changelog

All notable changes to tui-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
