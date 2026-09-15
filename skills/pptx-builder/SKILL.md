---
name: pptx-builder
description: Use when the user wants a PowerPoint presentation (.pptx) — a slide deck with a title slide, bullet slides, two-column, table or text slides and speaker notes
tools: code_execute, bash
---
# Building a PowerPoint deck

A `.pptx` is a zip of XML parts. The generator below writes one with nothing
but Python's standard library, from a list of slides — so it runs in the
`code_execute` container as it is, no packages, no network. It produces a
clean 16:9 deck: dark-blue titles, Calibri, one blank layout, real tables,
speaker notes where you give them.

## 1. Plan the deck first

Decide the slides before generating. One message per slide; at most six
bullets of one line each; a table no wider than five columns and no longer
than eight rows. A title slide first, and a closing slide that says what
happens next. Speaker notes are where the sentences go — the slide carries
the headline, the notes carry the argument.

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
  "path": "/mnt/host/roadmap.pptx",    # see above
  "title": "Roadmap 2027",             # document property
  "author": "Mindconnect",
  "slides": [
    {"type": "title", "title": "Roadmap 2027", "subtitle": "Where we go next", "notes": "Open with the numbers."},
    {"type": "bullets", "title": "Where we stand",
     "bullets": ["**Revenue** up 12 %", "Costs flat", ["Cloud spend down", "Hiring paused"], "Churn below 2 %"]},
    {"type": "two-column", "title": "Options",
     "left": {"heading": "Build", "bullets": ["Own stack", "Slow"]},
     "right": {"heading": "Buy", "bullets": ["Faster", "Lock-in"]}},
    {"type": "table", "title": "Figures", "rows": [["Quarter", "Revenue", "Costs"], ["Q1", "1.2 M", "0.9 M"]]},
    {"type": "text", "title": "Closing", "paragraphs": ["Decide by March.", "Then we build."]},
  ]
}
```

Slide types: `title`, `bullets` (a nested list is one level of sub-bullets),
`two-column`, `table` (first row is the header), `text`. Any slide may carry
`"notes"`. `**bold**` works inside any text; nothing else is interpreted.

## 3. Run it

One `code_execute` call, language `python`: the generator verbatim, then the
spec, then `build(spec, spec["path"])`. Keep the generator unchanged; change
only the spec. It prints `wrote <path>: N slides` on success.

```python
import json, re, sys, zipfile
from xml.sax.saxutils import escape

EMU = 12700  # per point; slide is 16:9, 13.333 x 7.5 in
SLIDE_W, SLIDE_H = 12192000, 6858000
NS = ('xmlns:a="http://schemas.openxmlformats.org/drawingml/2006/main" '
      'xmlns:r="http://schemas.openxmlformats.org/officeDocument/2006/relationships" '
      'xmlns:p="http://schemas.openxmlformats.org/presentationml/2006/main"')
XML = '<?xml version="1.0" encoding="UTF-8" standalone="yes"?>'

def runs(text, size, bold=False, color=None):
    out = []
    for tok in re.split(r'(\*\*.+?\*\*)', text):
        if not tok: continue
        b = bold or tok.startswith("**")
        if tok.startswith("**"): tok = tok[2:-2]
        fill = f'<a:solidFill><a:srgbClr val="{color}"/></a:solidFill>' if color else ""
        out.append(f'<a:r><a:rPr lang="en-US" sz="{size*100}"{" b=\"1\"" if b else ""} dirty="0">{fill}</a:rPr>'
                   f'<a:t>{escape(tok)}</a:t></a:r>')
    return "".join(out)

def textbox(sid, name, x, y, w, h, paras, anchor="t"):
    return (f'<p:sp><p:nvSpPr><p:cNvPr id="{sid}" name="{name}"/><p:cNvSpPr txBox="1"/><p:nvPr/></p:nvSpPr>'
            f'<p:spPr><a:xfrm><a:off x="{x}" y="{y}"/><a:ext cx="{w}" cy="{h}"/></a:xfrm>'
            f'<a:prstGeom prst="rect"><a:avLst/></a:prstGeom></p:spPr>'
            f'<p:txBody><a:bodyPr wrap="square" anchor="{anchor}"><a:normAutofit/></a:bodyPr><a:lstStyle/>{paras}</p:txBody></p:sp>')

def para(text, size, bold=False, color=None, bullet=False, level=0, align=None):
    ppr = f'<a:pPr lvl="{level}"' + (f' algn="{align}"' if align else "") + ' marL="{0}" indent="{1}">'.format(342900 * (level + 1) if bullet else 0, -342900 if bullet else 0)
    ppr += ('<a:buChar char="&#8226;"/>' if bullet else '<a:buNone/>') + '</a:pPr>'
    return f'<a:p>{ppr}{runs(text, size, bold, color)}</a:p>'

def table_shape(sid, x, y, w, rows, header=True):
    cols = len(rows[0]); cw = w // cols; rh = 370840
    grid = "".join(f'<a:gridCol w="{cw}"/>' for _ in range(cols))
    trs = []
    for i, row in enumerate(rows):
        tcs = []
        for cell in row:
            hdr = header and i == 0
            fill = '<a:solidFill><a:srgbClr val="1F3864"/></a:solidFill>' if hdr else ('<a:solidFill><a:srgbClr val="F2F2F2"/></a:solidFill>' if i % 2 == 0 else "")
            tcs.append(f'<a:tc><a:txBody><a:bodyPr/><a:lstStyle/>{para(str(cell), 14, hdr, "FFFFFF" if hdr else None)}</a:txBody>'
                       f'<a:tcPr marL="68580" marR="68580" marT="34290" marB="34290">{fill}</a:tcPr></a:tc>')
        trs.append(f'<a:tr h="{rh}">' + "".join(tcs) + '</a:tr>')
    return (f'<p:graphicFrame><p:nvGraphicFramePr><p:cNvPr id="{sid}" name="Table {sid}"/><p:cNvGraphicFramePr><a:graphicFrameLocks noGrp="1"/></p:cNvGraphicFramePr><p:nvPr/></p:nvGraphicFramePr>'
            f'<p:xfrm><a:off x="{x}" y="{y}"/><a:ext cx="{w}" cy="{rh*len(rows)}"/></p:xfrm>'
            f'<a:graphic><a:graphicData uri="http://schemas.openxmlformats.org/drawingml/2006/table">'
            f'<a:tbl><a:tblPr firstRow="1" bandRow="1"/><a:tblGrid>{grid}</a:tblGrid>{"".join(trs)}</a:tbl></a:graphicData></a:graphic></p:graphicFrame>')

def slide_xml(slide):
    kind = slide.get("type", "bullets")
    shapes = []
    M = 609600  # 0.67in margin
    if kind == "title":
        shapes.append(textbox(2, "Title", M, 2200000, SLIDE_W - 2*M, 1400000,
                              para(slide["title"], 44, True, "1F3864", align="ctr"), "b"))
        if slide.get("subtitle"):
            shapes.append(textbox(3, "Subtitle", M, 3700000, SLIDE_W - 2*M, 1000000,
                                  para(slide["subtitle"], 24, False, "595959", align="ctr")))
    else:
        shapes.append(textbox(2, "Title", M, 400000, SLIDE_W - 2*M, 900000,
                              para(slide["title"], 32, True, "1F3864"), "b"))
        top = 1500000; bw = SLIDE_W - 2*M
        if kind == "bullets":
            ps = "".join(para(t, 20, bullet=True, level=lvl) for t, lvl in bullets(slide.get("bullets", [])))
            shapes.append(textbox(3, "Body", M, top, bw, SLIDE_H - top - M, ps))
        elif kind == "two-column":
            half = (bw - 300000) // 2
            for i, key in enumerate(("left", "right")):
                col = slide.get(key, {})
                ps = para(col.get("heading", ""), 22, True, "2E74B5") if col.get("heading") else ""
                ps += "".join(para(t, 18, bullet=True, level=lvl) for t, lvl in bullets(col.get("bullets", [])))
                shapes.append(textbox(3 + i, key.capitalize(), M + i * (half + 300000), top, half, SLIDE_H - top - M, ps))
        elif kind == "table":
            shapes.append(table_shape(3, M, top, bw, slide["rows"], slide.get("header", True)))
        elif kind == "text":
            ps = "".join(para(t, 18) for t in slide.get("paragraphs", []))
            shapes.append(textbox(3, "Body", M, top, bw, SLIDE_H - top - M, ps))
        else:
            raise ValueError(f"unknown slide type {kind!r}")
    return (f'{XML}<p:sld {NS}><p:cSld><p:spTree><p:nvGrpSpPr><p:cNvPr id="1" name=""/><p:cNvGrpSpPr/><p:nvPr/></p:nvGrpSpPr>'
            f'<p:grpSpPr><a:xfrm><a:off x="0" y="0"/><a:ext cx="0" cy="0"/><a:chOff x="0" y="0"/><a:chExt cx="0" cy="0"/></a:xfrm></p:grpSpPr>'
            f'{"".join(shapes)}</p:spTree></p:cSld><p:clrMapOvr><a:masterClrMapping/></p:clrMapOvr></p:sld>')

def bullets(items):
    """strings, or {"text":..,"level":1} dicts, or nested lists for sub-bullets"""
    out = []
    def walk(xs, lvl):
        for x in xs:
            if isinstance(x, list): walk(x, lvl + 1)
            elif isinstance(x, dict): out.append((x["text"], x.get("level", lvl)))
            else: out.append((x, lvl))
    walk(items, 0)
    return out

def notes_xml(text):
    return (f'{XML}<p:notes {NS}><p:cSld><p:spTree><p:nvGrpSpPr><p:cNvPr id="1" name=""/><p:cNvGrpSpPr/><p:nvPr/></p:nvGrpSpPr>'
            '<p:grpSpPr><a:xfrm><a:off x="0" y="0"/><a:ext cx="0" cy="0"/><a:chOff x="0" y="0"/><a:chExt cx="0" cy="0"/></a:xfrm></p:grpSpPr>'
            '<p:sp><p:nvSpPr><p:cNvPr id="2" name="Notes"/><p:cNvSpPr><a:spLocks noGrp="1"/></p:cNvSpPr><p:nvPr><p:ph type="body" idx="1"/></p:nvPr></p:nvSpPr>'
            f'<p:spPr/><p:txBody><a:bodyPr/><a:lstStyle/>{para(text, 12)}</p:txBody></p:sp>'
            '</p:spTree></p:cSld><p:clrMapOvr><a:masterClrMapping/></p:clrMapOvr></p:notes>')

THEME = (f'{XML}<a:theme xmlns:a="http://schemas.openxmlformats.org/drawingml/2006/main" name="Plain"><a:themeElements>'
    '<a:clrScheme name="Plain"><a:dk1><a:srgbClr val="000000"/></a:dk1><a:lt1><a:srgbClr val="FFFFFF"/></a:lt1><a:dk2><a:srgbClr val="1F3864"/></a:dk2><a:lt2><a:srgbClr val="EEEEEE"/></a:lt2>'
    '<a:accent1><a:srgbClr val="2E74B5"/></a:accent1><a:accent2><a:srgbClr val="ED7D31"/></a:accent2><a:accent3><a:srgbClr val="A5A5A5"/></a:accent3><a:accent4><a:srgbClr val="FFC000"/></a:accent4>'
    '<a:accent5><a:srgbClr val="5B9BD5"/></a:accent5><a:accent6><a:srgbClr val="70AD47"/></a:accent6><a:hlink><a:srgbClr val="0563C1"/></a:hlink><a:folHlink><a:srgbClr val="954F72"/></a:folHlink></a:clrScheme>'
    '<a:fontScheme name="Plain"><a:majorFont><a:latin typeface="Calibri Light"/><a:ea typeface=""/><a:cs typeface=""/></a:majorFont><a:minorFont><a:latin typeface="Calibri"/><a:ea typeface=""/><a:cs typeface=""/></a:minorFont></a:fontScheme>'
    '<a:fmtScheme name="Plain"><a:fillStyleLst><a:solidFill><a:schemeClr val="phClr"/></a:solidFill><a:solidFill><a:schemeClr val="phClr"/></a:solidFill><a:solidFill><a:schemeClr val="phClr"/></a:solidFill></a:fillStyleLst>'
    '<a:lnStyleLst><a:ln w="6350"><a:solidFill><a:schemeClr val="phClr"/></a:solidFill></a:ln><a:ln w="12700"><a:solidFill><a:schemeClr val="phClr"/></a:solidFill></a:ln><a:ln w="19050"><a:solidFill><a:schemeClr val="phClr"/></a:solidFill></a:ln></a:lnStyleLst>'
    '<a:effectStyleLst><a:effectStyle><a:effectLst/></a:effectStyle><a:effectStyle><a:effectLst/></a:effectStyle><a:effectStyle><a:effectLst/></a:effectStyle></a:effectStyleLst>'
    '<a:bgFillStyleLst><a:solidFill><a:schemeClr val="phClr"/></a:solidFill><a:solidFill><a:schemeClr val="phClr"/></a:solidFill><a:solidFill><a:schemeClr val="phClr"/></a:solidFill></a:bgFillStyleLst></a:fmtScheme>'
    '</a:themeElements></a:theme>')
EMPTY_TREE = ('<p:cSld><p:spTree><p:nvGrpSpPr><p:cNvPr id="1" name=""/><p:cNvGrpSpPr/><p:nvPr/></p:nvGrpSpPr>'
              '<p:grpSpPr><a:xfrm><a:off x="0" y="0"/><a:ext cx="0" cy="0"/><a:chOff x="0" y="0"/><a:chExt cx="0" cy="0"/></a:xfrm></p:grpSpPr></p:spTree></p:cSld>')
CLRMAP = '<p:clrMap bg1="lt1" tx1="dk1" bg2="lt2" tx2="dk2" accent1="accent1" accent2="accent2" accent3="accent3" accent4="accent4" accent5="accent5" accent6="accent6" hlink="hlink" folHlink="folHlink"/>'
TXSTYLES = ('<p:txStyles><p:titleStyle><a:lvl1pPr><a:defRPr sz="3200"/></a:lvl1pPr></p:titleStyle>'
            '<p:bodyStyle><a:lvl1pPr><a:defRPr sz="2000"/></a:lvl1pPr></p:bodyStyle><p:otherStyle><a:lvl1pPr><a:defRPr sz="1800"/></a:lvl1pPr></p:otherStyle></p:txStyles>')
MASTER = f'{XML}<p:sldMaster {NS}>{EMPTY_TREE}{CLRMAP}<p:sldLayoutIdLst><p:sldLayoutId id="2147483649" r:id="rId1"/></p:sldLayoutIdLst>{TXSTYLES}</p:sldMaster>'
LAYOUT = f'{XML}<p:sldLayout {NS} type="blank" preserve="1">{EMPTY_TREE}<p:clrMapOvr><a:masterClrMapping/></p:clrMapOvr></p:sldLayout>'
NOTES_MASTER = f'{XML}<p:notesMaster {NS}>{EMPTY_TREE}{CLRMAP}</p:notesMaster>'
REL = 'http://schemas.openxmlformats.org/officeDocument/2006/relationships/'
def rels(pairs):
    return (f'{XML}<Relationships xmlns="http://schemas.openxmlformats.org/package/2006/relationships">'
            + "".join(f'<Relationship Id="rId{i+1}" Type="{REL}{t}" Target="{tg}"/>' for i, (t, tg) in enumerate(pairs)) + '</Relationships>')

def build(spec, path):
    slides = spec["slides"]; n = len(slides)
    has_notes = any(s.get("notes") for s in slides)
    ct = [f'{XML}<Types xmlns="http://schemas.openxmlformats.org/package/2006/content-types">',
          '<Default Extension="rels" ContentType="application/vnd.openxmlformats-package.relationships+xml"/>',
          '<Default Extension="xml" ContentType="application/xml"/>',
          '<Override PartName="/ppt/presentation.xml" ContentType="application/vnd.openxmlformats-officedocument.presentationml.presentation.main+xml"/>',
          '<Override PartName="/ppt/slideMasters/slideMaster1.xml" ContentType="application/vnd.openxmlformats-officedocument.presentationml.slideMaster+xml"/>',
          '<Override PartName="/ppt/slideLayouts/slideLayout1.xml" ContentType="application/vnd.openxmlformats-officedocument.presentationml.slideLayout+xml"/>',
          '<Override PartName="/ppt/theme/theme1.xml" ContentType="application/vnd.openxmlformats-officedocument.theme+xml"/>',
          '<Override PartName="/docProps/core.xml" ContentType="application/vnd.openxmlformats-package.core-properties+xml"/>']
    if has_notes:
        ct.append('<Override PartName="/ppt/notesMasters/notesMaster1.xml" ContentType="application/vnd.openxmlformats-officedocument.presentationml.notesMaster+xml"/>')
    files = {}
    for i, s in enumerate(slides, 1):
        ct.append(f'<Override PartName="/ppt/slides/slide{i}.xml" ContentType="application/vnd.openxmlformats-officedocument.presentationml.slide+xml"/>')
        files[f"ppt/slides/slide{i}.xml"] = slide_xml(s)
        srels = [("slideLayout", "../slideLayouts/slideLayout1.xml")]
        if s.get("notes"):
            ct.append(f'<Override PartName="/ppt/notesSlides/notesSlide{i}.xml" ContentType="application/vnd.openxmlformats-officedocument.presentationml.notesSlide+xml"/>')
            files[f"ppt/notesSlides/notesSlide{i}.xml"] = notes_xml(s["notes"])
            files[f"ppt/notesSlides/_rels/notesSlide{i}.xml.rels"] = rels([("notesMaster", "../notesMasters/notesMaster1.xml"), ("slide", f"../slides/slide{i}.xml")])
            srels.append(("notesSlide", f"../notesSlides/notesSlide{i}.xml"))
        files[f"ppt/slides/_rels/slide{i}.xml.rels"] = rels(srels)
    ct.append('</Types>')
    pres_rels = [("slideMaster", "slideMasters/slideMaster1.xml")] + [("slide", f"slides/slide{i}.xml") for i in range(1, n+1)] + [("theme", "theme/theme1.xml")]
    if has_notes: pres_rels.append(("notesMaster", "notesMasters/notesMaster1.xml"))
    sld_ids = "".join(f'<p:sldId id="{256+i}" r:id="rId{1+i}"/>' for i in range(1, n+1))
    notes_lst = f'<p:notesMasterIdLst><p:notesMasterId r:id="rId{len(pres_rels)}"/></p:notesMasterIdLst>' if has_notes else ""
    files["ppt/presentation.xml"] = (f'{XML}<p:presentation {NS} saveSubsetFonts="1"><p:sldMasterIdLst><p:sldMasterId id="2147483648" r:id="rId1"/></p:sldMasterIdLst>'
        f'{notes_lst}<p:sldIdLst>{sld_ids}</p:sldIdLst><p:sldSz cx="{SLIDE_W}" cy="{SLIDE_H}"/><p:notesSz cx="6858000" cy="9144000"/></p:presentation>')
    files["ppt/_rels/presentation.xml.rels"] = rels(pres_rels)
    files["ppt/slideMasters/slideMaster1.xml"] = MASTER
    files["ppt/slideMasters/_rels/slideMaster1.xml.rels"] = rels([("slideLayout", "../slideLayouts/slideLayout1.xml"), ("theme", "../theme/theme1.xml")])
    files["ppt/slideLayouts/slideLayout1.xml"] = LAYOUT
    files["ppt/slideLayouts/_rels/slideLayout1.xml.rels"] = rels([("slideMaster", "../slideMasters/slideMaster1.xml")])
    files["ppt/theme/theme1.xml"] = THEME
    if has_notes:
        files["ppt/notesMasters/notesMaster1.xml"] = NOTES_MASTER
        files["ppt/notesMasters/_rels/notesMaster1.xml.rels"] = rels([("theme", "../theme/theme1.xml")])
    files["docProps/core.xml"] = (f'{XML}<cp:coreProperties xmlns:cp="http://schemas.openxmlformats.org/package/2006/metadata/core-properties" '
        'xmlns:dc="http://purl.org/dc/elements/1.1/" xmlns:dcterms="http://purl.org/dc/terms/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">'
        f'<dc:title>{escape(spec.get("title", ""))}</dc:title><dc:creator>{escape(spec.get("author", ""))}</dc:creator></cp:coreProperties>')
    files["_rels/.rels"] = rels([("officeDocument", "ppt/presentation.xml")]).replace(
        "</Relationships>", '<Relationship Id="rId2" Type="http://schemas.openxmlformats.org/package/2006/relationships/metadata/core-properties" Target="docProps/core.xml"/></Relationships>')
    with zipfile.ZipFile(path, "w", zipfile.ZIP_DEFLATED) as z:
        z.writestr("[Content_Types].xml", "".join(ct))
        for name, content in files.items(): z.writestr(name, content)
    print(f"wrote {path}: {n} slides")

spec = { ... }          # your spec here
build(spec, spec["path"])
```

## 4. Check and report

After the call, confirm the file exists and is not empty
(`import os; print(os.path.getsize(path))` in a second call, or `ls -l` via
`bash` on the host path) and tell the user the path and the slide titles. If
the run fails, read the traceback: a `KeyError` or `ValueError` names the
slide that is wrong in the spec.
