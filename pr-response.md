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
**My position:** `public=True` is the correct default for CineLog watchlists.
**Reasoning:** CineLog is a community film tracking app — the social layer is the product, not an add-on. `CollectionEntry` (watched films) has no visibility flag at all and is implicitly shared; watchlists should follow the same spirit. On platforms users will compare this to (Letterboxd, Goodreads), watchlists default to public because the core value prop is discovery: seeing what friends plan to watch, getting recommendations from people with similar taste, and finding films through the community rather than an algorithm. Making the default `public=True` also reflects the most common user journey — someone signs up, adds films they're excited about, and naturally expects others to see that activity. Power users who want privacy can set `public=False` per entry, which puts the control in their hands without gatekeeping the primary use case behind a setting most new users won't find.
**Tradeoff acknowledged:** A `public=False` default would be privacy-first — users never accidentally expose a watchlist entry they'd rather keep private (guilty pleasures, surprise gifts for a partner, professional research they don't want colleagues to see). If we had a global account-level privacy toggle, I'd want that to default to private and let users opt in. But at the per-entry level, `public=True` is the right launch default. I'm open to revisiting if we add a `default_visibility` user preference in a future iteration.

## Comment 5 — Sort order
**My position:** I'll implement the maintainer's preference — `WatchlistEntry.date_added.desc()`, newest first.
**Reasoning:** I chose alphabetical initially because it makes a list scannable: if you have 50 films on your watchlist and want to find one, alphabetical is predictable. But the maintainer's argument convinced me when I thought about actual usage. A watchlist isn't a static catalog you consult like a dictionary — it's a queue. Users add films as they hear about them, and the ones they added most recently are the ones they're actively thinking about. The identical sort order in `get_collection()` (also newest-first) reinforces that consistency: across the app, "date added, newest first" is the canonical order. Alphabetical sort is a feature users can request later, but the launch default should surface what's fresh.
**Engagement with reviewer's point:** You're right that "most users want to see what they added recently" — the watchlist is a working list, not an archive. I'll change the `order_by` from `Film.title.asc()` to `WatchlistEntry.date_added.desc()` in `get_watchlist()`, remove the now-unnecessary `.join(Film)`, and add a sort-order test to `tests/test_watchlist.py` modeled after `test_get_collection_returns_newest_first`.

## Comment 6 — Rebase
**What conflicted:** Only `.gitignore` had merge conflicts — both the feature branch and `origin/main` had independently added entries to it. The conflict was trivial to resolve by keeping both sets of rules.
**How I resolved it:** Rebased with `git fetch origin && git rebase origin/main`, manually resolved the `.gitignore` conflict, and continued. No other files conflicted. However, this created a silent data loss: `WatchlistEntry` existed in the initial commit `014ae54` but was removed from `models.py` in the main-branch refactor at `07ca580` ("migrate film IDs from integer to UUID"). None of the feature commits touched `models.py`, so git saw no conflict — it just silently took main's version without the class. This is a rebase blind spot: git only conflicts when both sides change the same lines, not when one side removes code the other side depends on.
**How I verified no conflict remains:** Ran `pytest tests/ -v` and discovered `ImportError: cannot import name 'WatchlistEntry' from 'models'`. Re-added the `WatchlistEntry` class to `models.py`, adapting `film_id` from `db.Integer` to `db.String(36)` to match the UUID migration, and added an explicit `film = db.relationship("Film", backref="watchlist_entries")` since the refactor no longer provided an implicit backref. Full suite passes (6/6). Key lesson: always run `git rebase --exec "pytest tests/"` to catch breakage at the offending commit rather than discovering it afterward.

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->
