---
name: xlsx-builder
description: Use when the user wants an Excel workbook (.xlsx) — one or more sheets with a header row, column widths, number and date formats, formulas, totals, a frozen header or an autofilter
tools: code_execute, bash
---
# Building an Excel workbook

A workbook is written by `mc_office.xlsx_builder`, which the `code_execute`
python image carries: you describe the sheets as a small spec, it writes the
`.xlsx`. It produces a clean workbook: bold header row on a light-blue fill,
frozen, with real number formats and real formulas Excel calculates on
opening.

## 1. Shape the data first

Decide the sheets and their columns before generating. One table per
sheet, the header in row 1, one row per record — no merged cells, no
titles above the table, no blank separator rows; that is what makes a
sheet sortable and filterable. Numbers stay numbers (`1200000`, not
`"1.2 M"`): the column's format does the display. Dates as `YYYY-MM-DD`
strings in a column with a date format become real Excel dates.

## Where the file lands

`code_execute` works in the chat's working directory, mounted under its real
path; its description names it. Give `path` relative to it (or as that full
path) and the file is the user's the moment the call returns — they find it
in the chat's Files dialog. Tell them the full host path. Do not write to
`/mnt/host`; that mount, where there is one, is read-only.

## 2. The spec

```python
spec = {
  "path": "sales-2026.xlsx",  # relative: lands in the chat's working directory
  "title": "Sales 2026",                 # document property
  "author": "Mindconnect",
  "sheets": [
    {"name": "Revenue",                  # at most 31 characters
     "autofilter": True,                 # filter buttons on the header row
     "freeze_header": True,              # default; False to leave the header unfrozen
     "columns": [
       {"header": "Quarter", "width": 12},
       {"header": "Date",    "width": 12, "format": "yyyy-mm-dd"},
       {"header": "Revenue", "width": 14, "format": "#,##0.00"},
       {"header": "Growth",  "width": 10, "format": "0.0%"},
       {"header": "Note",    "width": 24},
     ],
     "rows": [
       ["Q1", "2026-03-31", 1200000, 0.12, "on plan"],
       ["Q2", "2026-06-30", 1400000.5, 0.167, "strong"],
       [{"value": "Total", "bold": True}, "", {"value": "=SUM(C2:C3)", "bold": True}, "=AVERAGE(D2:D3)", ""],
     ]},
    {"name": "Raw", "rows": [["no header", 1, True], ["second", 2.5, False]]},
  ]
}
```

| Field | Meaning |
|-------|---------|
| `columns` | optional; gives the header row, and per column a `width` (characters, default 14) and a `format` applied to every data cell below |
| `rows` | lists of cells. A number is a number, a bool a bool, a string text; a string starting with `=` is a **formula** (`=SUM(C2:C9)`, `=C2*D2`, `=IF(D2>0,"up","down")`); a `YYYY-MM-DD` string in a column whose format contains `y` becomes a date |
| a cell as `{"value": …, "bold": True, "format": "0.00"}` | overrides the column format or emphasises one cell — the total row, say |
| formats | Excel codes: `0`, `0.00`, `#,##0`, `#,##0.00`, `0%`, `0.0%`, `yyyy-mm-dd`, `dd.mm.yyyy`, `"€" #,##0.00`, `[h]:mm` |

Formulas use A1 references and English function names, exactly as typed
into Excel's formula bar. Row 1 is the header when `columns` is given, so
data starts at row 2. Values are written uncalculated; Excel (and
LibreOffice) calculate on opening, and a `fullCalcOnLoad` flag makes sure
of it.

## 3. Run it

One `code_execute` call, language `python`, with the spec and two lines — the
generator is part of the image as `mc_office.xlsx_builder`, so never paste or
rewrite it:

```python
from mc_office import xlsx_builder
spec = { ... }          # your spec here
xlsx_builder.build(spec, spec["path"])
```

It prints `wrote <path>: N sheet(s)` on success.

If the call fails with `ModuleNotFoundError: No module named 'mc_office'`, the
installation runs `code_execute` on a plain python image. Say so: the operator
sets `mindconnect.code-exec.languages` back to the default image
(`ghcr.io/mindconnect-ai/code-exec-python`). Offer the content as Markdown
meanwhile; do not pretend a file was delivered, and do not hand-write the
.xlsx format.

## 4. Check and report

After the call, confirm the file exists and is not empty
(`import os; print(os.path.getsize(path))` in a second call, or `ls -l` via
`bash` on the host path) and tell the user the path, the sheets and their
row counts. If the run fails, read the traceback: a `KeyError` or
`TypeError` names the row that is wrong in the spec — usually a dict cell
without `value`, or a sheet without `rows`.
