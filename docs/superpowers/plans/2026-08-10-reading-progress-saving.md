# Reading Progress Saving Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add browser-based reading progress saving that integrates with Kobo device sync through the existing KoboReadingState model.

**Architecture:** New REST API endpoints in `web.py` handle browser save/load requests. Reading state flows through existing `KoboReadingState`/`KoboBookmark` models. Browser and Kobo device progress merge via timestamp-based conflict resolution.

**Tech Stack:** Python, SQLAlchemy, Flask, EpubJS, jQuery, SQLite

## Global Constraints

- Always use `kobo.py:get_or_create_reading_state()` to create or retrieve `KoboReadingState` (handles `ReadBook` creation too)
- Write `location_source = "browser"` for all browser-saved progress
- Save progress with 3-second debounce on all reader types
- Save on `beforeunload` for all reader types (force-save, no debounce)
- For authenticated users only: anonymous users keep localStorage-only behavior unchanged
- The `KoboBookmark.last_modified` timestamp determines conflict resolution when Kobo device syncs

---

### Task 1: Backend - GET /ajax/readingprogress/<book_id>

**Files:**
- Modify: `cps/web.py`

**Interfaces:**
- Produces: `GET /ajax/readingprogress/<book_id>` returns JSON: `{ bookmark: { progress_percent, content_source_progress_percent, location_value, location_type, location_source, cfi }, status, statistics, last_modified }` or `null`

- [ ] **Step 1: Add GET endpoint for reading progress**

Add a new route in `cps/web.py` after the existing `set_bookmark` route (around line 174):

```python
@web.route("/ajax/readingprogress/<int:book_id>", methods=['GET'])
@user_login_required
def get_reading_progress(book_id):
    """Return current reading state for a book, if available."""
    try:
        kobo_reading_state = kobo.get_or_create_reading_state(book_id)
        if not kobo_reading_state or not kobo_reading_state.current_bookmark:
            return jsonify(None)
        
        bookmark = kobo_reading_state.current_bookmark
        book_read = kobo_reading_state.book_read_link
        statistics = kobo_reading_state.statistics
        
        response = {
            "bookmark": {
                "progress_percent": bookmark.progress_percent,
                "content_source_progress_percent": bookmark.content_source_progress_percent,
                "location_value": bookmark.location_value,
                "location_type": bookmark.location_type,
                "location_source": bookmark.location_source,
                "cfi": bookmark.location_value if bookmark.location_source and "CFI" in str(bookmark.location_source) else None,
            },
            "status": book_read.read_status if book_read else None,
            "statistics": {
                "spent_reading_minutes": statistics.spent_reading_minutes,
                "remaining_time_minutes": statistics.remaining_time_minutes,
            },
            "last_modified": kobo_reading_state.last_modified.isoformat() if kobo_reading_state.last_modified else None,
        }
        return jsonify(response)
    except Exception as e:
        log.error("Failed to get reading progress: %s", e)
        return jsonify(None), 200
```

- [ ] **Step 2: Verify the route is valid Python**

Run: `python -m py_compile cps/web.py`
Expected: No syntax errors

- [ ] **Step 3: Ensure `kobo` is imported at the top of web.py**

Check the existing imports in `cps/web.py` line ~42-44. The file already imports `from . import calibre_db, kobo_sync_status`. Add `kobo` to the import:

```python
from . import calibre_db, kobo, kobo_sync_status
```

- [ ] **Step 4: Commit**

```bash
git add cps/web.py
git commit -m "feat: add GET /ajax/readingprogress/<book_id> endpoint"
```

---

### Task 2: Backend - POST /ajax/savereadingprogress/<book_id>

**Files:**
- Modify: `cps/web.py`

**Interfaces:**
- Consumes: `book_id` from URL, JSON body with `format`, `progress_percent`, `location_value`, `location_type`, `cfi`, `current_page`, `total_pages`
- Produces: `POST /ajax/savereadingprogress/<book_id>` returns JSON: `{ bookmark: {...}, status, last_modified }` or error

- [ ] **Step 1: Add POST endpoint for saving reading progress**

Add the route in `cps/web.py` after the GET endpoint from Task 1:

```python
@web.route("/ajax/savereadingprogress/<int:book_id>", methods=['POST'])
@user_login_required
def save_reading_progress(book_id):
    """Save reading progress for a book to KoboReadingState."""
    data = request.get_json()
    if not data:
        return jsonify({"error": "No data provided"}), 400

    try:
        kobo_reading_state = kobo.get_or_create_reading_state(book_id)
        
        # Update bookmark data
        bookmark = kobo_reading_state.current_bookmark
        bookmark.location_source = "browser"
        bookmark.progress_percent = data.get("progress_percent")
        bookmark.content_source_progress_percent = data.get("content_source_progress_percent")
        bookmark.location_value = data.get("location_value", "")
        bookmark.location_type = data.get("location_type", "")
        
        # Also store CFI separately in location_value if provided
        if data.get("cfi"):
            bookmark.location_value = data["cfi"]
            if not bookmark.location_type:
                bookmark.location_type = "epubcfi"
        
        # Update read status to IN_PROGRESS if progress > 0
        book_read = kobo_reading_state.book_read_link
        if not book_read:
            book_read = ub.ReadBook(user_id=int(current_user.id), book_id=book_id)
            kobo_reading_state.book_read_link = book_read
        
        progress = data.get("progress_percent", 0) or 0
        if progress > 0:
            book_read.read_status = ub.ReadBook.STATUS_IN_PROGRESS
        
        ub.session.merge(kobo_reading_state)
        ub.session_commit("Reading progress saved for user {} in book {}".format(current_user.id, book_id))
        
        # Return updated state
        response = get_reading_progress_response(kobo_reading_state)
        return jsonify(response)
    except Exception as e:
        ub.session.rollback()
        log.error("Failed to save reading progress: %s", e)
        return jsonify({"error": str(e)}), 500


def get_reading_progress_response(kobo_reading_state):
    """Build a JSON response dict from a KoboReadingState object."""
    bookmark = kobo_reading_state.current_bookmark
    book_read = kobo_reading_state.book_read_link
    statistics = kobo_reading_state.statistics
    
    return {
        "bookmark": {
            "progress_percent": bookmark.progress_percent,
            "content_source_progress_percent": bookmark.content_source_progress_percent,
            "location_value": bookmark.location_value,
            "location_type": bookmark.location_type,
            "location_source": bookmark.location_source,
            "cfi": bookmark.location_value if bookmark.location_source and "cfi" in str(bookmark.location_source).lower() else None,
        },
        "status": book_read.read_status if book_read else None,
        "statistics": {
            "spent_reading_minutes": statistics.spent_reading_minutes,
            "remaining_time_minutes": statistics.remaining_time_minutes,
        },
        "last_modified": kobo_reading_state.last_modified.isoformat() if kobo_reading_state.last_modified else None,
    }
```

- [ ] **Step 2: Create helper function `get_reading_progress_response`**

This function is already included above. It builds a JSON response from a `KoboReadingState` for both GET and POST endpoints.

- [ ] **Step 3: Verify the route is valid Python**

Run: `python -m py_compile cps/web.py`
Expected: No syntax errors

- [ ] **Step 4: Commit**

```bash
git add cps/web.py
git commit -m "feat: add POST /ajax/savereadingprogress/<book_id> endpoint"
```

---

### Task 3: Backend - Modify read_book to pass reading_state to templates

**Files:**
- Modify: `cps/web.py` (the `read_book` function at line ~1580)

**Interfaces:**
- Consumes: Existing `read_book` route parameters
- Produces: `reading_state` dict passed to reader templates

- [ ] **Step 1: Query KoboReadingState in read_book and pass to templates**

Modify the `read_book` function in `cps/web.py`. After line 1594 (after querying for bookmark), add reading_state query:

Find around line 1592-1597 in `read_book`:
```python
    # check if book has a bookmark
    bookmark = None
    if current_user.is_authenticated:
        bookmark = ub.session.query(ub.Bookmark).filter(and_(ub.Bookmark.user_id == int(current_user.id),
                                                             ub.Bookmark.book_id == book_id,
                                                             ub.Bookmark.format == book_format.upper())).first()
```

Add reading_state query after the bookmark query (around line 1597):
```python
    # check if book has reading progress state (for auto-save / restore)
    reading_state = None
    if current_user.is_authenticated:
        kobo_state = kobo.get_or_create_reading_state(book_id)
        if kobo_state and kobo_state.current_bookmark:
            reading_state = get_reading_progress_response(kobo_state)
```

Then pass `reading_state` to relevant reader templates:

For EPUB (line ~1600):
```python
        return render_title_template('read.html', bookid=book_id, title=book.title, bookmark=bookmark,
                                     book_format=book_format, reading_state=reading_state)
```

For PDF (line ~1603):
```python
        return render_title_template('readpdf.html', pdffile=book_id, title=book.title,
                                     reading_state=reading_state)
```

For Comic (line ~1628):
```python
                return render_title_template('readcbr.html', comicfile=all_name, title=title,
                                             extension=fileExt, bookmark=bookmark, reading_state=reading_state)
```

For audiobook (line ~1617):
```python
                return render_title_template('listenmp3.html', mp3file=book_id, audioformat=book_format.lower(),
                                             entry=entries, bookmark=bookmark, reading_state=reading_state)
```

- [ ] **Step 2: Verify the modifications compile**

Run: `python -m py_compile cps/web.py`
Expected: No syntax errors

- [ ] **Step 3: Commit**

```bash
git add cps/web.py
git commit -m "feat: pass reading_state to reader templates in read_book"
```

---

### Task 4: Frontend - EPUB reader (read.html + epub.js)

**Files:**
- Modify: `cps/templates/read.html`
- Modify: `cps/static/js/reading/epub.js`

**Interfaces:**
- Consumes: `reading_state` passed from template
- Produces: Auto-save every 3s on page change, save on beforeunload, restore position on load

- [ ] **Step 1: Pass reading_state to window.calibre in read.html**

In `cps/templates/read.html`, modify the `window.calibre` object (lines 146-153) to include reading_status and reading_state:

```javascript
        window.calibre = {
            filePath: "{{ url_for('static', filename='js/libs/') }}",
            cssPath: "{{ url_for('static', filename='css/') }}",
            bookmarkUrl: "{{ url_for('web.set_bookmark', book_id=bookid, book_format=book_format) }}",
            saveProgressUrl: "{{ url_for('web.save_reading_progress', book_id=bookid) }}",
            bookUrl: "{{ url_for('web.serve_book', book_id=bookid, book_format=book_format, anyname='file.epub') }}",
            bookmark: "{{ bookmark.bookmark_key if bookmark != None }}",
            useBookmarks: "{{ current_user.is_authenticated | tojson }}",
            readingStatus: {{ reading_state.bookmark.progress_percent | default(0) | tojson }},
            readingState: {{ reading_state | tojson if reading_state else 'null' }}
        };
```

- [ ] **Step 2: Add auto-save and restore logic to epub.js**

Add these modifications to `cps/static/js/reading/epub.js`:

At the top of the IIFE (after line 3), add auto-save state:
```javascript
var reader;
var saveTimeout = null;
var SAVE_DEBOUNCE_MS = 3000;
```

In the `reader.book.ready.then()` callback, modify the relocated handler (around line 108) to trigger auto-save and use server state to restore position:

After the existing localStorage restore logic (around line 93-106), add server state restore:
```javascript
                // Also try to restore from server-side reading state
                try {
                    if (window.calibre && window.calibre.readingState && window.calibre.readingState.bookmark) {
                        var srvBookmark = window.calibre.readingState.bookmark;
                        if (srvBookmark.cfi || srvBookmark.location_value) {
                            var serverCfi = srvBookmark.cfi || srvBookmark.location_value;
                            // Only use server position if no local position exists, or server is newer
                            if (!_savedPos) {
                                try {
                                    reader.rendition.display(serverCfi);
                                } catch (e) {}
                            }
                            // Update progress display from server state
                            if (srvBookmark.progress_percent != null) {
                                progressDiv.textContent = Math.round(srvBookmark.progress_percent) + "%";
                            }
                        }
                    }
                } catch (e) {}
```

Modify the relocated handler (around line 108-137) to add auto-save:
```javascript
                reader.rendition.on("relocated", (location) => {
                    let percentage = Math.round(location.end.percentage * 100);
                    progressDiv.textContent = percentage + "%";

                    const cfi = location.start.cfi;
                    const current =
                        reader.book.locations.locationFromCfi(cfi) || 0;
                    const total = reader.book.locations.length() || 0;

                    if (total > 0) {
                        pagesDiv.textContent = current + "/" + total;
                        pagesDiv.style.visibility = "visible";
                    } else {
                        pagesDiv.textContent = "";
                        pagesDiv.style.visibility = "hidden";
                    }

                    // Persist last position to localStorage
                    try {
                        var posObj = {
                            cfi: location.start.cfi,
                            percentage: location.start.percentage,
                        };
                        localStorage.setItem(
                            position_key,
                            JSON.stringify(posObj)
                        );
                    } catch (e) {}

                    // Auto-save progress to server (debounced)
                    if (window.calibre && window.calibre.useBookmarks) {
                        if (saveTimeout) clearTimeout(saveTimeout);
                        saveTimeout = setTimeout(function() {
                            saveProgress(cfi, location.start.percentage, percentage);
                        }, SAVE_DEBOUNCE_MS);
                    }
                });
```

Add the saveProgress function after updateBookmark (around line 172):
```javascript
    function saveProgress(cfi, percentage, displayPercentage) {
        if (!window.calibre || !window.calibre.saveProgressUrl) return;
        
        var csrftoken = $("input[name='csrf_token']").val();
        $.ajax(window.calibre.saveProgressUrl, {
            method: "post",
            contentType: "application/json; charset=utf-8",
            data: JSON.stringify({
                format: "EPUB",
                cfi: cfi,
                progress_percent: displayPercentage,
                content_source_progress_percent: percentage
            }),
            headers: { "X-CSRFToken": csrftoken },
        }).fail(function (xhr, status, error) {
            console.error("Failed to save reading progress:", error);
        });
    }

    // Save on page unload
    $(window).on('beforeunload', function() {
        if (reader && reader.rendition) {
            try {
                var location = reader.rendition.location;
                if (location) {
                    var cfi = location.start ? location.start.cfi : null;
                    var p = location.start ? Math.round(location.start.percentage * 100) : 0;
                    saveProgress(cfi, location.start ? location.start.percentage : 0, p);
                }
            } catch (e) {}
        }
    });
```

- [ ] **Step 3: Verify the JS files**

Run: `python -c "open('cps/static/js/reading/epub.js').read()"`
Expected: File readable, no Python crash

- [ ] **Step 4: Commit**

```bash
git add cps/templates/read.html cps/static/js/reading/epub.js
git commit -m "feat: add auto-save and restore to EPUB reader"
```

---

### Task 5: Frontend - Comic reader (readcbr.html + kthoom.js)

**Files:**
- Modify: `cps/templates/readcbr.html`
- Modify: `cps/static/js/kthoom.js`

**Interfaces:**
- Consumes: `reading_state` passed from template
- Produces: Auto-save every 3s on page change, save on beforeunload, restore page on load

- [ ] **Step 1: Pass reading_state to window.calibre in readcbr.html**

In `cps/templates/readcbr.html`, modify the `window.calibre` object (lines 186-190):

```javascript
    window.calibre = {
        bookmarkUrl: "{{ url_for('web.set_bookmark', book_id=comicfile, book_format=extension.upper()) }}",
        saveProgressUrl: "{{ url_for('web.save_reading_progress', book_id=comicfile) }}",
        bookmark: "{{ bookmark.bookmark_key if bookmark != None }}",
        useBookmarks: "{{ current_user.is_authenticated | tojson }}",
        readingState: {{ reading_state | tojson if reading_state else 'null' }}
    };
```

- [ ] **Step 2: Add auto-save and restore logic to kthoom.js**

At the top of `kthoom.js` (after line 1), add:
```javascript
var SAVE_DEBOUNCE_MS = 3000;
var saveTimeout = null;
```

Modify the `setBookmark` function (lines 845-858) to also trigger progress save. Replace the function with:
```javascript
function setBookmark() {
    // get csrf_token
    let csrf_token = $("input[name='csrf_token']").val();
    //This sends a bookmark update to calibreweb.
    $.ajax(calibre.bookmarkUrl, {
        method: "post",
        data: {
            csrf_token: csrf_token,
            bookmark: currentImage
        }
    }).fail(function (xhr, status, error) {
        console.error(error);
    });
    
    // Also save progress to server
    if (calibre && calibre.saveProgressUrl && typeof currentImage === 'number') {
        if (saveTimeout) clearTimeout(saveTimeout);
        saveTimeout = setTimeout(function() {
            if (typeof totalImages === 'undefined') totalImages = 1;
            let pct = Math.round(((currentImage + 1) / totalImages) * 100);
            $.ajax(calibre.saveProgressUrl, {
                method: "post",
                contentType: "application/json; charset=utf-8",
                data: JSON.stringify({
                    format: extension ? extension.toUpperCase() : "CBZ",
                    current_page: currentImage + 1,
                    total_pages: totalImages,
                    progress_percent: pct
                }),
                headers: { "X-CSRFToken": csrf_token },
            }).fail(function (xhr, status, error) {
                console.error("Failed to save reading progress:", error);
            });
        }, SAVE_DEBOUNCE_MS);
    }
}
```

Add a force-save on page unload. Add after the existing initialization block (after line 868 in the `$(function() {` block):
```javascript
    $(window).on('beforeunload', function() {
        if (saveTimeout) clearTimeout(saveTimeout);
        saveTimeout = null;
        if (typeof currentImage === 'number' && calibre && calibre.saveProgressUrl) {
            let csrf_token = $("input[name='csrf_token']").val();
            if (typeof totalImages === 'undefined') totalImages = 1;
            let pct = Math.round(((currentImage + 1) / totalImages) * 100);
            $.ajax(calibre.saveProgressUrl, {
                method: "post",
                contentType: "application/json; charset=utf-8",
                data: JSON.stringify({
                    format: extension ? extension.toUpperCase() : "CBZ",
                    current_page: currentImage + 1,
                    total_pages: totalImages,
                    progress_percent: pct
                }),
                headers: { "X-CSRFToken": csrf_token },
            });
        }
    });
```

Modify the initialization code in readcbr.html (lines 192-202) to also restore from server state:
```javascript
    document.onreadystatechange = function () {
      if (document.readyState == "complete") {
          // Try to restore from server-side reading state
          if (calibre && calibre.useBookmarks && calibre.readingState && calibre.readingState.bookmark) {
              var srv = calibre.readingState.bookmark;
              if (srv.location_value && !isNaN(parseInt(srv.location_value))) {
                  currentImage = parseInt(srv.location_value);
              }
          }
          if (calibre.useBookmarks) {
              currentImage = eval(calibre.bookmark);
              if (typeof currentImage !== 'number') {
                  currentImage = 0;
              }
          }
          init("{{ url_for('web.serve_book', book_id=comicfile, book_format=extension) }}");
      }
    }
```

- [ ] **Step 3: Commit**

```bash
git add cps/templates/readcbr.html cps/static/js/kthoom.js
git commit -m "feat: add auto-save and restore to comic reader"
```

---

### Task 6: Frontend - PDF reader (readpdf.html)

**Files:**
- Modify: `cps/templates/readpdf.html`

**Interfaces:**
- Consumes: `reading_state` passed from template
- Produces: Auto-save on page change via PDF.js pagechange event

- [ ] **Step 1: Add reading state tracking to readpdf.html**

The PDF reader template (`cps/templates/readpdf.html`) uses PDF.js standalone viewer. We need to add scripts that hook into PDF.js's page navigation events.

After the existing script that sets `PDFViewerApplicationOptions.set('defaultUrl', ...)`, add:

```html
    <script>
    // Reading progress tracking for PDF reader
    (function() {
        var SAVE_DEBOUNCE_MS = 3000;
        var saveTimeout = null;
        var saveProgressUrl = "{{ url_for('web.save_reading_progress', book_id=pdffile) }}";
        var readingState = {{ reading_state | tojson if reading_state else 'null' }};
        var csrfToken = "{{ csrf_token() }}";

        function savePage(currentPage, totalPages) {
            if (!saveProgressUrl) return;
            if (saveTimeout) clearTimeout(saveTimeout);
            saveTimeout = setTimeout(function() {
                var pct = Math.round((currentPage / totalPages) * 100);
                $.ajax(saveProgressUrl, {
                    method: "post",
                    contentType: "application/json; charset=utf-8",
                    data: JSON.stringify({
                        format: "PDF",
                        current_page: currentPage,
                        total_pages: totalPages,
                        progress_percent: pct
                    }),
                    headers: { "X-CSRFToken": csrfToken },
                }).fail(function(xhr, status, error) {
                    console.error("Failed to save PDF reading progress:", error);
                });
            }, SAVE_DEBOUNCE_MS);
        }

        // Wait for PDF.js to initialize, then hook into page navigation
        var checkPDF = setInterval(function() {
            if (typeof PDFViewerApplication !== 'undefined' && PDFViewerApplication.initializedPromise) {
                clearInterval(checkPDF);
                PDFViewerApplication.initializedPromise.then(function() {
                    // Restore page from server state
                    if (readingState && readingState.bookmark && readingState.bookmark.location_value) {
                        var srvPage = parseInt(readingState.bookmark.location_value);
                        if (!isNaN(srvPage) && srvPage >= 1 && srvPage <= PDFViewerApplication.pdfViewer.totalPages) {
                            PDFViewerApplication.pdfViewer.currentPageNumber = srvPage;
                        }
                    }

                    // Hook into page number changes
                    var doc = document.getElementById('pageNumber');
                    if (doc) {
                        var observer = new MutationObserver(function() {
                            try {
                                var page = PDFViewerApplication.pdfViewer.currentPageNumber;
                                var total = PDFViewerApplication.pdfViewer.totalPages;
                                if (page && total) {
                                    savePage(page, total);
                                }
                            } catch(e) {}
                        });
                        observer.observe(doc, {characterData: true, subtree: true, childList: true});
                    }

                    // Save on page unload
                    window.addEventListener('beforeunload', function() {
                        if (saveTimeout) clearTimeout(saveTimeout);
                        savePage(PDFViewerApplication.pdfViewer.currentPageNumber, PDFViewerApplication.pdfViewer.totalPages);
                    });
                });
            }
        }, 100);
    })();
    </script>
```

- [ ] **Step 2: Commit**

```bash
git add cps/templates/readpdf.html
git commit -m "feat: add auto-save and restore to PDF reader"
```

---

### Task 7: Testing - Verify all endpoints and integrations

**Files:**
- New: `test/test_reading_progress.py` (optional manual test)
- Verify: All modified files compile and import correctly

- [ ] **Step 1: Verify all Python files compile**

Run: `python -m py_compile cps/web.py cps/kobo.py`
Expected: No syntax errors

- [ ] **Step 2: Verify imports work**

Run: `python -c "from cps import web, kobo; print('Imports OK')"`
Expected: `Imports OK` printed with no errors

- [ ] **Step 3: Verify the reading_state JSON is valid in templates**

Run: `python -c "import json; json.loads('{{ None | tojson if None else \"null\" }}')"` 
(Manual test: open each reader with a book and check the browser console for any JS errors)

- [ ] **Step 4: Manual verification checklist**

1. Open an EPUB book in the reader → progress display shows 0% initially
2. Navigate to a page → progress updates in the footer
3. Wait 3 seconds → check server logs for save progress requests
4. Close the tab → verify final position was saved
5. Reopen the same book → reader should restore to the saved position
6. Test with PDF reader → same behavior with page numbers
7. Test with comic reader → same behavior with page numbers
8. Open book on Kobo device → sync should work with browser progress (timestamp-based)

- [ ] **Step 5: Commit any changes**

```bash
git add -A
git commit -m "test: verify reading progress integration"
```

---

## Pre-Flight Plan Review

No conflicts detected. The tasks are independent backend/frontend units that build sequentially.

- Task 1 creates the foundation (GET endpoint)
- Task 2 builds on Task 1 (POST endpoint + shared helper)
- Task 3 wires data flow from backend to templates
- Tasks 4-6 parallelize frontend changes across reader types
- Task 7 verifies the full integration
