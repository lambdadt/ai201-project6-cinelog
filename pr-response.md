# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` (line 17) to follow the project's `verb_to_noun` naming convention. Updated the import (`line 8`) and the call (`line 32`) in `routes/watchlist/watchlist.py` — the only call site.
**How I verified:** Ran a project-wide grep for `save_to_watchlist` across all source files. The only matches were in `.git/logs` (commit history), confirming no call sites were missed. The test `test_add_to_watchlist_nonexistent_film_raises` imports and calls the renamed function and passes.

## Comment 2 — Deduplication
**What I did:** Added deduplication logic to `add_to_watchlist()` in `services/watchlist_service.py` (lines 35–41), following the exact pattern from `add_to_collection()` in `services/collection_service.py` (lines 47–53): query `WatchlistEntry` for an existing row with the same `user_id`/`film_id`, and raise `AlreadyInCollectionError` if one exists.
**How I verified:** Ran `pytest tests/ -v` — all 5 tests pass, including the existing collection dupe test (`test_add_to_collection_duplicate_raises`) and the new watchlist test, confirming the logic doesn't break anything and that the `AlreadyInCollectionError` is importable and functional. I used `test_add_to_collection_duplicate_raises` (tests/test_collection.py:78) as the model.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` with `test_add_to_watchlist_nonexistent_film_raises`, modeled after `test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py:98–107`. Uses the same fixtures (`app`, `sample_user`), same `with app.app_context()` wrapper, and same `pytest.raises(FilmNotFoundError)` assertion.
**How I verified:** Ran `pytest tests/test_watchlist.py -v` — 1 passed. Then ran the full suite with `pytest tests/ -v` to confirm no regressions — all 5 tests pass. I used `test_add_to_collection_nonexistent_film_raises` as the model test.

## Comment 4 — Default visibility
**My position:**
**Reasoning:**
**Tradeoff acknowledged:**

## Comment 5 — Sort order
**My position:**
**Reasoning:**
**Engagement with reviewer's point:**

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->
