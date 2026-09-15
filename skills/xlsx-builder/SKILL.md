---
name: xlsx-builder
description: Use when the user wants an Excel workbook (.xlsx) — one or more sheets with a header row, column widths, number and date formats, formulas, totals, a frozen header or an autofilter
tools: code_execute, bash
---
# Building an Excel workbook

An `.xlsx` is a zip of XML parts. The generator below writes one with nothing
but Python's standard library, from a small spec — so it runs in the
`code_execute` container as it is, no packages, no network. It produces a
clean workbook: bold header row on a light-blue fill, frozen, with real
number formats and real formulas Excel calculates on opening.

## 1. Shape the data first

Decide the sheets and their columns before generating. One table per
sheet, the header in row 1, one row per record — no merged cells, no
titles above the table, no blank separator rows; that is what makes a
sheet sortable and filterable. Numbers stay numbers (`1200000`, not
`"1.2 M"`): the column's format does the display. Dates as `YYYY-MM-DD`
strings in a column with a date format become real Excel dates.

## Where the file lands

`code_execute` runs in a container. Read the tool's own description before
choosing the path:

- It says the host directory is mounted **WRITABLE** at `/mnt/host` → write to
  `/mnt/host/<name>.pptx`; that is the user's directory, the file is theirs
  the moment the call returns. Say where it is.
- It says the mount is READ-ONLY, or mentions no mount → a file written to
  `/workspace` stays inside the sandbox where nobody can open it. Then, if
  the agent has `bash` and the host has `python3`, run the very same script
  through `bash` (`python3 - <<'PY' … PY`) with a path in the working
  directory instead. If neither is possible, say so and offer the content as
  Markdown; do not pretend a file was delivered.

Never pip-install anything for this: the generator below is standard library
only, so it runs in the plain `python` image without network.

## 2. The spec

```python
spec = {
  "path": "/mnt/host/sales-2026.xlsx",   # see above
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

One `code_execute` call, language `python`: the generator verbatim, then the
spec, then `build(spec, spec["path"])`. Keep the generator unchanged; change
only the spec. It prints `wrote <path>: N sheet(s)` on success.

```python
import json, re, sys, zipfile
from datetime import date, datetime
from xml.sax.saxutils import escape

XML = '<?xml version="1.0" encoding="UTF-8" standalone="yes"?>'
NS_MAIN = 'xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main" xmlns:r="http://schemas.openxmlformats.org/officeDocument/2006/relationships"'
DATE_RE = re.compile(r"^\d{4}-\d{2}-\d{2}$")

def col_letter(n):  # 1 -> A
    s = ""
    while n: n, r = divmod(n - 1, 26); s = chr(65 + r) + s
    return s

def serial(d):  # Excel 1900 date system
    return (d - date(1899, 12, 30)).days

class Styles:
    """cellXfs: 0 default, 1 header, then one per number format, and a bold variant of each."""
    def __init__(self):
        self.formats = {}   # fmt code -> numFmtId
        self.xfs = [("", False), ("", True)]   # (fmt, bold) -> index in order
    def xf(self, fmt, bold=False):
        key = (fmt or "", bold)
        if key not in self.xfs: self.xfs.append(key)
        return self.xfs.index(key)
    def fmt_id(self, fmt):
        if not fmt: return 0
        builtin = {"0": 1, "0.00": 2, "#,##0": 3, "#,##0.00": 4, "0%": 9, "0.00%": 10}
        if fmt in builtin: return builtin[fmt]
        if fmt not in self.formats: self.formats[fmt] = 164 + len(self.formats)
        return self.formats[fmt]
    def xml(self):
        for fmt, _ in self.xfs: self.fmt_id(fmt)
        numfmts = "".join(f'<numFmt numFmtId="{i}" formatCode="{escape(f, {chr(34): "&quot;"})}"/>' for f, i in self.formats.items())
        xfs = []
        for fmt, bold in self.xfs:
            fid = self.fmt_id(fmt)
            is_header = (fmt, bold) == ("", True) and self.xfs.index((fmt, bold)) == 1
            xfs.append(f'<xf numFmtId="{fid}" fontId="{1 if bold else 0}" fillId="{2 if is_header else 0}" borderId="{1 if is_header else 0}" xfId="0"'
                       + (' applyNumberFormat="1"' if fid else "") + (' applyFont="1"' if bold else "") + (' applyFill="1" applyBorder="1"' if is_header else "") + "/>")
        return (f'{XML}<styleSheet xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main">'
                + (f'<numFmts count="{len(self.formats)}">{numfmts}</numFmts>' if self.formats else "")
                + '<fonts count="2"><font><sz val="11"/><name val="Calibri"/></font><font><b/><sz val="11"/><color rgb="FF1F3864"/><name val="Calibri"/></font></fonts>'
                '<fills count="3"><fill><patternFill patternType="none"/></fill><fill><patternFill patternType="gray125"/></fill>'
                '<fill><patternFill patternType="solid"><fgColor rgb="FFD9E2F3"/><bgColor indexed="64"/></patternFill></fill></fills>'
                '<borders count="2"><border><left/><right/><top/><bottom/><diagonal/></border>'
                '<border><left/><right/><top/><bottom style="thin"><color rgb="FF1F3864"/></bottom><diagonal/></border></borders>'
                '<cellStyleXfs count="1"><xf numFmtId="0" fontId="0" fillId="0" borderId="0"/></cellStyleXfs>'
                f'<cellXfs count="{len(xfs)}">{"".join(xfs)}</cellXfs>'
                '<cellStyles count="1"><cellStyle name="Normal" xfId="0" builtinId="0"/></cellStyles></styleSheet>')

def cell_xml(ref, value, style, fmt):
    s = f' s="{style}"' if style else ""
    if value is None or value == "": return f'<c r="{ref}"{s}/>'
    if isinstance(value, bool): return f'<c r="{ref}"{s} t="b"><v>{int(value)}</v></c>'
    if isinstance(value, (int, float)): return f'<c r="{ref}"{s}><v>{value}</v></c>'
    if isinstance(value, (date, datetime)): return f'<c r="{ref}"{s}><v>{serial(value if isinstance(value, date) else value.date())}</v></c>'
    text = str(value)
    if text.startswith("="): return f'<c r="{ref}"{s}><f>{escape(text[1:])}</f></c>'
    if fmt and "y" in fmt.lower() and DATE_RE.match(text): return f'<c r="{ref}"{s}><v>{serial(date.fromisoformat(text))}</v></c>'
    return f'<c r="{ref}"{s} t="inlineStr"><is><t xml:space="preserve">{escape(text)}</t></is></c>'

def sheet_xml(sheet, styles):
    cols = sheet.get("columns", [])
    rows = list(sheet.get("rows", []))
    header = [c["header"] if isinstance(c, dict) else str(c) for c in cols] if cols else None
    fmts = [(c.get("format") if isinstance(c, dict) else None) for c in cols] if cols else []
    all_rows = ([header] if header else []) + rows
    ncols = max([len(r) for r in all_rows] + [len(cols)]) if all_rows else 1
    body = []
    for ri, row in enumerate(all_rows, 1):
        is_header = header is not None and ri == 1
        cells = []
        for ci, v in enumerate(row, 1):
            fmt = fmts[ci - 1] if ci - 1 < len(fmts) else None
            bold = is_header or (isinstance(v, dict) and v.get("bold"))
            if isinstance(v, dict): fmt = v.get("format", fmt); v = v.get("value")
            style = 1 if is_header else (styles.xf(fmt, bold) if (fmt or bold) else 0)
            cells.append(cell_xml(f"{col_letter(ci)}{ri}", v, style, fmt))
        body.append(f'<row r="{ri}">{"".join(cells)}</row>')
    widths = "".join(f'<col min="{i}" max="{i}" width="{c.get("width", 14)}" customWidth="1"/>'
                     for i, c in enumerate(cols, 1) if isinstance(c, dict)) if cols else ""
    dim = f"A1:{col_letter(ncols)}{max(len(all_rows), 1)}"
    freeze = ('<sheetViews><sheetView workbookViewId="0"' + (' tabSelected="1"' if sheet.get("_first") else "") + '>'
              '<pane ySplit="1" topLeftCell="A2" activePane="bottomLeft" state="frozen"/></sheetView></sheetViews>') \
        if header and sheet.get("freeze_header", True) else ""
    autofilter = f'<autoFilter ref="{dim}"/>' if header and sheet.get("autofilter", False) and len(all_rows) > 1 else ""
    return (f'{XML}<worksheet {NS_MAIN}><dimension ref="{dim}"/>{freeze}<sheetFormatPr defaultRowHeight="15"/>'
            + (f"<cols>{widths}</cols>" if widths else "") + f'<sheetData>{"".join(body)}</sheetData>{autofilter}</worksheet>')

def build(spec, path):
    sheets = spec["sheets"]; styles = Styles()
    sheet_parts = []
    for i, sh in enumerate(sheets, 1):
        sh["_first"] = i == 1
        sheet_parts.append(sheet_xml(sh, styles))
    names = [escape(sh.get("name", f"Sheet{i}"))[:31] for i, sh in enumerate(sheets, 1)]
    ct = (f'{XML}<Types xmlns="http://schemas.openxmlformats.org/package/2006/content-types">'
          '<Default Extension="rels" ContentType="application/vnd.openxmlformats-package.relationships+xml"/>'
          '<Default Extension="xml" ContentType="application/xml"/>'
          '<Override PartName="/xl/workbook.xml" ContentType="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet.main+xml"/>'
          '<Override PartName="/xl/styles.xml" ContentType="application/vnd.openxmlformats-officedocument.spreadsheetml.styles+xml"/>'
          '<Override PartName="/docProps/core.xml" ContentType="application/vnd.openxmlformats-package.core-properties+xml"/>'
          + "".join(f'<Override PartName="/xl/worksheets/sheet{i}.xml" ContentType="application/vnd.openxmlformats-officedocument.spreadsheetml.worksheet+xml"/>' for i in range(1, len(sheets) + 1))
          + "</Types>")
    rels = ('<Relationships xmlns="http://schemas.openxmlformats.org/package/2006/relationships">'
            '<Relationship Id="rId1" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/officeDocument" Target="xl/workbook.xml"/>'
            '<Relationship Id="rId2" Type="http://schemas.openxmlformats.org/package/2006/relationships/metadata/core-properties" Target="docProps/core.xml"/></Relationships>')
    wb_rels = ('<Relationships xmlns="http://schemas.openxmlformats.org/package/2006/relationships">'
               + "".join(f'<Relationship Id="rId{i}" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/worksheet" Target="worksheets/sheet{i}.xml"/>' for i in range(1, len(sheets) + 1))
               + f'<Relationship Id="rId{len(sheets) + 1}" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/styles" Target="styles.xml"/></Relationships>')
    wb = (f'{XML}<workbook {NS_MAIN}><sheets>'
          + "".join(f'<sheet name="{n}" sheetId="{i}" r:id="rId{i}"/>' for i, n in enumerate(names, 1))
          + '</sheets><calcPr calcId="191029" fullCalcOnLoad="1"/></workbook>')
    core = (f'{XML}<cp:coreProperties xmlns:cp="http://schemas.openxmlformats.org/package/2006/metadata/core-properties" '
            'xmlns:dc="http://purl.org/dc/elements/1.1/" xmlns:dcterms="http://purl.org/dc/terms/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">'
            f'<dc:title>{escape(spec.get("title", ""))}</dc:title><dc:creator>{escape(spec.get("author", ""))}</dc:creator></cp:coreProperties>')
    with zipfile.ZipFile(path, "w", zipfile.ZIP_DEFLATED) as z:
        z.writestr("[Content_Types].xml", ct); z.writestr("_rels/.rels", XML + rels)
        z.writestr("xl/workbook.xml", wb); z.writestr("xl/_rels/workbook.xml.rels", XML + wb_rels)
        z.writestr("xl/styles.xml", styles.xml()); z.writestr("docProps/core.xml", core)
        for i, part in enumerate(sheet_parts, 1): z.writestr(f"xl/worksheets/sheet{i}.xml", part)
    print(f"wrote {path}: {len(sheets)} sheet(s)")

spec = { ... }          # your spec here
build(spec, spec["path"])
```

## 4. Check and report

After the call, confirm the file exists and is not empty
(`import os; print(os.path.getsize(path))` in a second call, or `ls -l` via
`bash` on the host path) and tell the user the path, the sheets and their
row counts. If the run fails, read the traceback: a `KeyError` or
`TypeError` names the row that is wrong in the spec — usually a dict cell
without `value`, or a sheet without `rows`.
