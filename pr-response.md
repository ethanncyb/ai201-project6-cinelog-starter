# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used an AI assistant to speed up orientation and reduce documentation omissions:
- Codebase orientation: compared `services/collection_service.py` + `tests/test_collection.py` patterns to mirror how CineLog handles domain errors, deduplication, and test structure for the watchlist feature.
- Design stress-test: drafted the initial positions for Comment 4 (visibility default) and Comment 5 (sort order) and then refined them to explicitly address the tradeoffs and the maintainer’s “added recently” point with CineLog-specific context.
- Documentation: converted the implemented behavior into a reviewer-friendly PR description and concrete manual testing steps for the `/watchlist/<user_id>` endpoints.

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
This PR implements CineLog’s **watchlist** feature (simulated code review).

What’s included:
- A new watchlist domain model (`WatchlistEntry`) and service functions in `services/watchlist_service.py`.
- A new watchlist API blueprint with endpoints:
  - `GET /watchlist/<user_id>`: returns the user’s watchlist items (including `date_added` and `public`).
  - `POST /watchlist/<user_id>/add`: adds a film to the user’s watchlist by `film_id`.

Design decisions (explicit):
- **Default visibility**: `public = True` for newly added watchlist entries.
- **Sort order**: watchlist results are returned **newest-first** by `date_added` (descending).

Manual testing (end to end):
1. Start the app (in the project root):
   ```bash
   pip install -r requirements.txt
   python app.py
   ```
   The server should run at `http://127.0.0.1:5000`.

2. Create a test `user` and `film` (so you have valid UUIDs to call the watchlist endpoints):
   ```bash
   python - <<'PY'
   from app import create_app, db
   from models import User, Film

   app = create_app()
   with app.app_context():
       db.create_all()
       user = User(username="watchlist_user", email="watchlist_user@example.com")
       film = Film(title="Watchlist Test Film", year=2026, genre="Drama")
       db.session.add_all([user, film])
       db.session.commit()
       print(user.id)
       print(film.id)
   PY
   ```
   Copy the printed values into `USER_ID` and `FILM_ID`.

3. Add the film to the watchlist:
   ```bash
   curl -X POST "http://127.0.0.1:5000/watchlist/$USER_ID/add" \
     -H "Content-Type: application/json" \
     -d "{\"film_id\":\"$FILM_ID\"}"
   ```
   Expect `201` and a JSON response containing `film_id` and `public` (should be `true` by default).

4. Fetch the watchlist and verify:
   ```bash
   curl "http://127.0.0.1:5000/watchlist/$USER_ID"
   ```
   Verify:
   - The returned list contains the film you added.
   - Each item includes `date_added` and `public`.
   - The first item is the most recently added one.

5. Verify sort order (newest-first):
   - Repeat step (2) to create a second `film_id` (a newer film entry).
   - Repeat step (3) for that second film.
   - Run step (4) again and confirm the second film appears before the first in the returned list.

6. Verify deduplication behavior:
   - Run step (3) again with the same `film_id`.
   - Confirm you do not end up with duplicate entries for the same `(user_id, film_id)` (the watchlist should not grow additional items for that same film).
