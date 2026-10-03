# table

Beautiful rounded CLI tables for ZZ. Pure ZZ, zero dependencies.

```toml
zz add table
```

```zz
import table

func main() {
    table.new(["NAME", "AGE"])
        |> table.add_row(["Alice", "25"])
        |> table.add_row(["Bob", "30"])
        |> table.render()
        |> println()
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

The builder is a plain value. The idiomatic style is a `|>` pipeline
with a single binding — no rebind chain, no mutation to forget:

```zz
t := table.new(["NAME", "AGE"])
    |> table.add_row(["Alice", "25"])
    |> table.set_title("Users")
```

(Dotted chaining like `t.add_row(..).set_title(..)` parses and runs
on the VM. It needs a post-0.1.6 toolchain natively (method-dispatch
fixes landed in dev after 0.1.6); until your `zz` ships them, prefer
pipelines for anything that runs native. Plain
`t = table.add_row(t, …)` rebinding keeps working everywhere.)
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
- `from(rows, headers, f)` — build from any row type via a mapper
  closure; the `std.sqlz` bridge, pipeline-first argument order.
- `column(rows, f)` / `from_cols(headers, cols)` — build column by
  column, then transpose (short columns pad with `""`).
- `set_style` / `set_align` / `set_align_all` / `set_padding` /
  `set_title` / `set_footer` / `set_row_lines` — builders above.
- `set_max_width` / `set_col_max_width` — truncation caps (`0` = off).
- `from_json(headers, text)` — JSON array-of-arrays / array-of-objects.
- `render(t)` — string (trailing newline included).
- `print(t)` — `println(render(t))`.
- `styles()` — the seven style names.
- `visible_len(s)` — visible width (handy for your own layout next
  to a table).

Cells are single-line by design: embedded `\n`/`\t` fold to spaces so
one bad cell can never break the grid.

## `std.sqlz` support

Query rows flow into tables through one mapper closure. `from` takes
rows first, so `|>` pipelines read top to bottom:

```zz
import std.sqlz
import table

struct User { id: int, name: str }

mydb := sqlz.open(":memory:")
users: [User] = mydb.query("""SELECT id, name FROM users""")

users
    |> table.from(["ID", "NAME"], |u: User| [str(u.id), u.name])
    |> table.render()
    |> println()
```

Unannotated queries (`row{c0, c1, …}` dicts) work the same way with an
untyped mapper: `|r| [str(r.c0), str(r.c1)]`. Two smaller builders
compose column by column:

```zz
ids := table.column(users, |u: User| str(u.id))
t := table.from_cols(["ID", "NAME"], [ids, names])
```

See `examples/sqlz_demo.zz` (`cd examples && zz install && zz run
sqlz_demo.zz`) for filter-then-display pipelines.

## Truncation

```zz
t = table.set_max_width(t, 20)      // every column, 0 = off (default)
t = table.set_col_max_width(t, 2, 12) // one column, overrides global
```

Overlong cells truncate to a clean `…` (`"a very long…"`), ANSI-safe:
colors survive truncation and a reset is re-appended, so styling
never bleeds into borders. Widths, padding, and alignment all use the
post-truncation size.

## JSON

```zz
t := table.from_json(["id", "name"], "[{\"id\": 1, \"name\": \"a\"}]")
```

Array-of-arrays keep cell order; array-of-objects pick columns in
`headers` order (missing keys render blank, pass `[]` to derive
headers from the first object's keys). Numbers keep their text,
booleans print true/false, null and nesting render blank. Total:
invalid JSON yields a headers-only table, never an error.

## Tests

`zz test` runs 32 checks: golden rounded output, per-column
alignment (incl. sticky `set_align_all`), footer separator, title,
all seven glyph sets, fallback paths, uneven rows, empty tables,
newline folding, ANSI-safe widths, bulk APIs, row lines, padding
clamps, five `std.sqlz` bridge checks (struct rows, `|>` pipelines,
unannotated rows, `column`/`from_cols`, empty results), plus eleven
v0.2.0 checks (truncation goldens, per-column caps, colored
truncation without bleed, markdown/minimal truncation, JSON shapes,
derived headers, invalid-JSON empties, pipeline fluency).
