# table

Beautiful rounded CLI tables for ZZ. Pure ZZ, zero dependencies.

```toml
[dependencies.table]
path = "…"
# or: zz add table
```

```zz
import table

func main() {
    t := table.new(["NAME", "AGE"])
    t = table.add_row(t, ["Alice", "25"])
    t = table.add_row(t, ["Bob", "30"])
    println(table.render(t))
}
```

```
╭───────┬─────╮
│ NAME  │ AGE │
├───────┼─────┤
│ Alice │ 25  │
│ Bob   │ 30  │
╰───────┴─────╯
```

The builder is a plain value: every call hands back the next state,
so re-bind it (`t = table.add_row(t, …)`) on each step.
See `examples/demo.zz` (`cd examples && zz install && zz run demo.zz`).

## One-liner

```zz
println(table.simple(["A", "B"], [["1", "2"]]))
```

## Styles

```zz
t = table.set_style(t, "rounded")  // default ╭─╮││╰─╯
t = table.set_style(t, "sharp")    // ┌─┐││└─┘
t = table.set_style(t, "heavy")    // ┏━┓┃┃┗┛
t = table.set_style(t, "double")   // ╔═╗║║╚╝
t = table.set_style(t, "markdown") // | pipes + --- separator
t = table.set_style(t, "minimal")  // no frame, ─ under header
t = table.set_style(t, "ascii")    // +-+|| fallback for dumb terminals
```

Unknown names fall back to `"rounded"`. `table.styles()` lists all seven.

## Alignment, padding, rows

```zz
t = table.set_align(t, 1, "right")   // "left" | "center" | "right", per column
t = table.set_align_all(t, "center") // every column at once
t = table.set_padding(t, 2)          // spaces per side, clamped 0..8
t = table.set_row_lines(t, true)     // separator between every row
t = table.add_rows(t, rows)          // bulk append
t = table.from_rows(headers, rows)   // build in one call
```

## Title and footer

```zz
t = table.set_title(t, "Stock")      // centered above the table
t = table.set_footer(t, ["Total", "14"])
```

The footer renders after its own separator, before the bottom border —
the natural place for totals. Markdown renders the title as `# …`.

## Colors

Pass `std.colors` output straight in. Widths are measured on the
visible text (ANSI `\e[…m` sequences count as 0), so columns stay
aligned under any styling:

```zz
import std.colors

t := table.new([colors.bold("NAME"), colors.bold("AGE")])
t = table.add_row(t, ["Alice", colors.green("25")])
table.print(t) // render + println in one call
```

## Functions

- `new(headers)` — empty table with headers.
- `from_rows(headers, rows)` — table with all rows at once.
- `simple(headers, rows)` — build and render immediately.
- `add_row(t, row)` / `add_rows(t, rows)` — append (short rows pad
  with `""`, long rows render whole, never panics).
- `set_style` / `set_align` / `set_align_all` / `set_padding` /
  `set_title` / `set_footer` / `set_row_lines` — builders above.
- `render(t)` — string (trailing newline included).
- `print(t)` — `println(render(t))`.
- `styles()` — the seven style names.
- `visible_len(s)` — visible width (handy for your own layout next
  to a table).

Cells are single-line by design: embedded `\n`/`\t` fold to spaces so
one bad cell can never break the grid.

## Tests

`zz test` runs 16 checks: golden rounded output, per-column
alignment (incl. sticky `set_align_all`), footer separator, title, all seven glyph sets, fallback
paths, uneven rows, empty tables, newline folding, ANSI-safe widths,
bulk APIs, row lines, and padding clamps.
