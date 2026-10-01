# AGENTS.md — sheets-viewer

## Purpose
Single-file, read-only spreadsheet viewer used as the target of RAG citation links:
`sheets-viewer.html?file=<path>&sheet=<name>[&highlight=a|b][&columns=A|B][&note=...]`
opens a workbook on the cited sheet, scrolls to and highlights the cited rows/columns.

## Stack
One self-contained HTML file (`sheets-viewer.html`, ~940 KB) with inline JS/CSS and
inlined SheetJS Community Edition 0.18.5. No build step, no package manager, no deps.
Strict Content-Security-Policy: no CDNs, fonts or analytics.

## Install / run / test
- No install. Serve the file over http(s) from the same site as the workbooks
  (it fetches `?file=`; `file://` will not work), e.g. from a folder containing it
  and some workbooks: `python3 -m http.server 8000` then open
  `http://localhost:8000/sheets-viewer.html?file=/book.xlsx&sheet=Sheet1`.
- Settings near the top of the script: `FILE_PATH_PREFIX` (default `"/"`),
  `MAX_FILE_BYTES` (50 MB).
- No automated tests.

## Where outputs / data go
Nothing is written; the viewer is read-only and keeps no state on disk or server.

## Do not move
- `sheets-viewer.html` — the deliverable; links from RAG answers point at this filename.
- `LICENSE`, `NOTICE.md`, `licenses/Apache-2.0.txt` — required attribution for the
  upstream Sheets Viewer (MIT) and the embedded SheetJS (Apache-2.0); NOTICE.md links them.

## Rules
- Docs go in `docs/` (e.g. `docs/rich-ui-embeds.md`), never the repo root.
- Keep it one file with zero external network requests; do not loosen the CSP.
- Keep the SheetJS license banner where the library is embedded.
