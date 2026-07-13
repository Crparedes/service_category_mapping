# SIRP service category report

## Layout

```
report.qmd          the report
data/                the 8 ABox files (AUV, EM, L, M, PR, RI, T, TF)
reference/quantities.ttl   SI Digital Framework quantities graph (units + names source)
schema/kcdb-sc.ttl   TBox / vocabulary (not required to render, kept for reference)
annotations/         <code>_annotations.csv per area -- the persistent, hand-editable
                     confirmation_status/comment store; see "Two files, not one" below
generated/           written on each render: <code>_service_categories.csv per area
```

## Requirements

R packages: `rdflib`, `dplyr`, `tibble`, `tidyr`, `stringr`, `readr`, `purrr`,
`fs`, `htmltools`. Install with:

```r
install.packages(c("rdflib", "dplyr", "tibble", "tidyr", "stringr", "readr",
                    "purrr", "fs", "htmltools"))
```

Plus [Quarto](https://quarto.org).

## Rendering

```
quarto render report.qmd
```

Produces `report.html` (self-contained), always refreshes
`generated/<code>_service_categories.csv`, and creates
`annotations/<code>_annotations.csv` **only** the first time (see below).

## Two files, not one

Each area's browser table is one combined view, but it's backed by two CSVs
with different lifetimes:

- **`generated/<code>_service_categories.csv`** -- fully derived from the
  source graphs (plus `quantities.ttl`) on every render. Always safe to
  overwrite, and always overwritten. One row per (service category, related
  SI quantity) pair -- a category with no related quantity at all keeps one
  row with those columns blank rather than being dropped; a category with
  several related quantities gets one row per quantity.

- **`annotations/<code>_annotations.csv`** -- just `mapping_id, notation,
  related_si_quantity, confirmation_status, comment`. This is the
  persistent, single source of truth for confirmation status/comments, and
  **the render never overwrites it once it exists** -- not to fix a typo,
  not to append a newly-appeared mapping, nothing. It's written exactly
  once, the first time it's missing, as a starting point you can edit
  directly (in a spreadsheet app, a text editor, whatever) or replace
  wholesale via the in-browser download button. This is what the earlier
  version got wrong: editing a file in `generated/` and having it
  overwritten next render was expected behaviour for that folder (it's
  always fully regenerated) but the wrong place to have been editing in the
  first place -- `annotations/` is the one that's meant to survive edits.

`mapping_id` (`notation::related_si_quantity`) is the explicit join key
shared by both files -- simpler to match on by hand than an implicit
two-column key, and it's what keeps annotations correctly attached to their
quantity even as the underlying data changes: if a service category later
gains an additional related quantity, that quantity gets its own new
`mapping_id`, defaulting to `"unknown"` / `""` in both the report and
`generated/`, entirely independent of the existing rows in
`annotations/<code>_annotations.csv` -- nothing there gets touched, moved,
or reordered by the new arrival.

Columns depend on which classification scheme the area uses -- hardcoded per
area in `area_scheme` (setup chunk of `report.qmd`), not inferred from the
data:

- `"generic"` (`AUV`, `EM`, `L`, `M`, `PR`, `T`, `TF`), walking
  `IndividualService → Subservice → Service → Branch`:
  `mapping_id, notation, branch, service, subservice, individual_service,
  status, related_si_quantity, quantity_name, si_unit, confirmation_status,
  comment`.
- `"section"` (`RI`, ionizing radiation), which doesn't use that chain at
  all — its own `skos:note`s reference CCRI conventions specifically:
  `mapping_id, notation, individual_service, section, medium, source,
  quantity_axis, status, related_si_quantity, quantity_name, si_unit,
  confirmation_status, comment`.

To add a future area with a new scheme (e.g. `QM`), add its entry to
`area_scheme` and a matching case to the `switch()` right after it in
`parse_area()` — rendering will stop with a clear error if either is
missing, rather than silently misclassifying it.

`quantity_name` and `si_unit` are both resolved fresh from
`reference/quantities.ttl` on every render (not stored in either CSV).

In the browser, each table is sortable (click a column header, toggles
ascending/descending) and filterable (the global search box, substring
match across the whole row, plus a per-column filter row -- see "Per-column
filters" below).

## Quantity names and unit resolution

`reference/quantities.ttl` gives each SI quantity kind (e.g. `DENS`) an
`si:hasUnit`, either a direct unit (`units:gray`) or a compound tree built
from `si:UnitProduct` (a × b) and `si:UnitPower` (base ^ exponent) blank
nodes. `report.qmd` walks that tree recursively into symbol/exponent pairs
and formats it (e.g. `Pa·s·m⁻³` for acoustic impedance). `quantity_name` is
built the same way from `skos:prefLabel@en` instead — `density` for `DENS`,
with each factor of a genuine composite parenthesised for readability, e.g.
`(mass)·(time⁻¹)` (a single atomic name stays plain, no parentheses).

Some `relatedSIQuantity` codes used in the ABox files are themselves
composite — e.g. `MASS.TIME-1`, `PRES.VOLU.TIME-1` — a dot-separated product
of base quantity codes, each optionally suffixed `-N` for a negative
exponent. These aren't entries in `quantities.ttl`; the resolver parses the
code itself and combines each component's own unit and name, so
`EMIR.AREA-1` correctly comes out as `s⁻¹·m⁻²` / `(emission rate)·(area⁻¹)`.
Note that composite names are a literal symbolic join of the component
names, not idiomatic English phrasing ("emission rate per area") — composing
that "per" scoping correctly in natural language is exactly the open
metrological-linguistics question, so this deliberately doesn't take a
stance on it. Treat `quantity_name` as a starting point to hand-adjust for
anything that needs to read naturally, not a finished gloss.

Three outcomes are distinguished for `si_unit` (and analogously for
`quantity_name`, minus the dimensionless case — every named quantity has a
label):
- a resolved unit string (e.g. `kg·m⁻³`)
- `"1"` for a quantity that exists in `quantities.ttl` but genuinely has no
  `si:hasUnit` — a dimensionless ratio quantity (relative humidity,
  refractive index, etc.)
- blank/`NA` for a code that isn't in `quantities.ttl` at all. Across the
  current files this is only `HUCO` and `PMDP` — worth checking whether
  that's an expected gap or a missing quantities.ttl entry.

## Recording CC confirmation status

Each table is editable in the browser (status dropdown + comment cell) on
any row that has a related quantity; a row for a service category with no
related quantity at all leaves those two cells blank and non-editable, since
there's nothing to confirm and no `mapping_id` to attach an edit to. Two
ways to actually persist an edit:

1. **Edit `annotations/<code>_annotations.csv` directly** -- in a
   spreadsheet app or a text editor, keyed by `mapping_id`. This file is
   never touched by the render process once it exists, so nothing you type
   there is ever at risk of being overwritten by `quarto render`.
2. **Edit in the browser, then download and replace.** Since a static HTML
   file can't write back to disk, click **"Download updated annotations CSV
   for `<code>`"** after editing, then replace
   `annotations/<code>_annotations.csv` with the download. This is a
   manual, deliberate replacement -- the render itself will never do this
   for you.

Either way, on the next `quarto render`, `annotations/<code>_annotations.csv`
is read and joined onto the freshly parsed quantity list by `mapping_id` --
not by row position, and not merged/appended automatically. A quantity
mapping no longer present in the source graph just doesn't appear in the
report anymore (its row in `annotations/` is left alone, unused, in case the
mapping reappears later); a mapping that's new since the annotations file
was last touched shows up with default values (`"unknown"`, `""`) in the
report and in `generated/<code>_service_categories.csv`, but doesn't get
written into `annotations/` until you explicitly add it there yourself.

## Page layout: tabs instead of a TOC

There's no table of contents anymore — `toc: false` in the YAML. Instead,
`report.qmd` wraps the per-area tables in a `::: {.panel-tabset}` fenced
div, with one tab per area labelled by its Consultative Committee acronym
(`area_cc_label` in the setup chunk: `AUV → CCAUV`, `EM → CCEM`, `L → CCL`,
`M → CCM`, `PR → CCPR`, `RI → CCRI`, `T → CCT`, `TF → CCTF`) — the area's
full official title (from `dcterms:title`) still appears as a line inside
each tab, just not in the tab label itself.

`page-layout: full` in the YAML only widens the *available* page grid --
content still defaults to normal body width unless it explicitly opts into
one of Quarto's wide-column classes. The tabset div is tagged
`.column-screen-inset` (`::: {.panel-tabset .column-screen-inset}`) so it
actually uses that width rather than sitting in the narrower default column.

Separately, the confirmation-status `<select>` was overflowing into the
comment column even inside a fixed-width table cell: form controls have an
intrinsic `min-width` based on their content (here, the longest option text,
"under discussion") that CSS `width: 100%` alone doesn't override under
`table-layout: fixed`. Fixed with an explicit `min-width: 0` on
`.status-cell select`, plus a little extra room in `col_widths["confirmation_status"]`.

## Per-column filters

Each table has a filter row right under the headers -- a text input per
column, a dropdown for `confirmation_status` since it only has three
possible values. These combine with each other and with the global search
box (all AND'd together) via one shared `applyAllFilters()` function; the
search box and column filters used to be two separate JS functions
(`filterTable()` for the global box only) but were unified so adding the
column filters didn't mean maintaining two overlapping filtering code paths.

## Row status coloring

Rows are softly tinted by `confirmation_status` -- green for confirmed,
yellow for under discussion, orange for unknown -- computed once in R at
render time (`status_row_class` in `editable_combined_table()`) and kept in
sync afterward by `updateRowStatusClass()` in JS, called on the dropdown's
`onchange`, so changing the status in-browser updates the tint immediately
without needing a re-render. The two places share the same three class
names (`row-confirmed` / `row-under-discussion` / `row-unknown`) -- if you
rename a status value or a CSS class, update it in both `status_row_class`
(R) and `ROW_STATUS_CLASS_MAP` (JS).

## Why only one tab (or, earlier, only one TOC entry) ever showed up

Root cause: Quarto's HTML format enables Pandoc's `native_divs` extension,
which tries to parse any *bare* `<div>` it finds directly in the markdown
stream into Pandoc's own AST -- rather than just passing it through as raw
text -- using a line/tag heuristic, not a full HTML parser. `render-tables`
was `cat()`-ing each area's table as plain, unfenced HTML text, and that
heuristic choked on trying to balance tags across these large single-line
`<div>` blocks (hence the repeated "Div ... unclosed ... closing implicitly"
warnings, and eventually pandoc reporting the closing `:::` itself as a
stray, unrecognised string). A heading -- or a `:::` fence -- caught inside
a div that Pandoc thinks is still open doesn't register as a real
heading/fence, so it's invisible to whatever depends on finding one: first
the TOC, then `.panel-tabset`'s own tab detection, which is exactly why all
8 tables ended up merged into a single "CCAUV" tab.

An earlier attempt fixed this by adding explicit blank lines around each
`cat()` call, on the theory that Pandoc uses blank lines to decide where an
HTML block ends -- that helped make the blocks self-contained but didn't
stop `native_divs` from trying (and failing) to parse their internal
structure in the first place. The actual fix is to tell Pandoc not to parse
that content at all: each area's table is now wrapped in an explicit
` ```{=html} ... ``` ` raw-block fence, exactly like the shared
`<style>`/`<script>` block earlier in the file already was (which is why
*that* block never produced this warning). Content inside a `{=html}` fence
is stored as a single raw block verbatim, with no div-balancing or markdown
interpretation applied to it, regardless of size or how it's nested. Nothing
inside `editable_combined_table()`'s output should ever contain a literal
` ``` ` sequence (it's all `htmltools::htmlEscape()`-processed data plus
static markup), which is the one thing that could prematurely close such a
fence.

## Why the rasqal "bnodeid1 ... bound but unused" warnings appeared

The per-area title lookup used to match `dcterms:title` via an anonymous
node (`[] dcterms:title ?title`) specifically to avoid an earlier "variable
bound but unused" warning on a named `?s` — which didn't actually help,
since rasqal just warns about its own auto-generated blank-node variable
instead (`bnodeid1`). Since every ABox file's `dcterms:title` turns out to
sit on a fixed, predictable IRI (`https://si-digital-framework.org/kcdb-sc/<code>`,
confirmed in all 8 files), the query now targets that IRI directly with no
unprojected variable at all, which removes the warning at the source rather
than papering over it.

## A note on `report.qmd`'s table-in-a-loop pattern

`render-tables` mixes `cat()` calls for markdown headings with `cat()` calls
for the raw per-area HTML table, once per area, inside a single
`results: asis` chunk. This used to be built on `DT::datatable()`, which
I've since dropped entirely in favour of the same plain HTML/JS table
already used for the editable columns (see the "columns not showing" issue
you hit earlier — DT's widget-dependency handling across repeated calls
inside one loop was the prime suspect, and this sidesteps it rather than
debugging it blind). The current version has no htmlwidgets at all, just
plain strings passed to `cat()` — a much simpler and more predictable
pattern — but it still wasn't possible to render Quarto/R directly to
confirm it end to end, so flag it back to me if anything still looks off.

## Why the SPARQL is split into so many tiny queries

An earlier version used one query per table with several `OPTIONAL` blocks
(sibling and nested). That hung on the larger files (`RI.ttl`, `EM.ttl`):
`rasqal`, the SPARQL engine behind the R `rdflib` package, doesn't reorder or
optimise joins the way a real triple store does, and several independent
`OPTIONAL`s in one query can evaluate as a near-cartesian product. The
`queries` chunk now issues only trivial, mandatory single-triple-pattern
queries with no `OPTIONAL` at all; `parse_area()` reassembles the wide table
via a chain of R-side `left_join()`s instead. This was checked for
correctness (each join is 1:1 or many:1, so nothing multiplies rows) but,
like the DT-loop pattern above, couldn't be timed/render-tested directly —
if any single query is still slow on your machine, it'll be obvious which
one from where the render pauses, and it's now easy to isolate.
