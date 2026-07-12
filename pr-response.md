# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

I used Claude to help generate curl commands to help test the add_to_watchlist() function to see if the fucntion runs as expected for comment 1. I realized that there wasn't any function to seed any data, so when I was running the curl command, it was leading into SQLAlchemy related errors. I prompted the error statement, and utilized Claude to help seed in data in cinelog.db to be able to test the curl command as expected, with using a database visualizer to see the generated IDs.

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
**What I did:**
**How I verified:**

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