# PR Response Doc — CineLog Watchlist Feature

## AI Usage
Used Claude Code (claude-sonnet-4-6) to help identify and implement all six review comments. I directed the work: checked out the branch, understood each comment, reviewed the diffs, and verified the changes. Claude wrote the code edits and tests; I reviewed each change and ran the test suite to confirm correctness.

## Comment 1 — Rename
**What I did:**
Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` and updated the call site in `routes/watchlist/watchlist.py` to match. The new name follows the project's `verb_to_noun` convention established by `add_to_collection()` and `remove_from_collection()`.

**How I verified:**
Searched all files for `save_to_watchlist` — no references remain. All tests pass with the new name.

## Comment 2 — Deduplication
**What I did:**
Added `AlreadyInWatchlistError` exception class to `services/watchlist_service.py`. In `add_to_watchlist()`, I query for an existing `WatchlistEntry` with the same `user_id` + `film_id` before inserting and raise the error if one is found. I also added a `UniqueConstraint("user_id", "film_id")` to the `WatchlistEntry` model as a database-level guard. The route handler in `routes/watchlist/watchlist.py` catches `AlreadyInWatchlistError` and returns a 409 response.

**How I verified:**
`test_add_to_watchlist_duplicate_raises` confirms the error is raised and only one entry exists after two add attempts.

## Comment 3 — Missing test
**What I did:**
Added `test_add_to_watchlist_nonexistent_film_raises` in `tests/test_watchlist.py`. It passes a fake UUID (`00000000-0000-0000-0000-000000000000`) to `add_to_watchlist()` and asserts that `FilmNotFoundError` is raised — matching the pattern from `test_add_to_collection_nonexistent_film_raises` in `test_collection.py`.

**How I verified:**
`pytest tests/` — 8 passed, 0 failed. The new test is included in that count.

## Comment 4 — Default visibility
**My position:**
I kept `public=True` as the default.

**Reasoning:**
CineLog is described as a *community* film tracking app. Users who add films to a watchlist in a social context generally want those lists discoverable — that is the primary value of a community tracker. Defaulting to public maximises that value out of the box without requiring users to opt in.

**Tradeoff acknowledged:**
A public-by-default setting means users who want privacy have to take action. This is the right tradeoff for a community app, but it should be clearly documented in onboarding. The `public` boolean is stored per-entry, so a future update can add per-entry or per-user privacy controls without a schema change.

## Comment 5 — Sort order
**My position:**
Changed the sort order from alphabetical (`Film.title.asc()`) to date added descending (`WatchlistEntry.date_added.desc()`).

**Reasoning:**
I agree with the reviewer. A watchlist grows over time, and the films a user added most recently are the ones they are most likely actively thinking about watching. Alphabetical order treats the watchlist like a static catalogue rather than a live queue. Sorting newest-first matches the behaviour of `get_collection()` and is consistent with how most list-based apps surface recent activity.

**Engagement with reviewer's point:**
The reviewer explicitly said they were open to discussion but wanted a documented decision. Newest-first is the right default; alphabetical can be added later as an optional `?sort=` query parameter if users ask for it.

## Comment 6 — Rebase
**What conflicted:**
The `feature/watchlist` branch was based on the pre-refactor state of `models.py` where `Film.id` was an integer primary key. The `main` branch had since merged a commit (`07ca580`) that migrated `Film.id` to a UUID (`String(36)`). The `WatchlistEntry` model on the watchlist branch used `db.Column(db.Integer, db.ForeignKey("film.id"))` for `film_id`, which was now inconsistent with the UUID `Film.id` on main.

**How I resolved it:**
Ran `git rebase origin/main` to replay the watchlist commits on top of main. The rebase applied cleanly but the `WatchlistEntry` model additions were lost during the auto-merge of `models.py`. I manually updated `models.py` to add `WatchlistEntry` with `film_id = db.Column(db.String(36), db.ForeignKey("film.id"))` to match the UUID type. I also updated the stale docstring in `watchlist_service.py` that still read `film_id (int): ... (Note: integer — pre-refactor)` to correctly document `film_id (str): UUID of the film.`

**How I verified no conflict remains:**
`pytest tests/` — all 8 tests pass against the rebased codebase, including tests that create films (with auto-generated UUID IDs) and add them to the watchlist.

## Commit History (`git log --oneline` on `feature/watchlist`)

```
eb0832d test: add watchlist service tests covering happy path, dedup, and sort order
45da074 feat: add Watchlist service and route with deduplication and error handling
fbdf880 fix: update film retrieval to use db.session.get in collection service
54abe0b feat: add WatchListEntry model with UUID film_id and unique constraint
bbe206c Merge pull request #2 from ascherj/chore/add-gitignore
718a9a8 chore: add .gitignore for generated files
07ca580 refactor: migrate film IDs from integer to UUID
014ae54 feat: initial CineLog API with film collection feature
```

The four commits at the top of the branch (`54abe0b` through `eb0832d`) represent the watchlist feature, each scoped to one logical change per the project's contributing guidelines. The branch is rebased on top of `main` (no merge commits).

## PR Description

### Watchlist Feature

This PR adds a watchlist to CineLog — a list of films a user wants to watch in the future, distinct from their collection (films already watched).

**What it does:**
- `GET /watchlist/<user_id>` — returns the user's watchlist, sorted newest-added first
- `POST /watchlist/<user_id>/add` — adds a film to the watchlist; returns 404 if the film doesn't exist, 409 if it's already on the list
- Films on the watchlist carry a `public` boolean and a `date_added` timestamp in the response

**Design decisions:**
- Watchlist entries default to `public=True` (see Comment 4 above for full reasoning)
- Sort order is newest-first to surface recently-added films, consistent with `GET /collection/<user_id>`
- Deduplication is enforced at both the service layer (raises `AlreadyInWatchlistError` → 409) and the database layer (unique constraint on `user_id` + `film_id`)

**How to manually test:**
```bash
# Start the app
python app.py

# Add a film to a user's watchlist
curl -X POST http://localhost:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_uuid>"}'

# View the watchlist (should appear newest first)
curl http://localhost:5000/watchlist/<user_id>

# Try adding the same film again — should return 409
curl -X POST http://localhost:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_uuid>"}'

# Try a fake film_id — should return 404
curl -X POST http://localhost:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "00000000-0000-0000-0000-000000000000"}'
```
