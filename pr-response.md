# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used AI for codebase orientation and workflow guidance during this project. Specifically, I used it to sanity-check which files on the `feature/watchlist` branch needed changes, to compare the watchlist work against the existing `collection_service.py` patterns, and to help turn `tests/test_collection.py` into a matching `tests/test_watchlist.py` structure. I still verified all code changes manually in the repo and confirmed behavior by running the test suite locally.

## Comment 1 — Rename
**What I did:**
I renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` so the watchlist service follows the project’s `verb_to_noun` naming convention. I also updated the call site in the watchlist route so the POST endpoint now calls `add_to_watchlist(...)` instead of the old function name. While I was there, I updated the route/service documentation to refer to `film_id` as a UUID instead of an integer.

**How I verified:**
I checked the watchlist route import and POST handler to confirm the old name was no longer used. Then I ran the full test suite with `python -m pytest tests -v` to make sure the rename did not break imports or execution flow.

## Comment 2 — Deduplication
**What I did:**
I added deduplication logic to `add_to_watchlist()` so the service checks for an existing `WatchlistEntry` for the same `user_id` and `film_id` before inserting a new row. If a duplicate is found, the function now raises `DuplicateWatchlistEntryError` instead of silently creating multiple watchlist rows for the same film.

**Reasoning:**
I followed the same general pattern used by the collection service: validate the film exists first, then check for an existing entry, then insert only if no duplicate is present. That keeps the watchlist behavior consistent with the rest of the codebase and avoids relying on a database error to enforce business rules.

**How I verified:**
I added a watchlist duplicate test in `tests/test_watchlist.py` and confirmed that adding the same film twice raises the expected exception and leaves only one row in the database. I also ran the full test suite afterward.

## Comment 3 — Missing test
**What I did:**
I created `tests/test_watchlist.py` and used `tests/test_collection.py` as the model for fixture setup, test structure, and assertion style. At minimum, I added the required nonexistent-film test for `add_to_watchlist()`, and I also added additional watchlist tests for basic insert behavior, duplicate handling, and current sort behavior.

**Reasoning:**
Using the collection tests as the template kept the watchlist test file consistent with the codebase’s existing style. While writing the watchlist tests, I also found a real implementation issue in `get_watchlist()`: the function assumed `WatchlistEntry` exposed a `.film` relationship, but on this branch it did not. I updated the service so it retrieves the film record explicitly by `film_id`, which made the test pass and made the service behavior match the actual model setup on the branch.

**How I verified:**
I ran `python -m pytest tests -v` and confirmed that both the collection tests and the new watchlist tests passed.

## Comment 4 — Default visibility
**My position:**
I would keep `public=True` as the default for watchlist entries.

**Reasoning:**
CineLog is framed as a community film tracking app, so the default behavior should support discovery and social visibility unless a user chooses otherwise. A public-by-default watchlist lowers friction for the common case where users are using the product to signal taste, track future viewing, or share recommendations. That default also aligns with the “community” framing of the app better than making every entry opt-in private.

**Tradeoff acknowledged:**
The main downside is privacy: some users may reasonably assume a watchlist is personal planning space rather than social output. A private-by-default model would be safer for cautious users, but it would also add friction and make the watchlist feel less connected to CineLog’s community features. If privacy becomes a stronger product priority later, I would revisit this and consider clearer UI around visibility rather than changing the default immediately.

## Comment 5 — Sort order
**My position:**
I would change the watchlist sort order to date added (newest first) instead of alphabetical title order.

**Reasoning:**
A watchlist is usually a planning tool, and in that context recency is more meaningful than title ordering. Newest-first helps users quickly find the films they most recently saved, which is often the most relevant slice of the list when they return to it. It also matches the existing collection behavior more closely, which already treats date-added order as important.

**Engagement with reviewer’s point:**
I understand the case for alphabetical order because it can make scanning easier in a long static list, especially when someone is searching mentally by title. But in CineLog’s context, a watchlist is more dynamic than a catalog, and users are likely to revisit it based on what they just added rather than to browse it like a directory. If alphabetical lookup becomes important later, I would rather support that with filtering or client-side sorting controls than make alphabetical the only default order.

## Comment 6 — Rebase
**What conflicted:**
While my watchlist branch was open, the main branch moved film IDs from integer-based assumptions to UUID-based identifiers. The watchlist code still had pre-refactor wording and assumptions in the route/service docs, and I needed to make sure the watchlist branch matched the updated ID model before resubmitting.

**How I resolved it:**
I rebased `feature/watchlist` onto the updated `main` branch and resolved the watchlist changes on top of the UUID-based code. I updated watchlist-related code and documentation to treat `film_id` as a UUID, kept the branch linear with no merge commits, and reran the tests after the rebase.

**How I verified no conflict remains:**
I reran the full test suite after rebasing and checked the branch history to confirm the branch was rebased cleanly rather than merged.

## PR Description
This PR completes the watchlist review fixes for CineLog’s `feature/watchlist` branch. I renamed the service function to `add_to_watchlist()` to match project naming conventions, added duplicate protection in the watchlist service, created a watchlist test file based on the collection test structure, and fixed watchlist service behavior uncovered while writing those tests.

For design decisions, I kept `public=True` as the default because CineLog is positioned as a community film-tracking product and public watchlists support lower-friction sharing and discovery. I also chose date-added ordering as the better default sort for the watchlist because it reflects the way users are likely to revisit recently saved films more than they browse the list alphabetically.

### Manual testing steps
1. Start the app with `python app.py`.
2. Send a POST request to `/watchlist/<user_id>/add` with a valid UUID `film_id` and confirm the entry is created.
3. Repeat the same POST request with the same `film_id` and confirm the service rejects the duplicate.
4. Send a POST request with a nonexistent UUID `film_id` and confirm the service raises the expected not-found behavior.
5. Send a GET request to `/watchlist/<user_id>` and confirm the saved films are returned with watchlist metadata.
6. Run `python -m pytest tests -v` and confirm the full suite passes.