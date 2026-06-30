# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:**

Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` and updated the single route call site in `routes/watchlist/watchlist.py`.

**How I verified:**
- Searched the repo for `save_to_watchlist(` to confirm there were no remaining references.
- Ran `pytest tests/ -v` to confirm the import + route wiring still works and nothing else broke.

## Comment 2 — Deduplication
**What I did:**
Added an explicit deduplication check to `add_to_watchlist()` so adding the same film twice for the same user raises an error instead of creating duplicate rows. I introduced `AlreadyInWatchlistError` to make the failure mode explicit.

**How I verified:**
I modeled the logic directly on `services/collection_service.py:add_to_collection()`:
- Validate the film exists first (raise `FilmNotFoundError` if not).
- Query for an existing `(user_id, film_id)` entry and raise if present.

Then I re-ran `pytest tests/ -v` to ensure the existing suite stayed green.

## Comment 3 — Missing test
**What I did:**
Created `tests/test_watchlist.py` and added `test_add_to_watchlist_nonexistent_film_raises`, which asserts that calling `add_to_watchlist()` with a non-existent `film_id` raises `FilmNotFoundError`.

**How I verified:**
Modeled the structure and assertions after `tests/test_collection.py::test_add_to_collection_nonexistent_film_raises` (same fixture pattern, same “raise a domain error, not an integrity error” intent). Verified by running `pytest tests/ -v` and confirming the new test passes.

## Comment 4 — Default visibility
**My position:**
Keep the default as **public = True** for new watchlist entries.

**Reasoning:**
In CineLog’s context (“community film tracking”), a watchlist is most valuable when it supports lightweight social discovery: “what are my friends excited to watch?” A default-public watchlist optimizes for the most common low-friction workflow (save a film → it shows up in the watchlist and is shareable without extra steps). This aligns with the idea that the watchlist is a *curation surface* rather than a private notes space.

**Tradeoff acknowledged:**
Default-public does create a privacy risk for users who treat watchlists as personal (e.g., sensitive topics, guilty-pleasure viewing, or simply not wanting others to see intent-to-watch). The alternative default (private) optimizes for privacy-by-default but adds repeated friction for the many users who want to share; I’m choosing to optimize for CineLog’s community/discovery value, with the expectation that a future UX affordance (visibility toggle) can address the privacy case cleanly.

## Comment 5 — Sort order
**My position:**
Change the watchlist sort order to **date-added (newest first)**.

**Reasoning:**
When a user opens a watchlist, the most likely task is to act on what they *recently* decided to watch (or to confirm “did I already save this?”). Newest-first supports that workflow directly and keeps the “current intent” at the top. It also matches CineLog’s existing `get_collection()` behavior (newest-first), which makes the API more predictable across similar endpoints.

**Engagement with reviewer's point:**
I agree with the maintainer’s argument that most users care about what they added recently. Alphabetical is useful for scanning a large list, but it’s better served by client-side sorting/search once a UI exists. At the service layer, newest-first provides higher default utility and consistency with other CineLog endpoints.

## Comment 6 — Rebase
**What conflicted:**
The `feature/watchlist` branch was originally built on the pre-refactor model where `Film.id` (and related foreign keys like `WatchlistEntry.film_id`) were integers. `main` refactored film IDs to UUID strings.

**How I resolved it:**
Rebased `feature/watchlist` onto `origin/main` and updated the watchlist code to match the UUID-based model:
- Ensured `WatchlistEntry.film_id` is a UUID string FK to `Film.id`
- Updated watchlist service + route docs/tests to treat `film_id` as a UUID string

**How I verified no conflict remains:**
Confirmed the branch history is linear after rebase and re-ran `pytest tests/ -v` to ensure the suite passes on the rebased code.

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->
