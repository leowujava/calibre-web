# Reading Progress Saving Design

**Date:** 2026-08-10
**Status:** Approved

## Overview

Add reading progress saving for browser-based readers (EPUB, comic, PDF) that integrates with the existing Kobo device sync infrastructure. Reading progress is saved to the existing `KoboReadingState`/`KoboBookmark` models, shared with Kobo device data. On reader load, the most recent position (from any source) is restored.

## Architecture

All reading position data flows through the existing `KoboReadingState` entity in `ub.py`. Browser readers save via a new REST API endpoint; Kobo devices save via the existing sync endpoint. Both write to the same `KoboBookmark` table with a `location_source` field distinguishing `"browser"` from `"kobo"`. The most recently modified record wins (timestamp-based conflict resolution).

## Components

### 1. Backend API (`cps/web.py`)

**GET /ajax/readingprogress/<book_id>**
- Returns current reading state as JSON: `{ progress_percent, location_value, location_type, location_source, cfi, current_page, total_pages }`
- If no state exists, returns `null`
- If book not found, returns 404

**POST /ajax/savereadingprogress/<book_id>**
- Accepts: `{ format, progress_percent, location_value, location_type, cfi, current_page, total_pages }`
- Creates or updates `KoboReadingState` + `KoboBookmark` with `location_source="browser"`
- Sets `ReadBook.read_status = STATUS_IN_PROGRESS`
- Returns 200 with current state

### 2. Kobo Sync Merge (`cps/kobo.py`)

Modify `HandleStateRequest` to compare `KoboBookmark.last_modified` timestamps between browser and Kobo device writes. Only overwrite if the incoming update is newer.

### 3. Reader Templates

**EPUB (`cps/templates/read.html`)**: Pass `reading_state` via `window.calibre`
**Comic (`cps/templates/readcbr.html`)**: Pass `reading_state` via `window.calibre`
**PDF (`cps/templates/readpdf.html`)**: Pass `reading_state` via `window.calibre`

### 4. Browser Reader JavaScript

**EPUB (`eps/static/js/reading/epub.js`)**
- On init: restore `window.calibre.readingState` position (CFI location)
- On `rendition.on("relocated")`: update progress display, schedule debounced save (3s)
- On `beforeunload`: force-save current position
- Auto-load locations if not yet generated

**Comic (`cps/static/js/kthoom.js`)**
- On init: restore `window.calibre.readingState.currentPage`
- On page change: schedule debounced save (3s)
- On `beforeunload`: force-save

**PDF (`cps/templates/readpdf.html`)**
- Add progress tracking via pdf.js `pagechange` event
- Debounced save (3s) on page navigation

## Data Flow

```
Reader opens → GET /ajax/readingprogress/<book_id>
  → Server returns { progress_percent, location_value, cfi }
  → Reader positions to saved location

User reads → (every 3s debounce) POST /ajax/savereadingprogress/<book_id>
  → Server updates KoboBookmark with source="browser"

Reader closes → force-save on beforeunload

Kobo sync → existing /kobo/v1/library/<uuid>/state
  → Server compares timestamps, most recent wins
```

## Files Modified

| File | Change |
|---|---|
| `cps/web.py` | Add GET/POST `/ajax/readingprogress/<book_id>` and GET/POST `/ajax/savereadingprogress/<book_id>` |
| `cps/kobo.py` | Modify `HandleStateRequest` timestamp-based conflict resolution |
| `cps/templates/read.html` | Pass `reading_state` to `window.calibre` |
| `cps/static/js/reading/epub.js` | Load saved position, debounced auto-save, save on unload |
| `cps/templates/readcbr.html` | Pass `reading_state` to `window.calibre` |
| `cps/static/js/kthoom.js` | Load saved position, debounced auto-save, save on unload |
| `cps/templates/readpdf.html` | Add progress tracking and auto-save |

## Error Handling

- If `KoboReadingState` doesn't exist: create it via `get_or_create_reading_state()`
- If book not found: return 404
- If user not authenticated: return 401 (anonymous keeps localStorage only)
- DB errors: log and return 500, reader continues with localStorage/local fallback
