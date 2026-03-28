# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page static website for looking up Mario Kart World intermission route videos. All data comes from `mkw_routes.xlsx`; the site is entirely self-contained in `index.html` with route data embedded as a JSON array.

## Updating route data

Whenever `mkw_routes.xlsx` changes, regenerate the embedded JSON using this pattern:

```python
import openpyxl, json

wb = openpyxl.load_workbook('mkw_routes.xlsx')
ws = wb.active

routes = []
for r in range(2, ws.max_row + 1):
    row = [ws.cell(r, c).value for c in range(1, ws.max_column + 1)]
    if not row[0]:  # skip trailing empty rows
        continue
    def clean(v):
        if isinstance(v, str): return v.replace('\xa0', ' ')
        return v
    routes.append({
        'source':               clean(row[0]),
        'destination':          clean(row[1]),
        'label':                clean(row[2]),
        'usedInGP':             bool(row[3]),
        'grandPrix':            clean(row[4]),
        'usedInKT':             bool(row[5]),
        'knockoutTour':         clean(row[6]),
        'extraSections':        bool(row[7]),
        'differentFromReverse': bool(row[8]),
        'trackVariant':         clean(row[9]),
        'trackVariantAlt':      clean(row[10]),
        'trackEntrance':        clean(row[11]),
        'trackEntranceAlt':     clean(row[12]),
        'notes':                clean(row[13]),
    })

routes_json = json.dumps(routes, ensure_ascii=True)
```

Then splice it into `index.html` using a safe string replacement (no `re.sub` — it mishandles backslashes in the replacement string):

```python
with open('index.html') as f:
    old = f.read()

MARKER_START = '\nconst ROUTES = '
MARKER_END   = ';\n'
idx_start = old.index(MARKER_START)
idx_end   = old.index(MARKER_END, idx_start + len(MARKER_START))
new_html  = old[:idx_start] + '\nconst ROUTES = ' + routes_json + old[idx_end:]

with open('index.html', 'w') as f:
    f.write(new_html)
```

## Spreadsheet structure (`mkw_routes.xlsx`)

- **Routes** sheet: one route per row, header in row 1, data from row 2
  - Columns (0-indexed): source, destination, label, usedInGP, grandPrix, usedInKT, knockoutTour, extraSections, differentFromReverse, trackVariant, trackVariantAlt, trackEntrance, trackEntranceAlt, notes
  - The spreadsheet has trailing empty rows — always skip rows where `row[0]` is None
- **Track Name Mapping** sheet: track name → abbreviation (used in route labels); not currently embedded in the site

## Videos

`videos/<RouteLabel>.mp4` — filenames match route labels exactly (e.g. `videos/rWS-BC.mp4`). The site attempts to load the video for the selected route and falls back to a placeholder if the file doesn't exist (detected via the `<source>` element's `error` event).

## Site architecture

Everything is in `index.html`:

- **Data**: `const ROUTES = [...]` — the full 202-route array, embedded on a single line
- **UI**: Two ways to select a route:
  1. Source dropdown → filters destination dropdown → auto-displays route
  2. Quick-find input with custom autocomplete (matches on route label substring)
  - Selecting via either method syncs the other
- **Route display**: video player (or placeholder) + facts table + optional notes card
- **Facts display logic**: `trackVariantAlt` and `trackEntranceAlt` take precedence over their primary fields when present; GP/KT names are shown when the boolean is true

## Serving locally

```bash
brew install caddy
caddy file-server --root . --listen :8080
```

Python's `http.server` does **not** support HTTP range requests and will break video seeking — use Caddy or nginx instead.
