# PR Response Doc — CineLog Watchlist Feature

## AI Usage

## Comment 1 — Rename
**What I did:**
Renamed `save_to_watchlist()` to `add_to_watchlist()` in
`services/watchlist_service.py` and updated the import and call site
in `routes/watchlist/watchlist.py`. Used a project-wide search to
confirm those were the only two locations referencing the old name.

**How I verified:**
Ran `pytest tests/ -v` after the rename confirming all 4 existing tests passed,
confirming no call sites were missed and nothing broke.

## Comment 2 — Deduplication
**What I did:**
Added a `AlreadyInWatchlistError` exception class to
`services/watchlist_service.py`, following the same pattern as
`AlreadyInCollectionError` in `collection_service.py`. Added a
deduplication check in `add_to_watchlist()` that queries for an
existing `WatchlistEntry` with the same `user_id` and `film_id`
before creating a new one. If a match is found, raises
`AlreadyInWatchlistError` instead of creating a duplicate.

**How I verified:**
Ran `pytest tests/ -v` after the change and confrimed all tests passed. The
deduplication behavior is also directly tested in `test_watchlist.py`
(Comment 3).

## Comment 3 — Missing test
**What I did:**
Created `tests/test_watchlist.py` following the exact fixture and
assertion structure from `tests/test_collection.py`. Used the same
`app`, `sample_user`, and `sample_film` fixture pattern. Wrote three
tests: one for nonexistent film_id raising `FilmNotFoundError`
(directly requested by the reviewer), one for duplicate entries raising
`AlreadyInWatchlistError`, and one for the basic happy-path case
confirming a `WatchlistEntry` is created and persisted.

**How I verified:**
Ran `pytest tests/test_watchlist.py -v` and confrimed all 3 new tests passed.
Ran `pytest tests/ -v` and confrimed all 7 tests passed with no regressions.

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default.

**Reasoning:**
CineLog is a community film tracking app and at its core value proposition
is social discovery. A watchlist that defaults to private gives new
users a worse first time experience. They add films and see nothing social
happen. Defaulting to public means the community features work
immediately without requiring users to opt in through a settings page
they may never find.

The analogy I'd draw is to Letterboxd, which defaults all lists to
public. The expectation on a film community platform is that sharing
is the norm, not the exception. Users who want privacy can opt out,
but most users on a social platform don't expect their watchlist to
be hidden.

**Tradeoff acknowledged:**
The risk is surprising users who didn't realize their list was public. 
I'd suggest surfacing the visibility setting clearly in the UI when a 
user creates or views their watchlist, so the default is transparent even 
if it's public. If the product direction shifts toward a more private-first 
model, changing the default is a one-line fix in models.py.

## Comment 5 — Sort order
**My position:** Switch to date-added order (newest first).

**Reasoning:**
A watchlist functions as a queue, users add films when they feel
motivated to watch them, and the most recently added film is usually
the one they're most interested in watching next. Alphabetical order
optimizes for lookup by title, but most users on a watchlist aren't
scanning for a specific title, they're browsing what to watch next.
Date-added order serves that browsing behavior better.

There's also a consistency argument: get_collection() already sorts
by date_added descending. Having both lists use the same sort order
makes the API more predictable for anyone building a frontend.

**Engagement with reviewer's point:**
The reviewer's reasoning - "most users want to see what they added
recently" which directly matches how I think about watchlist behavior.
I implemented alphabetical initially because it felt predictable, but
upon reflection that's optimizing for a use case (finding a specific
known title) that's less common than browsing. I'm switching to
date-added descending to match both user behavior and the existing
collection sort order.

## Comment 6 — Rebase
**What conflicted:**
Two conflicts arose during the rebase. First, `.gitignore` had an
add/add conflict because main already had a `.gitignore` commit that
included `.pytest_cache/` while my branch added one without it.
Second, the rebase brought in main's refactored `models.py` which
changed `Film.id` from Integer to UUID, but this replaced my branch's
`models.py` which contained the `WatchlistEntry` model, dropping it
entirely.

**How I resolved it:**
For the `.gitignore` conflict, I kept both sets of entries merged into
one clean file including `.pytest_cache/` from main, then ran
`git rebase --continue`. For the missing `WatchlistEntry`, I re-added
the class to `models.py` with `film_id` updated from
`db.Column(db.Integer, ...)` to `db.Column(db.String(36), ...)` to
match main's UUID-based `Film.id`. Also updated the `watchlist_service.py`
docstring to reflect `film_id` is now a UUID string, not an integer.

**How I verified no conflict remains:**
Ran `pytest tests/ -v` and all 7 tests pass. Ran `git log --oneline`
to confirm no merge commits in the branch history.

## PR Description
