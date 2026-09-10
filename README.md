# tui-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

Layout, widgets and the damage model for a terminal user interface —
and **the writer belongs to the caller**.

- `tuiarea` — a rectangle, and a constraint solver that divides one;
- `tuibuf` — a cell buffer, and the diff that turns two of them into
  the smallest update;
- `tuiwidget` — block, paragraph, list, table, gauge, tabs, scrollbar,
  each a pure function from a value and an area into a buffer.

Nothing here writes to a terminal. `tuibuf.patch_bytes` **builds**
bytes into a `[u8]` the caller owns, using ansi-nv's writer, and the
caller writes them — once.

```
novo pkg add tui-nv
novo pkg build
novo test
```

## The one example that will work — the whole loop

```novo
use sgr
use seqwrite
use widths
use tuiarea
use tuibuf
use tuiwidget

fn frame(area: tuiarea.TuiRect, items: [Str], selected: Int) -> tuibuf.TuiBuffer
    // 1. LAYOUT.  One rectangle becomes three.
    let rows = tuiarea.split(area,
        tuiarea.layout(TuiVertical, [TuiFixed(1), TuiFill(1), TuiFixed(1)]))

    // 2. RENDER.  Every widget is a pure function into the buffer.
    var buf = tuibuf.buffer_new(area)
    buf = tuiwidget.render_tabs(buf, list.get(rows, 0),
                                tuiwidget.default_tabs(["files", "search"]),
                                widths.monospace_width())

    let block = tuiwidget.titled_block(" files ")
    let body = list.get(rows, 1)
    buf = tuiwidget.render_block(buf, body, block)

    var l = tuiwidget.default_list(items)
    l.selected = Some(selected)
    l.offset = tuiwidget.list_offset_for(l, tuiwidget.block_inner(block, body).height)
    tuiwidget.render_list(buf, tuiwidget.block_inner(block, body), l,
                          widths.monospace_width())

fn main() [io]
    let area = tuiarea.rect(0, 0, 80, 24)
    var previous = tuibuf.buffer_new(area)
    var style = sgr.attrs_default()

    loop
        // 3. DIFF.  Two buffers become the cells that changed.
        let current = frame(area, current_items(), current_selection())
        let patches = tuibuf.diff(previous, current)

        // 4. BYTES.  Built into a list, not written.
        let flush = tuibuf.patch_bytes([], patches, style)

        // 5. THE ONE EFFECT IN THE LOOP.  The program writes, once —
        //    which is what stops a screen update from tearing.
        write_bytes(flush.bytes)

        previous = current
        style = flush.style
```

Steps 1 to 4 are `[]`. Step 5 is the program's. That is the whole
argument for the layer.

## The load-bearing interface

```novo ignore
pub fn diff(prev: TuiBuffer, next: TuiBuffer) -> [TuiPatch]
pub fn patch_bytes(out: [u8], patches: [TuiPatch], from: SgrAttrs) -> TuiFlush

pub struct TuiFlush
    bytes: [u8]
    style: SgrAttrs      // what the terminal was left in
```

**The damage model is the package, and the writer is not in it.**

A terminal program that redraws the whole screen every frame works,
flickers, and saturates a slow link. The fix every program reinvents is
to remember what it drew last time and send only what changed — which
is exactly two buffers and a comparison.

- `diff` returns patches in **screen order**, row by row, left to
  right. That is not cosmetic: it is what lets `patch_bytes` emit a
  cursor move only when the next patch is not where the last one left
  the cursor.
- `patch_bytes` takes the buffer to append to and returns it. Nothing
  is written, so a frame can be rendered, diffed and **asserted on**
  with no terminal anywhere — and a caller can batch a whole frame into
  one write, which a screen update has to be.
- `TuiFlush.style` is the half a caller forgets. A frame's bytes leave
  the terminal in whatever attribute state the last patch needed, the
  next frame has to start from it, and there is no way to ask a
  terminal what state it is in. Getting this wrong is the difference
  between one `CSI m` per run of cells and one per cell.
- Two identical buffers diff to **nothing**. If they did not, every
  frame would be a full redraw and the model would be a slower way to
  do what it replaced.

`patch_bytes` is the one function here that knows an escape sequence
exists, and it knows because ansi-nv's writer is doing the work. It is
in this package rather than left to every consumer because it is ten
lines that every consumer would otherwise write, and one of the ten —
the style transition — is the line they would get wrong.

## The two dependencies, and why each

Both are `core`, so `dep-layer` holds either way. Neither is a
convenience.

**`ansi-nv`, for the style model.** A `TuiCell` carries an `SgrAttrs`,
so the diff a caller hands to `seqwrite.sgr_change` needs no
conversion. A colour model of tui-nv's own would mean translating every
changed cell at the boundary — in the one loop where the cost is per
cell — and a consumer that used both packages would hold two
vocabularies for the same thing. `patch_bytes` also uses
`seqwrite.cursor_to` and `seqwrite.sgr_change` directly, which is the
second half of the same argument.

**`textwrap-nv`, for the paragraph widget.** Wrapping at a display
width is the whole of what a paragraph does that `buffer_set_str` does
not, and `wrapped_height` is what a layout asks before it decides how
tall to make one. Re-implementing it here would be a second wrap that
disagrees with the first about where a line breaks.

Both are `{ path = "../<name>" }` in this tree, and the coordinator
converts each to `^0.0.1` at publish — a path dependency is refused by
`novo pkg publish`, which is the rule that stops a published tarball
from sending every consumer to a directory on somebody else's machine.

## A widget is a value and a function, not an object

There is no `trait Widget` and no widget that holds a buffer. A
`TuiBlock` is what a block looks like; `render_block` draws one. That
is ratatui's immediate-mode model, and here there is a second reason
for it: `dyn Trait` is charged the union of every impl in the program
(SPEC § 5.6), so one consumer with an effectful widget would make this
whole surface effectful and cost the package its `core` budget.

A program that wants a heterogeneous list of things to draw writes an
enum of its own widgets and one `match` — the same code, with the cost
where the program can see it.

**The state a widget needs is on the value, and the program owns it.**
A list's selection and scroll offset are fields of `TuiListView`, not
something the widget remembers. `list_offset_for` is a function the
program calls to update its own field, rather than a hidden update: a
widget that silently scrolled would disagree with the program about
where the list is, which is how a selection ends up half a screen from
where the arrow keys think it is.

**The hit tests are part of the widget set.** `list_item_at`,
`tab_at`, `scrollbar_position_at` and `table_columns` are the half that
usually gets left out, and leaving it out is why every terminal program
has its own wrong arithmetic for "which row did the user click".

## The layer, and why

`core` — no effects. Layout is arithmetic, rendering is a function into
a buffer, diffing is a comparison, and the bytes go into a list.

**No `@tier(embedded)` claim, and none is intended.** A cell buffer for
an 80×24 screen is 1920 structs; the widget surface is `Str`
throughout; and a device with a display has a framebuffer rather than a
terminal. The audit's `core-embedded` row passes as "makes no device
claim", which is the honest reading. ansi-nv and keymap-nv — the two a
device driving a serial console genuinely wants — make the claim and
build it.

## How this relates to novomux, and what the split would cost

novomux has a layout (`compute_split_geom` over a `[Split]` tree), a
damage model (`render_diff` between two `grid.Grid`s) and a widget set
(`emit_box`, `emit_status_bar`, `emit_label`) — and all three are
welded to a terminal. `render_diff` writes bytes to stdout as it walks;
`emit_*` are `[io, ffi]` because emitting *is* writing; the geometry is
in one-based terminal coordinates with a status row subtracted, so
`Geom` cannot describe a rectangle that is not a novomux pane.

This package is what those three become when the writer is taken out of
them, and the taking-out is the cost:

| novomux today | here |
| --- | --- |
| `render_diff` writes as it walks | `diff` returns patches; `patch_bytes` builds; the caller writes |
| `emit_*` are `[io, ffi]` | every render is `[]` |
| `Geom` is a pane in a 1-based screen with a status row | `TuiRect` is a 0-based rectangle |
| `[Split]` is a binary tree of panes | `[TuiConstraint]` is a flat list, applied recursively |
| the grid is `novo-vte`'s packed cells behind `std.array` handles | `TuiBuffer` is a list of value cells |

The `[io, ffi]` rows are not novomux's fault: they come from
`std.array`'s externs declaring both labels for calls that only move
memory, which is filed against the toolchain. But the writing is
novomux's, and it is in every function.

## Where the names come from, and the ones that were taken

Public type and variant names are unique across the whole assembly, so
a package's names have to be unique across the registry too. This
package had the most to change of the four in this lane, because every
noun it wants is a noun something else already uses.

| here | the obvious name | why not |
| --- | --- | --- |
| `TuiRect` | `Rect` | `Rect` is a **standard-library struct** (`std.canvas`) and a `std.window` variant — a package may not redeclare either |
| `TuiCell` | `Cell` | novobook declares `Cell`; `CellPool` and `CellMesh` are standard-library types |
| `TuiBuffer` | `Buffer` | `Buffer` is a standard-library struct |
| `TuiConstraint`, `TuiLayoutSpec`, `TuiDirection` | `Constraint`, `Layout`, `Direction` | all three are certain to collide, and `Layout` is the one four packages will want |
| `TuiListView`, `TuiTableView` | `List`, `Table` | `List` is the language's own list; `TableDef` and `TableStats` are already published by sql-engine-nv |
| `TuiRow` | `Row` | csv-nv publishes `Row` |
| `TuiPatch`, `TuiFlush`, `TuiWrite` | `Patch`, `Flush`, `Write` | `Write` is a standard-library **trait**; the other two are generic enough to collide |
| `TuiAlign` | reuse textwrap-nv's `WrapAlign` | different question — one places a title in a border, the other aligns the lines of a wrapped paragraph — and reusing it would make this package's public surface change whenever textwrap-nv's did |
| `TuiHorizontal`, `TuiFixed`, `TuiAlignStart`, … | `Horizontal`, `Fixed`, `Left`, … | enum **variants** collide by bare name; `Left` and `Right` are exactly the kind that bites, and `Linear`, `Start`, `End` and `Current` are all standard-library variants |
| `TuiAlignStart` / `TuiAlignMiddle` / `TuiAlignEnd` | `TuiAlignLeft` / `Centre` / `Right` | a horizontal name for something a scrollbar also uses vertically was worse than a neutral one |
| module `tuiarea`, `tuibuf`, `tuiwidget` | `layout`, `buffer`, `widget` | `tui` is a **standard-library module**, so the package cannot own its own name; the other three are names another package will want |

That list is longer than the other three packages' put together, and it
is evidence rather than a complaint: every entry exists because a type's
identity is its bare name across the whole assembly.

## The reference implementation

**ratatui** for the whole shape: immediate mode, the constraint layout,
the cell buffer, the diff, and the widget vocabulary — block,
paragraph, list, table, gauge, tabs, scrollbar. **tui-rs** before it
for the damage model. **textual** for the observation that hit-testing
belongs with the widget that drew the thing being hit. **novomux** for
what a terminal multiplexer actually needs from a layout, which is
where `TuiFill`'s weights and the leftover-cell rule come from.

Deliberately left out, and where it went instead:

- **Anything that writes.** `patch_bytes` builds; the caller writes.
- **The escape sequences.** ansi-nv.
- **Input.** keymap-nv. A widget here reports what a click landed on;
  deciding what that means is the program's.
- **A retained widget tree, focus, and event routing.** All three need
  state that survives a frame, and a program that has them has its own
  model of them. What this package owes such a program is a layout and
  a renderer.
- **Themes.** A theme is a set of `SgrAttrs` and a program's
  configuration. novomux's `theme.nv` is what one looks like, and it is
  a program's file rather than a library's type.
- **Scrolling text with its own history.** That is a terminal emulator,
  and novo-vte is the model for one.

## Status

Every function is `todo()`. Three suites, all red, all for the same
reason — every assertion reaches `not implemented: tui-nv.<fn>`, which
is the expected result until the bodies land.

```
novo test --isolate tests/tuiarea_tests.nv     # geometry and the constraint solver
novo test --isolate tests/tuibuf_tests.nv      # the cell buffer and the damage model
novo test --isolate tests/tuiwidget_tests.nv   # the widgets, and the hit tests
```

`novo doc` renders and its four examples compile.
