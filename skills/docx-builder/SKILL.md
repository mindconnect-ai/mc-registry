---
name: docx-builder
description: Use when the user wants a Word document (.docx) with a title, headings, bullet or numbered lists, tables, page breaks or bold/italic text — richer than the plain sections document_write produces
tools: code_execute, bash
---
# Building a Word document

A `.docx` is a zip of XML parts. The generator below writes one with nothing
but Python's standard library, from a small JSON-like spec — so it runs in the
`code_execute` container as it is, no packages, no network. Use it when the
user wants headings, lists, tables or inline emphasis; for a few plain
sections the `document_write` tool is enough and needs no code.

## 1. Draft the content first

Agree the outline with the user's request before generating: title, the
sections and what goes in each. Write real prose — a document of bullet
fragments reads like notes. Keep tables to what fits a page width (up to
about five columns).

## Where the file lands

`code_execute` runs in a container. Read the tool's own description before
choosing the path:

- It says the host directory is mounted **WRITABLE** at `/mnt/host` → write to
  `/mnt/host/<name>.docx`; that is the user's directory, the file is theirs
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
  "path": "/mnt/host/quarterly-report.docx",   # see above
  "title": "Quarterly Report",                  # optional, big title on top
  "author": "Mindconnect",                      # optional, document property
  "blocks": [
    {"type": "heading", "level": 1, "text": "Summary"},          # level 1-4
    {"type": "paragraph", "text": "Revenue grew **12 %** while costs stayed *flat*; see `Table 1`."},
    {"type": "bullets", "items": ["First point", "Second point"]},
    {"type": "numbered", "items": ["Step one", "Step two"]},
    {"type": "table", "rows": [["Quarter", "Revenue"], ["Q1", "1.2 M"]], "header": True},
    {"type": "page_break"},
  ]
}
```

Inline markup inside any text: `**bold**`, `*italic*`, `` `code` ``. Nothing
else is interpreted; `&`, `<` and `>` are plain characters. Fonts are
Calibri, headings dark blue, A4 with normal margins — a clean default the
user can restyle in Word.

## 3. Run it

One `code_execute` call, language `python`: the generator verbatim, then the
spec, then `build(spec, spec["path"])`. Keep the generator unchanged; change
only the spec. It prints `wrote <path>` on success.

```python
import json, re, sys, zipfile
from xml.sax.saxutils import escape

def runs(text):
    """**bold**, *italic*, `code` -> w:r elements"""
    out = []
    for tok in re.split(r'(\*\*.+?\*\*|\*.+?\*|`.+?`)', text):
        if not tok: continue
        props = ""
        if tok.startswith("**"): tok, props = tok[2:-2], "<w:b/>"
        elif tok.startswith("*"): tok, props = tok[1:-1], "<w:i/>"
        elif tok.startswith("`"): tok, props = tok[1:-1], '<w:rFonts w:ascii="Consolas" w:hAnsi="Consolas"/>'
        out.append(f'<w:r><w:rPr>{props}</w:rPr><w:t xml:space="preserve">{escape(tok)}</w:t></w:r>')
    return "".join(out)

def para(text, style=None, numid=None):
    ppr = ""
    if style: ppr += f'<w:pStyle w:val="{style}"/>'
    if numid: ppr += f'<w:numPr><w:ilvl w:val="0"/><w:numId w:val="{numid}"/></w:numPr>'
    return f"<w:p><w:pPr>{ppr}</w:pPr>{runs(text)}</w:p>"

def table(rows, header=True):
    trs = []
    for i, row in enumerate(rows):
        tcs = []
        for cell in row:
            shade = '<w:shd w:val="clear" w:fill="D9E2F3"/>' if header and i == 0 else ""
            txt = f"**{cell}**" if header and i == 0 else str(cell)
            tcs.append(f'<w:tc><w:tcPr>{shade}</w:tcPr>{para(txt)}</w:tc>')
        trs.append("<w:tr>" + "".join(tcs) + "</w:tr>")
    b = 'w:val="single" w:sz="4" w:space="0" w:color="999999"'
    borders = "".join(f"<w:{e} {b}/>" for e in ("top","left","bottom","right","insideH","insideV"))
    return (f'<w:tbl><w:tblPr><w:tblW w:w="0" w:type="auto"/><w:tblBorders>{borders}</w:tblBorders>'
            f'<w:tblCellMar><w:left w:w="80" w:type="dxa"/><w:right w:w="80" w:type="dxa"/></w:tblCellMar>'
            f'</w:tblPr>{"".join(trs)}</w:tbl><w:p/>')

def body(blocks):
    parts = []
    for b in blocks:
        t = b.get("type")
        if t == "heading": parts.append(para(b["text"], f'Heading{b.get("level", 1)}'))
        elif t == "paragraph": parts.append(para(b["text"]))
        elif t == "bullets": parts += [para(x, "ListParagraph", 1) for x in b["items"]]
        elif t == "numbered": parts += [para(x, "ListParagraph", 2) for x in b["items"]]
        elif t == "table": parts.append(table(b["rows"], b.get("header", True)))
        elif t == "page_break": parts.append('<w:p><w:r><w:br w:type="page"/></w:r></w:p>')
        else: raise ValueError(f"unknown block type {t!r}")
    return "".join(parts)

W = 'xmlns:w="http://schemas.openxmlformats.org/wordprocessingml/2006/main"'
def heading_style(n, size, color):
    return (f'<w:style w:type="paragraph" w:styleId="Heading{n}"><w:name w:val="heading {n}"/>'
            f'<w:basedOn w:val="Normal"/><w:next w:val="Normal"/><w:qFormat/>'
            f'<w:pPr><w:keepNext/><w:spacing w:before="{240 if n==1 else 200}" w:after="80"/><w:outlineLvl w:val="{n-1}"/></w:pPr>'
            f'<w:rPr><w:b/><w:color w:val="{color}"/><w:sz w:val="{size}"/></w:rPr></w:style>')
STYLES = (f'<?xml version="1.0" encoding="UTF-8" standalone="yes"?><w:styles {W}>'
          '<w:docDefaults><w:rPrDefault><w:rPr><w:rFonts w:ascii="Calibri" w:hAnsi="Calibri" w:cs="Calibri"/>'
          '<w:sz w:val="22"/><w:lang w:val="en-US"/></w:rPr></w:rPrDefault>'
          '<w:pPrDefault><w:pPr><w:spacing w:after="120" w:line="276" w:lineRule="auto"/></w:pPr></w:pPrDefault></w:docDefaults>'
          '<w:style w:type="paragraph" w:default="1" w:styleId="Normal"><w:name w:val="Normal"/><w:qFormat/></w:style>'
          '<w:style w:type="paragraph" w:styleId="Title"><w:name w:val="Title"/><w:basedOn w:val="Normal"/><w:qFormat/>'
          '<w:pPr><w:spacing w:after="240"/></w:pPr><w:rPr><w:b/><w:sz w:val="52"/><w:color w:val="1F3864"/></w:rPr></w:style>'
          + heading_style(1, 32, "1F3864") + heading_style(2, 26, "2E74B5") + heading_style(3, 24, "2E74B5") + heading_style(4, 22, "404040") +
          '<w:style w:type="paragraph" w:styleId="ListParagraph"><w:name w:val="List Paragraph"/><w:basedOn w:val="Normal"/>'
          '<w:pPr><w:ind w:left="720"/><w:contextualSpacing/></w:pPr></w:style></w:styles>')
NUMBERING = (f'<?xml version="1.0" encoding="UTF-8" standalone="yes"?><w:numbering {W}>'
             '<w:abstractNum w:abstractNumId="0"><w:multiLevelType w:val="hybridMultilevel"/>'
             '<w:lvl w:ilvl="0"><w:start w:val="1"/><w:numFmt w:val="bullet"/><w:lvlText w:val="•"/><w:lvlJc w:val="left"/>'
             '<w:pPr><w:ind w:left="720" w:hanging="360"/></w:pPr><w:rPr><w:rFonts w:ascii="Symbol" w:hAnsi="Symbol" w:hint="default"/></w:rPr></w:lvl></w:abstractNum>'
             '<w:abstractNum w:abstractNumId="1"><w:multiLevelType w:val="hybridMultilevel"/>'
             '<w:lvl w:ilvl="0"><w:start w:val="1"/><w:numFmt w:val="decimal"/><w:lvlText w:val="%1."/><w:lvlJc w:val="left"/>'
             '<w:pPr><w:ind w:left="720" w:hanging="360"/></w:pPr></w:lvl></w:abstractNum>'
             '<w:num w:numId="1"><w:abstractNumId w:val="0"/></w:num><w:num w:numId="2"><w:abstractNumId w:val="1"/></w:num></w:numbering>')
CONTENT_TYPES = ('<?xml version="1.0" encoding="UTF-8" standalone="yes"?>'
    '<Types xmlns="http://schemas.openxmlformats.org/package/2006/content-types">'
    '<Default Extension="rels" ContentType="application/vnd.openxmlformats-package.relationships+xml"/>'
    '<Default Extension="xml" ContentType="application/xml"/>'
    '<Override PartName="/word/document.xml" ContentType="application/vnd.openxmlformats-officedocument.wordprocessingml.document.main+xml"/>'
    '<Override PartName="/word/styles.xml" ContentType="application/vnd.openxmlformats-officedocument.wordprocessingml.styles+xml"/>'
    '<Override PartName="/word/numbering.xml" ContentType="application/vnd.openxmlformats-officedocument.wordprocessingml.numbering+xml"/>'
    '<Override PartName="/docProps/core.xml" ContentType="application/vnd.openxmlformats-package.core-properties+xml"/>'
    '</Types>')
ROOT_RELS = ('<?xml version="1.0" encoding="UTF-8" standalone="yes"?>'
    '<Relationships xmlns="http://schemas.openxmlformats.org/package/2006/relationships">'
    '<Relationship Id="rId1" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/officeDocument" Target="word/document.xml"/>'
    '<Relationship Id="rId2" Type="http://schemas.openxmlformats.org/package/2006/relationships/metadata/core-properties" Target="docProps/core.xml"/>'
    '</Relationships>')
DOC_RELS = ('<?xml version="1.0" encoding="UTF-8" standalone="yes"?>'
    '<Relationships xmlns="http://schemas.openxmlformats.org/package/2006/relationships">'
    '<Relationship Id="rId1" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/styles" Target="styles.xml"/>'
    '<Relationship Id="rId2" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/numbering" Target="numbering.xml"/>'
    '</Relationships>')

def build(spec, path):
    blocks = list(spec.get("blocks", []))
    title = spec.get("title")
    doc = (f'<?xml version="1.0" encoding="UTF-8" standalone="yes"?><w:document {W}><w:body>'
           + (para(title, "Title") if title else "") + body(blocks)
           + '<w:sectPr><w:pgSz w:w="11906" w:h="16838"/><w:pgMar w:top="1417" w:right="1417" w:bottom="1134" w:left="1417" w:header="708" w:footer="708" w:gutter="0"/></w:sectPr>'
           '</w:body></w:document>')
    core = ('<?xml version="1.0" encoding="UTF-8" standalone="yes"?>'
            '<cp:coreProperties xmlns:cp="http://schemas.openxmlformats.org/package/2006/metadata/core-properties" '
            'xmlns:dc="http://purl.org/dc/elements/1.1/" xmlns:dcterms="http://purl.org/dc/terms/" '
            'xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">'
            f'<dc:title>{escape(title or "")}</dc:title><dc:creator>{escape(spec.get("author", ""))}</dc:creator></cp:coreProperties>')
    with zipfile.ZipFile(path, "w", zipfile.ZIP_DEFLATED) as z:
        z.writestr("[Content_Types].xml", CONTENT_TYPES)
        z.writestr("_rels/.rels", ROOT_RELS)
        z.writestr("word/_rels/document.xml.rels", DOC_RELS)
        z.writestr("word/document.xml", doc)
        z.writestr("word/styles.xml", STYLES)
        z.writestr("word/numbering.xml", NUMBERING)
        z.writestr("docProps/core.xml", core)
    print(f"wrote {path}")

spec = { ... }          # your spec here
build(spec, spec["path"])
```

## 4. Check and report

After the call, confirm the file exists and is not empty
(`import os; print(os.path.getsize(path))` in a second call, or `ls -l` via
`bash` on the host path) and tell the user the path and what the document
contains — sections and tables, in one or two sentences. If the run fails,
read the traceback: a `KeyError` or `ValueError` names the block that is
wrong in the spec.
