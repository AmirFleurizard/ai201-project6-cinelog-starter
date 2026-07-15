# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used Claude as an AI tool throughout this project for codebase
orientation, pattern matching, and conflict resolution guidance.

**Instance 1 - Codebase orientation:**
I gave Claude the contents of models.py, collection_service.py, and
test_collection.py and asked it to summarize the naming conventions,
deduplication pattern, and test structure. It correctly identified the
verb_to_noun naming convention and the pattern used in
add_to_collection() for deduplication. I verified both against the
actual code before applying them to the watchlist service.

**Instance 2 - Rebase conflict resolution:**
When the rebase produced a conflict between my branch's models.py and
main's refactored version, I asked Claude to help me understand what
had changed and what needed updating. It identified that WatchlistEntry
had been dropped during the rebase and that film_id needed to change
from Integer to String(36) to match the UUID refactor. I verified this
by reading the diff output and confirmed the fix by running the test
suite.

**Instance 3 - Design decision stress-testing (Comments 4 and 5):**
After drafting my responses for default visibility and sort order, I
asked Claude what counterarguments a careful reviewer might raise. For
Comment 4, it raised the concern about surprising users who didn't
realize their list was public, I incorporated this into my tradeoff
acknowledgment. For Comment 5, it confirmed that consistency with
get_collection() sort order was a strong argument, which I included
in my reasoning. My final arguments are my own reasoning grounded in
CineLog's context — Claude helped me identify gaps, not write the
responses.

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

### Watchlist Feature — Add films to a personal watchlist

This PR adds a watchlist feature to CineLog so users can save films
they want to watch later, separate from their collection of films
they've already seen.

**What's included:**
- `WatchlistEntry` model with `user_id`, `film_id`, `date_added`,
  and `public` fields
- `add_to_watchlist(user_id, film_id)` service function with film
  existence check and deduplication
- `get_watchlist(user_id)` service function returning entries sorted
  by date added (newest first)
- `GET /watchlist/<user_id>` endpoint to retrieve a user's watchlist
- `POST /watchlist/<user_id>/add` endpoint to add a film
- Tests for nonexistent film, duplicate entry, and basic add

**Design decisions:**

*Default visibility (`public=True`):*
Watchlist entries default to public because CineLog is a community
platform where social discovery is the core value. Defaulting to
private would require users to opt in to social features, reducing
the community experience for new users. Users who want privacy can
opt out. The visibility setting should be surfaced clearly in the UI.

*Sort order (date-added descending):*
Watchlist entries are sorted by date added, newest first — matching
the sort order of `get_collection()`. A watchlist functions as a
queue of films to watch next, and the most recently added film is
usually the most top-of-mind. This also makes the API consistent
across both list features.

**How to manually test:**

1. Start the app: `python app.py`

2. Add a film to a user's watchlist:
```bash
curl -s -X POST http://127.0.0.1:5000/watchlist//add \
  -H "Content-Type: application/json" \
  -d '{"film_id": ""}' | python -m json.tool
```

3. View the user's watchlist:
```bash
curl -s http://127.0.0.1:5000/watchlist/ | python -m json.tool
```

4. Verify deduplication — add the same film twice and confirm the
   second request returns an error, not a duplicate entry.

5. Verify nonexistent film — use a fake UUID as film_id and confirm
   FilmNotFoundError is returned.

6. Run the full test suite: `pytest tests/ -v` — all 7 tests should pass.