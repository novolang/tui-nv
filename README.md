# tui-nv

A **terminal user interface** is a program that draws a whole screen of
characters rather than printing lines: an editor, a file browser, a
system monitor. This package is the drawing half of one in novo-lang. It
divides the screen into rectangles, renders widgets into a buffer of
character cells, and works out the smallest set of changes between one
frame and the next. Its model is
[ratatui](https://ratatui.rs)'s. It writes nothing to a terminal: the
bytes are built into a list the caller owns and the caller writes them.
It is built on [ansi-nv](https://novo-lang.org/packages/ansi-nv) for the
style model and the escape sequences, and
[textwrap-nv](https://novo-lang.org/packages/textwrap-nv) for wrapping a
paragraph.

## What it is

A terminal shows a rectangle of **cells**, each holding one character
and a set of display attributes — bold, underline, a foreground colour,
a background colour. A **buffer** here is a rectangle of cells with the
area it covers, and a **frame** is one buffer, fully rendered, ready to
be shown.

A program that sent the whole buffer every frame would work, flicker,
and saturate a slow connection. The alternative is the **damage model**:
keep the buffer that was drawn last time, compare it with the new one,
and send only the cells that differ. Each such cell is a **patch** — a
position and the cell that now belongs there. `tuibuf.diff` is that
comparison and `tuibuf.patch_bytes` turns a list of patches into escape
sequences.

Laying out the screen is the other half. A **constraint** says how much
of one axis a piece wants, and `tuiarea.split` takes a rectangle and a
list of constraints and answers one rectangle per constraint. There is
no tree and no widget that owns an area: a program builds a whole screen
by calling `split` again on the pieces it got back.

| Constraint | What it asks for |
| --- | --- |
| `TuiFixed(n)` | Exactly `n` cells. A status bar is `TuiFixed(1)` |
| `TuiPercentage(p)` | `p` percent of the axis, rounded down |
| `TuiRatio(a, b)` | The fraction `a/b`. A third is not 33 percent, and three of them are not 99 |
| `TuiMinimum(n)` | At least `n` cells, and more if there is room |
| `TuiMaximum(n)` | At most `n` cells, and fewer if there is not room |
| `TuiFill(w)` | Whatever is left, shared with the other fills in proportion to the weights |

A **widget** here is a value and a function, not an object. `TuiBlock`
describes what a bordered box looks like and `tuiwidget.render_block`
draws one into a buffer. Nothing is retained between frames, so nothing
can be stale. That is what immediate mode means, and it is why the whole
of the drawing is arithmetic.

| Widget | What it is |
| --- | --- |
| Block | A bordered, titled region, with a footer and per-part styles |
| Paragraph | Text, wrapped or clipped, with a scroll position |
| List | A vertical list with an optional selection and its own offset |
| Table | Sized columns, a header that does not scroll, and rows |
| Gauge | A progress bar with a label drawn on it |
| Tabs | A row of labels with one selected and a divider between them |
| Scrollbar | A track and a thumb whose length is the viewport over the content |

| Quantity | Value |
| --- | --- |
| Cells in a buffer | `area.width * area.height`, row-major; 1920 for an 80 by 24 screen |
| Border styles | 7: none, plain, rounded, double, thick, quadrant, ASCII |
| Constraints | 6 |
| Alignments | 3: start, middle, end |
| Width of a continuation cell | 0, and its glyph is 0 |

No function in this package performs input or output.

## Install

```
novo pkg add tui-nv
```

## Example

A whole frame: layout, render, diff, and the one write the program
makes.

```novo
use sgr
use widths
use tuiarea
use tuibuf
use tuiwidget

// One frame: a tab row, a bordered body with a list inside it, and
// the rest of the screen below.  The list view holds `items` as a list
// it may write into, so they arrive as a `var` parameter.
fn frame(area: tuiarea.TuiRect, var items: [Str], selected: Int) -> tuibuf.TuiBuffer
    let w = widths.unicode_width()

    // One rectangle becomes three: a tab row, the body, a status line.
    let rows = tuiarea.split(area,
        tuiarea.layout(TuiVertical, [TuiFixed(1), TuiFill(1), TuiFixed(1)]))

    var buf = tuibuf.buffer_new(area)
    buf = tuiwidget.render_tabs(buf, list.get(rows, 0),
                                tuiwidget.default_tabs(["files", "search"]), w)

    let block = tuiwidget.titled_block(" files ")
    let body = list.get(rows, 1)
    buf = tuiwidget.render_block(buf, body, block, w)

    // Everything inside a block is drawn into the block's inner area.
    let inner = tuiwidget.block_inner(block, body)

    var l = tuiwidget.default_list(items)
    l.selected = Some(selected)
    // The program updates its own scroll position; the widget does not.
    l.offset = tuiwidget.list_offset_for(l, inner.height)
    tuiwidget.render_list(buf, inner, l, w)

fn main() [io]
    let area = tuiarea.rect(0, 0, 80, 24)

    // The frame the terminal is showing, and the one to show next.
    let previous = tuibuf.buffer_new(area)
    let current = frame(area, ["one", "two", "three"], 2)

    // The cells that differ, in screen order.
    let patches = tuibuf.diff(previous, current)
    // The escape sequences for them, appended to an empty list.
    let flush = tuibuf.patch_bytes([], patches, sgr.attrs_default())

    // The program writes `flush.bytes` in one call, so the update
    // cannot tear, and starts the next frame from `flush.style`.
    println(str.from_int(list.len(flush.bytes)) + " bytes")
```

Build and test with `novo pkg build` and `novo test`.

## What the package contains

| Module | Contents |
| --- | --- |
| `tuiarea` | The rectangle and its arithmetic — intersection, union, margin, clamping, centring — and the constraint solver that divides one rectangle into several. |
| `tuibuf` | The cell, the buffer and the operations on it, the diff of two buffers into patches, and the builder that turns patches into escape sequences and reports the style the terminal was left in. |
| `tuiwidget` | The seven widgets as values with a render function each, the border glyphs, and the hit tests that say what a click landed on. |

## How to choose an entry point

**`tuibuf.diff` and `tuibuf.patch_bytes` are the ordinary pair.** Render
a frame, diff it against the last one, build the bytes, write them once.

**`tuibuf.buffer_bytes` sends a whole buffer unconditionally.** It is
the first frame, and the frame after a resize or after the program was
suspended and resumed, where what the terminal is showing is not known.

**`tuibuf.diff_region` restricts the comparison to a rectangle**, for a
caller that knows only one part of the screen changed.

**`tuibuf.damage_rect` is the coarse answer**: the smallest rectangle
covering every patch. A caller may decide to redraw that rectangle
rather than emit ten thousand cursor moves.

**`tuiarea.split` takes a list of constraints, and `tuiarea.split_at`
takes one offset.** The second is for a caller that wants a header off
the top and does not want to build a list to say so.

**`tuiwidget.block_inner` is called before anything is drawn inside a
block.** A border costs a cell on each side it is drawn on, and the
inner rectangle is what every other widget is given.

**`tuibuf.buffer_text` reads a buffer back as lines of text**, styles
discarded. It is how a frame is asserted on in a test, and how a program
copies the screen.

## The rules a user needs

1. **Nothing here writes.** `tuibuf.patch_bytes` and
   `tuibuf.buffer_bytes` append to the `[u8]` they are given and return
   it inside a `TuiFlush`. The caller writes it, and writing a whole
   frame in one call is what stops the update from tearing.
2. **Carry `TuiFlush.style` into the next frame.** A frame's bytes leave
   the terminal in whatever attribute state the last patch needed, and
   there is no way to ask a terminal what state it is in. Passing the
   previous frame's style is the difference between one attribute change
   per run of similar cells and one per cell. After a reset, pass
   `sgr.attrs_default()`.
3. **`diff` answers patches in screen order**, row by row and left to
   right. `patch_bytes` relies on it: it emits a cursor move only when
   the next patch is not where the last one left the cursor.
4. **Two identical buffers diff to nothing.** That is the property the
   model rests on. `tuibuf.cell_eq` is the comparison `diff` is written
   in terms of, and it is public so that a caller's own check gives the
   same answer.
5. **Buffers of different areas diff over what they share, and every
   cell of the new one outside the old one is a patch.** A buffer that
   grew has cells that have never been drawn.
6. **Positions here are zero-based; escape sequences are one-based.**
   The conversion happens in exactly one place, inside `patch_bytes`. A
   caller writing its own cursor moves does it itself.
7. **A wide character occupies two cells and the second is a
   continuation**, with glyph 0 and width 0, carrying the same style so
   a background colour runs across both halves. A renderer that skipped
   it would leave the old contents showing through the right half, and a
   diff that did not know about it would report the second cell as
   changed every frame.
8. **How many columns a character occupies is a parameter.** Every
   function that draws takes a `widths.WrapWidth`, and one rule measures
   a whole frame. `widths.unicode_width()` is UAX #11's East Asian Width
   with the zero-width marks; `widths.monospace_width()` measures every
   character as one column. A frame whose paragraphs are measured under
   one rule and whose titles are measured under another disagrees with
   itself about where a column is.
9. **A scrollbar is drawn from characters UAX #11 calls Ambiguous.**
   The box-drawing track, the full block and the four arrows are one
   column on a terminal configured for Western text and two on one
   configured for East Asian text. `tuiwidget.render_scrollbar` takes
   the answer from the rule it is given, and leaves out a glyph the rule
   does not give one column: a bar one column wide has no room for the
   continuation cell a two-column glyph takes.
10. **A block's border costs one cell on each side it is drawn on**,
    whatever the rule says about its glyphs, because that is the cost
    `tuiwidget.block_inner` answers with and a layout is built on it.
11. **The order `tuiarea.split` satisfies constraints in is a promise.**
    The margin comes off, then the spacing between pieces. `TuiFixed`,
    `TuiPercentage` and `TuiRatio` take what they asked for, in the
    order they were written, until the axis runs out. `TuiMinimum` takes
    its minimum and `TuiMaximum` takes the lesser of its maximum and
    what is left. `TuiFill` shares the remainder in proportion to the
    weights. Any cell left over by the rounding goes to the last piece
    that can grow, which is the last `TuiFill` or, where there is none,
    the last `TuiMinimum`.
12. **The last rule is the one that matters.** Without it a layout gains
    and loses a cell as the window is resized, and the whole screen
    shifts by one.
13. **A run of `TuiRatio` constraints divides the axis exactly.** Each
    one takes the cells between the running total before it and the
    running total after it, so three thirds of a hundred columns are 33,
    33 and 34.
14. **`split` always answers one rectangle per constraint, in the same
    order.** A constraint there was no room for gets an empty rectangle
    rather than being left out, so a caller may index the answer against
    the list it passed in. `tuiarea.spec_min` says how many cells a
    whole layout needs, for a program that would rather refuse a window
    too small than draw something illegible.
15. **The program owns a widget's state.** A list's selection and its
    scroll offset are fields of `TuiListView`.
    `tuiwidget.list_offset_for` is a function the program calls to
    update its own field. A widget that scrolled by itself would
    disagree with the program about where the list is.
16. **The hit tests come with the widgets.**
    `tuiwidget.list_item_at`, `tuiwidget.tab_at`,
    `tuiwidget.scrollbar_position_at` and `tuiwidget.table_columns` turn
    a position back into what is drawn there.
17. **Drawing outside a buffer's area does nothing.**
    `tuibuf.buffer_set` outside the area is silently ignored, and every
    widget clips through `tuiarea.rect_intersection`. A widget one cell
    too wide produces a slightly wrong picture rather than corrupting
    what is beside it or panicking.
18. **A wide character that would straddle the right edge is left out
    entirely** rather than half-drawn. `tuibuf.buffer_set_str` reports
    the column it stopped at, which is where the next styled run starts.
19. **`tuibuf.buffer_resize` keeps the cells that are in both areas.**
    That is what makes the frame after a resize a small diff rather than
    a whole screen.
20. **A gauge's ratio is clamped to 0.0 and 1.0.** A gauge that drew
    past its area on a ratio of 1.2 would corrupt what is beside it.
21. **There is no trait for a widget, and none is planned.** A program
    that wants a mixed list of things to draw writes an enum of its own
    widgets and one `match`.

## What is not included

- **Anything that writes.** See rule 1.
- **The escape sequences.** ansi-nv builds them.
  `tuibuf.patch_bytes` is the only function here that knows one exists,
  and it uses ansi-nv's writer.
- **Input.** [keymap-nv](https://novo-lang.org/packages/keymap-nv)
  decodes keys and mouse reports. A widget here says what a click landed
  on; deciding what that means is the program's.
- **A retained widget tree, focus, and event routing.** All three need
  state that survives a frame, and a program that has them has its own
  model of them. What this package gives such a program is a layout and
  a renderer.
- **Themes.** A theme is a set of `SgrAttrs` and belongs in a program's
  configuration.
- **Scrolling text with its own history.** That is a terminal emulator,
  and [novo-vte](https://novo-lang.org/packages/novo-vte) is the model
  for one.
- **A microcontroller build of its own.** Nothing here performs input
  or output, so nothing in the package prevents one, but there is no
  program in the tree that builds for a device and no claim that a
  buffer of 1920 cells and a widget surface of `Str` fits on one. A
  device with a display has a framebuffer rather than a terminal.

## Related packages

- [ansi-nv](https://novo-lang.org/packages/ansi-nv) is the style model
  and the writer. A `TuiCell` carries an `SgrAttrs`, so the style a diff
  hands to `seqwrite.sgr_change` needs no conversion.
- [textwrap-nv](https://novo-lang.org/packages/textwrap-nv) wraps text
  at a display width, which is the whole of what the paragraph widget
  does that a plain string write does not. Its `wrapped_height` is what
  a layout asks before deciding how tall to make a paragraph.
- [keymap-nv](https://novo-lang.org/packages/keymap-nv) is the other
  half of an interactive program: the bytes a terminal sends when a key
  is pressed or the mouse moves, decoded into events.
- [termios-nv](https://novo-lang.org/packages/termios-nv) owns the
  terminal — raw mode, the alternate screen, the window size — and is
  what writes the bytes this package builds.
- [novo-vte](https://novo-lang.org/packages/novo-vte) keeps a grid of
  cells that a stream of escape sequences changes. It is a terminal
  emulator's model. This package's buffer is drawn by a program rather
  than by incoming bytes.
- `std.tui` in the standard library writes escape sequences straight to
  standard output: it clears the screen, moves the cursor and wraps a
  string in colour codes. It is for a program that wants a coloured
  line, not a frame.

## Reference implementations

ratatui is the reference for the whole shape: immediate mode, the
constraint layout, the cell buffer, the diff, and the widget vocabulary.
tui-rs before it is where the damage model comes from. textual is where
the idea that hit-testing belongs with the widget that drew the thing
being hit comes from.

`TuiCell` is a struct rather than a packed integer, which is the one
deliberate difference from a terminal emulator's own grid: an emulator
packs the same information into two machine words because it holds a
hundred thousand cells, and pays for the packing in every accessor. Here
the field names are the documentation, and a consumer that needs the
packing writes it over this.

## Tests

```bash
novo test --isolate tests/tuiarea_tests.nv    # 18 tests: geometry and the solver
novo test --isolate tests/tuibuf_tests.nv     # 18 tests: the buffer and the damage model
novo test --isolate tests/tuiwidget_tests.nv  # 19 tests: the widgets and the hit tests
novo test --isolate tests/edges_tests.nv      # 29 tests: the answers at the edges
```

`tuiarea_tests.nv` asserts the solver's promise: that the pieces always
add up to the rectangle, that a ratio is not a percentage, that a
minimum is satisfied before a fill takes anything, that a maximum gives
back what it does not need, that margin and spacing come off first, and
that a constraint with no room gets an empty rectangle. It also checks
that centring rounds towards the top left and that a popup hanging off
the edge is moved rather than clipped.

`tuibuf_tests.nv` asserts that two identical buffers diff to nothing,
that one changed cell is one patch, that the patches come out in screen
order, that a style change alone is a change, that a wide character
takes two cells with a continuation after it, that a resize keeps the
cells in both areas, and that the flush reports the style the terminal
was left in.

`tuiwidget_tests.nv` asserts that a border costs a cell on every side it
is drawn on, that the ASCII border uses no box-drawing characters, that
a paragraph reports its wrapped height before it is drawn, that a list
keeps its selection visible without moving it, that a table's columns
are solved the way the screen is, that a gauge clamps a ratio outside
its range, and that a scrollbar's thumb is never shorter than one cell.

`edges_tests.nv` asserts the answers at the edge of each promise: a
layout with no constraints, a ratio with a denominator of zero, a
rectangle clamped into bounds that hold nothing, a combining mark that
takes no cell, a byte that begins no UTF-8 sequence, a buffer that grew,
a cell that went blank, every widget handed an empty area, a title wider
than its block, the same title measured under two width rules, a
scrollbar along either axis and with its end arrows drawn, a scrollbar
under a rule that calls its glyphs two columns, and the four border sets
no program in the tree draws with yet.

No test needs a terminal. Every line of `src/` is executed by the four
suites; `bash tests/coverage.sh` merges the per-suite coverage and
prints the number.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
