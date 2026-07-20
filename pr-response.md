# PR Response Doc — CineLog Watchlist Feature

## AI Usage

I used Claude Code (an AI coding assistant) at a few points during this project:

- **Orientation:** I had it summarize `models.py`, `services/collection_service.py`, and `tests/test_collection.py` before I read the review comments, then read the files myself to confirm the summaries matched. One thing it flagged that I verified in the code: `add_to_collection()` raises a custom `AlreadyInCollectionError` from a `filter_by().first()` check rather than relying on the database unique constraint, which is the pattern I copied for Comment 2.
- **Stress-testing Comments 4 and 5:** After drafting my visibility and sort order responses, I asked what counterarguments a reviewer would raise. For Comment 4 it pushed on "privacy by default is the safer principle," which I had mentioned but hadn't really answered, so I added the part about the explicit `public` parameter and the low sensitivity of what a watchlist actually exposes. The positions themselves (keep `public=True`, agree on date-added sorting) were mine and are argued from this codebase, not from generic best practices.
- **Commit hygiene:** I ran my final `git log --oneline` output past it to check conventional commit format before taking the screenshot, then double checked against the conventional commits spec myself.

## Comment 1 — Rename

**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` and updated the two references in `routes/watchlist/watchlist.py` (the import on line 8 and the call inside the `POST /add` handler). The reviewer is right that the project convention is `verb_to_noun`, and `add_to_collection()` / `remove_from_collection()` / `get_collection()` all use `add`/`remove`/`get`, so `save` was the odd one out. CONTRIBUTING.md also calls this convention out explicitly.

**How I verified:** Ran `grep -rn "save_to_watchlist"` across the project after the change and got zero hits outside the virtualenv, so no call site was missed. Then ran the full test suite to confirm nothing broke.

## Comment 2 — Deduplication

**What I did:** Followed the exact pattern from `add_to_collection()`: query `WatchlistEntry` for an existing row with the same `user_id` and `film_id`, and if one exists, raise a custom exception instead of inserting. I defined `AlreadyInWatchlistError` in `services/watchlist_service.py` because the collection service keeps its own errors (`AlreadyInCollectionError` etc.) in its own module, so watchlist errors belong in the watchlist module. I also updated the `POST /watchlist/<user_id>/add` route to translate that error into a 409 response and `FilmNotFoundError` into a 404, since that's what `routes/collection.py` does and the watchlist route previously would have returned a 500 for both cases.

**How I verified:** Wrote `test_add_to_watchlist_duplicate_raises`, which adds the same film twice, asserts the second call raises, and asserts the row count is still 1. It passes, and the full suite passes.

## Comment 3 — Missing test

**What I did:** Created `tests/test_watchlist.py` using `test_add_to_collection_nonexistent_film_raises` as the model. I copied the fixture structure from `test_collection.py` (the in-memory app fixture, `sample_user`, `sample_film`) and wrote `test_add_to_watchlist_nonexistent_film_raises`, which passes a made-up UUID and asserts `FilmNotFoundError` is raised. While I was in there I also added the happy path test and the duplicate test, because CONTRIBUTING.md says any new service function needs all three (happy path, duplicate handling, nonexistent ID), and the watchlist service had none.

**How I verified:** `pytest tests/test_watchlist.py -v` passes, and `pytest tests/ -v` shows all 11 tests passing.

## Comment 4 — Default visibility

**My position:** Keep `public=True` as the default, but stop relying on it silently. I added an explicit `public` parameter to `add_to_watchlist()` and the endpoint so callers can opt out at creation time, and the decision is now documented here and in the PR description rather than just inherited from the model definition.

**Reasoning:** The README describes CineLog as a community film tracking app in its first sentence. The product loop is social: you see what other people want to watch, they see yours, and that's where recommendations and discussion come from. If watchlists default to private, most users will never flip the toggle (defaults are sticky in both directions), profiles will look empty, and the community side of the app quietly dies. I'm optimizing for the passive majority who sign up for a social film tracker and expect it to behave like one. It also matters what a watchlist actually exposes: intent to watch a film, nothing more. There are no ratings, no viewing history, and no personal data in a `WatchlistEntry` beyond the film reference and a date. That's about the lowest-sensitivity data in the app, which is why I think a public default is defensible here in a way it wouldn't be for something like viewing history.

**Tradeoff acknowledged:** The real cost is that some users will assume their watchlist is private and be surprised that it isn't. Private-by-default is the safer principle in general, and if CineLog ever attaches anything more revealing to watchlist entries, this decision should be revisited. The mitigation I shipped is the explicit `public` parameter, so clients can surface a visibility choice in the add flow instead of burying the default. And because visibility lives on each entry rather than being global, an account-level "private by default" setting could be layered on later without a migration.

## Comment 5 — Sort order

**My position:** I agree with the reviewer. I changed `get_watchlist()` to sort by `date_added` descending and removed the alphabetical ordering.

**Reasoning:** The strongest argument is already in the codebase: `get_collection()` returns newest first, and the watchlist and collection views are near-identical siblings that users will flip between. Having one list sorted by recency and the other alphabetically would feel broken even if neither order is wrong on its own. Beyond consistency, a watchlist is a queue of intentions, and the entry you just added is the one you're most likely to act on or want to confirm made it in. Alphabetical order only helps when you're scanning for one specific title, and lookup is better served by search or filtering (the `/films/` endpoint already does this with query params) than by the default sort of the whole list.

**Engagement with reviewer's point:** The reviewer said most users want to see what they added recently, and I think that's right, but I'd add that we don't have to fully choose. Each item in the response includes its `date_added`, so a client that wants alphabetical display can re-sort locally. The API default just needs to serve the common case, and recency is the common case. I also wrote `test_get_watchlist_returns_newest_first` so this decision is pinned by a test rather than living only in this doc.

## Comment 6 — Rebase

**What conflicted:** This one surprised me. `git rebase origin/main` replayed all my commits without a single textual conflict, but the branch was still broken afterward. The refactor on `main` didn't just change `Film.id` from an autoincrement integer to a UUID string, it also removed the `WatchlistEntry` model from `models.py` entirely. Since none of my commits touched the `WatchlistEntry` class definition, git had nothing to flag, and I ended up with watchlist code importing a model that no longer existed. The conflict was semantic rather than textual: running the tests immediately failed with `ImportError: cannot import name 'WatchlistEntry' from 'models'`.

**How I resolved it:** Restored the `WatchlistEntry` model in `models.py` with `film_id` as `db.Column(db.String(36), db.ForeignKey("film.id"))`, matching how `CollectionEntry.film_id` looks after the refactor, and kept the `watchlist_entries` relationship on `Film` that `get_watchlist()` needs. In the same commit I updated the docstrings in the watchlist service and routes that still described `film_id` as an integer, so the code and docs agree that film IDs are UUIDs everywhere.

**How I verified no conflict remains:** The full suite passes (11 tests), including the nonexistent-film test, which passes a UUID string as the id. I booted the app and confirmed all three watchlist routes register. `git log --merges origin/main..HEAD` returns nothing, so the branch history is linear with no merge commits, and `git status` shows a clean tree with no leftover conflict markers.

## Stretch features

**`remove_from_watchlist()`:** Implemented following the `remove_from_collection()` pattern: look up the entry, raise `NotInWatchlistError` if it isn't there, otherwise delete and return `True`. Exposed as `DELETE /watchlist/<user_id>/remove` with a 404 on the error, mirroring the collection route. Covered by two tests (successful removal, removing something not on the list).

**Second test:** My extra test is `test_get_watchlist_returns_newest_first`. I picked it for two reasons. First, it locks in the Comment 5 decision so a future refactor can't silently flip the sort order back. Second, it's the only test that exercises `get_watchlist()` end to end, and writing it caught a real bug: `get_watchlist()` calls `entry.film`, but `WatchlistEntry` had no relationship to `Film` (only `CollectionEntry` had the `backref`), so the GET endpoint would have crashed with an `AttributeError` on any non-empty watchlist. That fix is its own commit.

**Visibility toggle:** `add_to_watchlist()` now takes `public=True` as a keyword argument and the `POST /add` endpoint reads an optional `public` field from the request body. Omitting it keeps today's behavior. Tested in `test_add_to_watchlist_public_flag`, which checks both the explicit `False` case and the default.

## Commit history

Final `git log --oneline` for `feature/watchlist` (screenshot below):

```
4c785f5 fix: restore WatchlistEntry with UUID film_id after main branch refactor
36e8507 test: add tests for remove_from_watchlist and visibility flag
6cbef9d feat: add public visibility parameter to add_to_watchlist
02cbe64 feat: add remove_from_watchlist service function and endpoint
c58166f test: add test for get_watchlist date added sort order
c0157eb fix: sort watchlist by date added to match collection behavior
51fabd1 fix: add film relationship to WatchlistEntry so entries can serialize film data
8fefc62 test: add watchlist tests for add, duplicate, and nonexistent film
9bca9bc fix: add deduplication check to prevent duplicate watchlist entries
f80e366 fix: rename save_to_watchlist to add_to_watchlist per naming convention
947499d fix: update film retrieval method to use db.session.get in collection and watchlist services
1b0f88f feat: add watchlist model, service, and endpoints
```

![git log screenshot](git-log.png)

## PR Description

**What the feature does:** Adds a watchlist to CineLog so users can save films they want to watch later, separate from the collection of films they've already logged. It ships a `WatchlistEntry` model and three endpoints: `GET /watchlist/<user_id>` to view a watchlist, `POST /watchlist/<user_id>/add` to add a film, and `DELETE /watchlist/<user_id>/remove` to take one off. Adding a duplicate returns a 409 and a nonexistent film returns a 404, matching the collection endpoints.

**Design decisions:**

1. *Default visibility:* Watchlist entries default to `public=True` because CineLog is a community app and discovery is the point; a watchlist only exposes intent to watch a film, which is low-sensitivity. The tradeoff is that privacy is opt-out, so the endpoint now accepts an explicit `public` field to make that choice visible to clients. Full reasoning is under Comment 4 above.
2. *Sort order:* Watchlists return newest first by `date_added`, replacing the original alphabetical order. This matches `get_collection()` so the two list views behave consistently, and it serves the common case of checking what you just added. Full reasoning is under Comment 5 above.

**How to manually test:**

1. `python -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt`
2. `python app.py` to start the API on `http://127.0.0.1:5000` (no frontend, so use curl).
3. Seed a film and grab its UUID. Quickest way in a second terminal:
   ```
   python -c "
   from app import create_app, db
   from models import Film, User
   app = create_app()
   with app.app_context():
       f = Film(title='Paddington 2', year=2017, genre='Comedy')
       u = User(username='demo', email='demo@example.com')
       db.session.add_all([f, u]); db.session.commit()
       print('film:', f.id); print('user:', u.id)"
   ```
4. Add it to the watchlist (expect 201 with the new entry, `public: true`):
   ```
   curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" -d '{"film_id": "<film_uuid>"}'
   ```
5. Add the same film again (expect 409) and a made-up UUID (expect 404).
6. `curl http://127.0.0.1:5000/watchlist/<user_id>` and confirm the film appears, newest first if you add more than one.
7. Try a private entry with `-d '{"film_id": "<other_uuid>", "public": false}'` and confirm `"public": false` in the response.
8. Remove it (expect 200, then 404 if you repeat):
   ```
   curl -X DELETE http://127.0.0.1:5000/watchlist/<user_id>/remove \
     -H "Content-Type: application/json" -d '{"film_id": "<film_uuid>"}'
   ```
9. `pytest tests/ -v` should show all 11 tests passing.
