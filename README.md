# sheets-viewer

A single-file, read-only spreadsheet viewer built to be the target of **citation
links** — e.g. from a RAG pipeline that answers questions from spreadsheets.
A link opens a workbook on the right sheet, scrolls to the cited rows or
columns, highlights them, and explains in a banner what is being cited.

- One HTML file, no build step, no dependencies to install.
- Fully self-contained: a Content-Security-Policy stops the page from loading
  or contacting anything except the linked spreadsheet on its own site.
  No analytics, CDNs or fonts.
- Reads `.xlsx .xlsm .xlsb .xls .ods .csv .tsv .numbers` and more (via SheetJS).
- Handles large sheets (tested at 100,000 rows): rows render in a movable
  5,000-row window, while search and citations cover every row.

## Hosting

Serve `sheets-viewer.html` from the **same site** as the workbooks. It fetches
the file named in the link, so it has to be opened over `http(s)://`, not by
double-clicking (browsers block `file://` pages from fetching files).

Two settings near the top of the script:

| Setting | Default | Meaning |
| --- | --- | --- |
| `FILE_PATH_PREFIX` | `"/"` | `?file=` may only point inside this folder. Set it to where your workbooks are served, e.g. `"/rag-files/"`. |
| `MAX_FILE_BYTES` | 50 MB | Bigger files show "too large to view here" plus a Download button instead of freezing the tab. |

Recommended server headers (they can't be set from inside the page):
`Content-Security-Policy: frame-ancestors 'none'` (stops other sites framing
the viewer) and your normal authentication on the workbook folder — the viewer
fetches files with the visitor's own credentials.

## Link format

```
sheets-viewer.html?file=<path>&sheet=<sheet name>[&highlight=v1|v2][&columns=Col A|Col B][&note=<claim>]
```

| Parameter | Use it when | What the viewer does |
| --- | --- | --- |
| `file` (required) | always | Loads the workbook. Must be a spreadsheet under `FILE_PATH_PREFIX` on this site. |
| `sheet` | always, if known | Opens that sheet. Alone, it means "the answer is about this sheet". |
| `highlight` | the answer points at specific rows | Highlights every row containing **all** the values (separated by `\|`), opens the sheet of the first match and scrolls to it. |
| `columns` | the answer is computed over columns (count, total, average, filter) | Highlights those columns. The header can be in any of the first 20 rows. |
| `note` | optional | Shows the cited claim in the banner ("Cited claim: …", max 300 chars). |

Matching ignores case. Numbers match raw or formatted (`1234.5` = `1,234.50`),
dates match ISO or as displayed (`2026-02-04` = `2/4/26`), and a numeric term
must equal the whole cell (`3` does not match `1,234.50`). If something isn't
found, the banner says so instead of failing silently.

### Having an AI produce citations

Have the model output a citation object and let your code build the URL
(models get URL-encoding wrong):

```json
{
  "file": "exports/folder_index_part002.xlsx",
  "sheet": "Index",
  "highlight": ["F-012345", "2025-03-14"],
  "columns": ["ProgramGroup", "Date Modified"],
  "note": "214 Program A files were modified in 2025."
}
```

```python
from urllib.parse import urlencode

def citation_url(c, viewer="/sheets-viewer.html"):
    q = {"file": c["file"], "sheet": c["sheet"]}
    if c.get("highlight"): q["highlight"] = "|".join(map(str, c["highlight"]))
    if c.get("columns"):   q["columns"]   = "|".join(c["columns"])
    if c.get("note"):      q["note"]      = c["note"]
    return viewer + "?" + urlencode(q)
```

Prompt text:

> When your answer uses a spreadsheet, add a citation object with `file` and
> `sheet`, plus one of: `highlight` — 2–3 values copied exactly from one cited
> row (an ID or name plus a date or amount) when the answer points at specific
> rows; `columns` — the column names your code filtered or aggregated on, when
> the answer is computed (count, total, filter, list all); or nothing extra
> when the answer describes the sheet. Always set `note` to the one-sentence
> claim. Never invent values or column names.

## Viewer features

Sheet tabs with per-sheet search-match counts · search across all sheets
(Enter / Shift+Enter, Cmd/Ctrl+F) · zoom slider · wrap long text ·
drag-to-resize columns (double-click an edge to fit) · sticky header row and
row numbers · view / copy the current sheet as CSV · Download the original ·
Top button · light / dark theme · one-axis scrolling (no diagonal drift).

## Limits and known issues

- Rows are numbered by position after skipping blank rows, so they can differ
  from Excel's row numbers.
- Search stops counting at 50,000 matches; highlight takes up to 10 values,
  columns up to 20.
- Memory is roughly 30× the file size once parsed (a 4.9 MB, 100k-row workbook
  uses ~130 MB).
- The bundled SheetJS is 0.18.5, which has two published advisories that need
  a crafted file: CVE-2023-30533 (prototype pollution, fixed in 0.19.3) and
  CVE-2024-22363 (ReDoS, fixed in 0.20.2). The page's CSP prevents a crafted
  file from sending data anywhere; worst case is a misbehaving or frozen tab.

## Credits

Based on [Sheets Viewer](https://github.com/MichalAFerber/sheets-viewer.us) by
Michal Ferber (MIT). See `LICENSE` and `NOTICE.md`.
