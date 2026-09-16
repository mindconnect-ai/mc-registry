---
name: docx-builder
description: Use when the user wants a Word document (.docx) with a title, headings, bullet or numbered lists, tables, page breaks or bold/italic text — richer than the plain sections document_write produces
tools: code_execute, bash
---
# Building a Word document

A document is written by `mc_office.docx_builder`, which the `code_execute`
python image carries: you describe the content as a small spec, it writes the
`.docx`. Use it when the user wants headings, lists, tables or inline
emphasis; for a few plain sections the `document_write` tool is enough and
needs no code.

## 1. Draft the content first

Agree the outline with the user's request before generating: title, the
sections and what goes in each. Write real prose — a document of bullet
fragments reads like notes. Keep tables to what fits a page width (up to
about five columns).

## Where the file lands

`code_execute` works in the chat's working directory, mounted under its real
path; its description names it. Give `path` relative to it (or as that full
path) and the file is the user's the moment the call returns — they find it
in the chat's Files dialog. Tell them the full host path. Do not write to
`/mnt/host`; that mount, where there is one, is read-only.

## 2. The spec

```python
spec = {
  "path": "quarterly-report.docx",  # relative: lands in the chat's working directory
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

One `code_execute` call, language `python`, with the spec and two lines — the
generator is part of the image as `mc_office.docx_builder`, so never paste or
rewrite it:

```python
from mc_office import docx_builder
spec = { ... }          # your spec here
docx_builder.build(spec, spec["path"])
```

It prints `wrote <path>` on success.

If the call fails with `ModuleNotFoundError: No module named 'mc_office'`, the
installation runs `code_execute` on a plain python image. Say so: the operator
sets `mindconnect.code-exec.languages` back to the default image
(`ghcr.io/mindconnect-ai/code-exec-python`). Offer the content as Markdown
meanwhile; do not pretend a file was delivered, and do not hand-write the
.docx format.

## 4. Check and report

After the call, confirm the file exists and is not empty
(`import os; print(os.path.getsize(path))` in a second call, or `ls -l` via
`bash` on the host path) and tell the user the path and what the document
contains — sections and tables, in one or two sentences. If the run fails,
read the traceback: a `KeyError` or `ValueError` names the block that is
wrong in the spec.
