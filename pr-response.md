# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

I used Claude to help generate curl commands to help test the add_to_watchlist() function to see if the fucntion runs as expected for comment 1. I realized that there wasn't any function to seed any data, so when I was running the curl command, it was leading into SQLAlchemy related errors. I prompted the error statement, and utilized Claude to help seed in data in cinelog.db to be able to test the curl command as expected, with using a database visualizer to see the generated IDs.

I used Claude to help guide me through rebasing, since I accidently opened another rebase and noticed a lot of my changes disappeared. It eventually led me to abort my rebase which retrived my previous rebase!

I used Claude to help understand the tradeoffs of visibility with `public=True` and understand potential design decisions to make and eventual tradeoffs that may occur. I included these in my final discussion for this topic/

Additionally, I used Claude to generate the test case for comment 3 and testing comment 2. I made sure to follow up to ensure the test works as intended and clarified any potential problems I had with the issue, such as with the test case using an exisiting database or not with generating test 3's test case. I ran pytest to ensure these new tests work as intended.

## Comment 1 — Rename
**What I did:** I used CTRL-SHIFT-F to find all instances of save_to_watchlist() mentioned. I used all of these function names and their references to add_to_watchlist() as the comment mentioned. I additionally made sure to change the name of the import statement to ensure the tests are ran correctly. 
**How I verified:** I ran `pytest` to ensure there is not any compilation errors. I further tested by using CURL to call the related endpoint and see if the function runs successfully. 

```
curl -X POST http://localhost:5000/watchlist/1/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": 1}'
```


## Comment 2 — Deduplication
**What I did:** Previously when a duplicate is detected, the function returns an unhandled database `IntegrityError` or silently succeeds with a new duplicate `WatchlistEntry` as the function doesn't have an explicit check for duplicate entries. To prevent this, I added `UniqueContraint`s in models.py to the `user_id` and `film_id` columns for the `WatchlistEntry` table. This ensures that when a new entry is created with matching `user_id` and `film_id` values, that a relevant error (`AlreadyInWatchlistError`) is raised and is handled with further error-checking implemented in `add_to_watchlist()`

**How I verified:** I asked Claude to generate the following test case then later ran pytest after verifying the test.

```python
def test_add_to_watchlist_duplicate_raises(app, sample_user, sample_film):
    """
    Adding the same film to a user's watchlist twice should raise
    AlreadyInWatchlistError, not silently create a duplicate entry.
    """
    with app.app_context():
        add_to_watchlist(user_id=sample_user, film_id=sample_film)

        with pytest.raises(AlreadyInWatchlistError):
            add_to_watchlist(user_id=sample_user, film_id=sample_film)

        # Confirm only one entry exists
        count = WatchlistEntry.query.filter_by(
            user_id=sample_user, film_id=sample_film
        ).count()
        assert count == 1
```

This test case passed, signifying that deduplication logic is handled as expected. 

## Comment 3 — Missing test
**What I did:** I added the `test_add_to_watchlist_nonexistent_film_raises` test case to the `test_watchlist.py`. This function operates by creating a fake film id, then looking for the `FilmNotFoundError` returned in the passing scenario for the test case in `add_to_watchlist`
**How I verified:** I ran `pytest tests/test_watchlist.py -v` to test if the test case works. I noticed that there could be a scenario that the fake film ID could be randomly generated, so I further prompted Claude to ask if this would be an issue, where the response mentioned that implementing this would be redundant since the `app` fixture resets the SQLite database.


## Comment 4 — Default visibility
**My position:** `public=True` refers to the database models for `WatchlistEntry` where in the current state of the app, all entries by default are publicly viewable by other users. My position on this is that the public column should stay as True.

**Reasoning:** In production, CineLog would only be valuable to users if Watchlist entries are visible to other viewers. This default setting would minimize the changes in the API to set entries to publicly not viewable, especially if reviews are intended to be sharable by design. Automatically privated entries may lose CineLog in engagement as fewer posts would be shared to users.

**Tradeoff acknowledged:** It's important to recognize that there could be some movies that are not socially acceptable to like or would be more questionable for a person to watch. Maybe a user is embarrassed to share they watched a certain movie. Additionally, it could be possible that the user is embarassed or does not want to share a polarizing rating to the average user. Having public-by-default posts may leak these moments that do not want to be shared, which could harm the platform's userbase.

## Comment 5 — Sort order
**My position:** I would use a sort order based on the time/date added to both `get_watchlist()` and `get_collection()`. If there are entries with the same time frame, then I would sort alphabetically.

**Reasoning:** Users would most likely want to access their most recent contributions, sorting by recency would ensure their most recent contributions would be at the top, making it easier for the user to access in production, or the API to view for what the user wants (a recent contribution) faster. 

**Engagement with reviewer's point:** We share a similar point, but I wanted to emphasize a potential edge case. The original code also sorts `get_collection()` by alphabetical, which could be better search capability by the user. However, I would argue for better consistency with using date added for the reason mentioend above. Additionally, it would allow the user to see changes made faster also.

## Comment 6 — Rebase
**What conflicted:** I had a merge conflicts with `.gitignore` and `models.py`.
**How I resolved it:** I accepted all changes from my commits using the VSCode editor. I had ChatGPT assist with what to do with what to do within the terminal. 
**How I verified no conflict remains:** When I merged, I ensured code is still functional by running `pytest` and reviewing the changes once the code is rebased.

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->

**Feature Overview**

Adds a Watchlist feature to CineLog, letting a user save films they intend to watch later. This mirrors the existing Collection feature's structure (model, service, routes) but tracks intent-to-watch rather than a rated, already-watched entry.

**New endpoints**
- `GET /watchlist/<user_id>` — returns the user's watchlist as a list of films, each annotated with `date_added` and `public`.
- `POST /watchlist/<user_id>/add` — body `{ "film_id": <int> }`; adds a film to the user's watchlist.
  - `404` if the film doesn't exist (`FilmNotFoundError`)
  - `409` if the film is already on the user's watchlist (`AlreadyInWatchlistError`)
  - `201` with the created entry on success

**New model:** `WatchlistEntry` (`models.py`) — `id`, `user_id` (FK → `user`), `film_id` (FK → `film`), `date_added`, `public` (defaults to `True`), with a `UniqueConstraint` on `(user_id, film_id)` to enforce one watchlist entry per film per user at the database level.

### Design decisions
- **Naming:** the service function is named `add_to_watchlist()` (not `save_to_watchlist()`) for consistency with `add_to_collection()`.
- **Deduplication:** rather than letting a duplicate insert raise a raw `IntegrityError`, `add_to_watchlist()` checks for an existing `(user_id, film_id)` entry up front and raises a dedicated `AlreadyInWatchlistError`, which the route maps to `409`. The `UniqueConstraint` on `WatchlistEntry` is a defense-in-depth backstop against race conditions, not the primary error path.
- **Default visibility (`public=True`):** watchlist entries are public by default, matching the social/discovery value of the app — most users want their activity visible, and defaulting to private would suppress engagement. Acknowledged tradeoff: some users may want privacy for embarrassing or polarizing picks; that's left as a per-entry toggle rather than changing the default.
- **Sort order:** `get_watchlist()` currently sorts alphabetically by film title (matching `get_collection()`'s existing behavior). Discussed switching both to sort by `date_added` (most-recent-first, falling back to alphabetical for ties) for consistency and to surface recent activity faster — flagged as a follow-up, not implemented in this PR.

### Manual testing
Ran the app locally and exercised both endpoints with seeded data (a user and film row) via curl:

```
curl -X POST http://localhost:5000/watchlist/1/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": 1}'

curl http://localhost:5000/watchlist/1
```

Additionally, using pytest tests changes that are made do not inhibit other parts of the code.