# Mixtape Bug Hunt — Submission

## AI Usage

I used Claude as a pair-debugging partner throughout this project — mainly to help me navigate the codebase and verify my findings, not to hand me answers I didn't understand.

- **Orientation:** Before opening any issue, I had Claude walk me through what each file (`app.py`, `models.py`, each `routes/` and `services/` file) is responsible for, and how a request flows from route → service → database. That's reflected in the codebase map below.
- **Reproducing before fixing:** For each bug, I ran small scripts against the app's own database to trigger the exact reported behavior first — e.g. simulating a Saturday-then-Sunday listen for the streak bug, or inserting a listening event a controlled number of hours old for the feed bug — before changing any code.
- **A real mistake I made and caught:** When fixing the notification bug (#4), I pasted the new notification code into the wrong function the first time, which accidentally broke the *existing* `add_to_playlist()` notification logic. I only caught it because I re-ran `pytest tests/` and asked Claude to show me the actual current contents of the file with `grep`, rather than assuming my edit had landed correctly. That's the same lesson as Milestone 3 in the brief — verify by running the code, don't assume the edit worked.
- **A second mistake in the feed fix:** When shrinking the "Listening Now" window, I initially typed `timedelta(hours=30)` instead of `timedelta(minutes=30)` — an easy typo, and one that actually made the bug *worse* (a 30-hour window instead of 30 minutes). My reproduction script caught it immediately because the "23.9 hours ago" case still showed `True` when it should have shown `False`. This is exactly why the brief has you reproduce a bug with concrete inputs before and after the fix — a silent typo like this would have been invisible without that check.
- **Issue #3/#5 (search duplicates) — where Claude's first read of the code was wrong:** The `outerjoin` against `song_tags` with no `.distinct()` looks like a textbook cause of duplicate rows. Claude initially expected this to reproduce cleanly. I tested it directly: a raw SQL query for a 3-tag song genuinely returned 3 duplicate rows, but calling the actual `search_songs()` function returned only 1 result, and the existing `test_search.py` suite already passed. It turned out this version of SQLAlchemy auto-deduplicates full-entity ORM results by primary key, even when the underlying SQL has repeats — so the bug wasn't reaching users, even though the query itself was still fragile. I verified this myself (raw SQL vs. ORM result, side by side) rather than taking either "it's obviously broken" or "it's obviously fine" at face value. I fixed it anyway by adding `.distinct()`, since relying on undocumented version-specific ORM behavior isn't something I want to leave in place even if it happens to work right now.

---

## Codebase Map

```
ai201-project5-mixtape-starter/
├── app.py                      # Flask app factory + SQLAlchemy init; registers 4 blueprints
├── models.py                   # SQLAlchemy models + 3 association tables
├── routes/                     # Thin HTTP layer — parses request, calls a service, formats JSON
│   ├── songs.py                # /songs — search, get, rate, listen
│   ├── playlists.py            # /playlists — create, get, list songs, add song
│   ├── users.py                # /users — profile, streak, notifications
│   └── feed.py                 # /feed — listening-now, activity
├── services/                   # All business logic lives here
│   ├── streak_service.py       # Listening streak increment/reset rules
│   ├── feed_service.py         # "Friends listening now" + activity feed
│   ├── search_service.py       # Song search by title/artist
│   ├── notification_service.py # Notification creation/retrieval, ratings
│   └── playlist_service.py     # Playlist CRUD + ordered song retrieval
├── tests/                      # pytest, one file per service
├── seed_data.py                # Populates 5 users, 13 songs, 3 playlists, listening history
└── requirements.txt
```

**Models (`models.py`):** `User`, `Song`, `Tag`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, plus three association tables — `friendships` (symmetric, both directions inserted manually in seed data), `song_tags` (plain many-to-many), and `playlist_entries` (many-to-many *with* a `position` column, which is how playlist ordering works — there's no ordering field on `Song` itself).

**Pattern:** every route is a thin wrapper — pull fields out of the request, call exactly one service function, translate the result or a raised `ValueError` into JSON + a status code. All actual logic lives in `services/`. To trace any bug: find the route, note the one service call it makes, read that function.

**Data flow — a friend rates a shared song:**
`POST /songs/<song_id>/rate` (`routes/songs.py::rate`) reads `user_id` and `score` and calls `notification_service.rate_song(user_id, song_id, score)`. That function validates the score, looks up the `Song` and the rating `User`, checks for an existing `Rating` (unique per user+song) to decide update-vs-insert, commits, and — after my fix — now also notifies the song's original sharer, matching the pattern already used by `add_to_playlist()`.

**Data flow — a friend adds a shared song to a playlist:**
`POST /playlists/<id>/songs` (`routes/playlists.py::add_song`) calls `notification_service.add_to_playlist()`, which loads the `Song`, `User`, and `Playlist`, appends the song to `playlist.songs` if not already present, commits, then — if the person adding it isn't the song's original sharer — calls `create_notification()`. This "notify unless it was you" pattern is what `rate_song()` was missing (Issue #4).

---

## Root Cause Analysis

### Issue #1 — My listening streak keeps resetting

**How I reproduced it:** In a Python shell, I set a user's `last_listened_at` to a Saturday, then called `update_listening_streak()` with a datetime for the following Sunday (a genuine consecutive day). The streak reset to `1` instead of incrementing to `6`.

**How I found the root cause:** `README.md` pointed to `streak_service.py`. Reading `update_listening_streak()` line by line, the issue was this condition: `elif days_since_last == 1 and today.weekday() != 6:` — the extra `and today.weekday() != 6` has nothing to do with "one day since last listen," which is what the function's own docstring describes.

**The root cause:** Python's `datetime.weekday()` returns `6` for Sunday. The increment branch only fires when exactly one day has passed *and* today isn't a Sunday. When a consecutive-day listen lands on a Sunday, that second condition is `False`, execution falls to the `else` branch, and the streak resets to `1` even though only one day passed.

**My fix and side-effect check:** Removed the `and today.weekday() != 6` clause, leaving `elif days_since_last == 1:`. This doesn't touch the same-day (`days_since_last == 0`) or skipped-day (`else`) branches. Ran the full `test_streaks.py` suite (5/5 pass, including an existing `test_streak_increments_on_sunday` test) and re-confirmed the Saturday→Sunday reproduction now correctly goes from 5 to 6.

---

### Issue #2 — The last song in a playlist never shows up

**How I reproduced it:** Called `get_playlist_songs()` on a seeded playlist with 7 songs (confirmed independently via the `playlist.songs` relationship) and got back only 6.

**How I found the root cause:** The function builds `songs` as a correctly ordered query result. The bug was the last line: `return [song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice drops the last element of the list every time, regardless of playlist length.

**The root cause:** A stray `[:-1]` slice on the return statement silently discards the last song in every playlist. It isn't conditional — it happens on every call.

**My fix and side-effect check:** Changed `songs[:-1]` to `songs`. Checked `get_user_playlists()` and `get_playlist()` in the same file — neither touches this list. Ran `test_playlists.py` (3/3 pass, including one that explicitly asserts the count is 5) and confirmed the reproduction case now returns all 7 songs.

---

### Issue #3 — Notified for playlist adds, but not for ratings

**How I reproduced it:** Called `rate_song(rater_id, song_id, score)` for a song shared by a different user, then checked `get_notifications(sharer_id)` before and after. The count didn't change.

**How I found the root cause:** Compared `add_to_playlist()` and `rate_song()` side by side in `notification_service.py`. `add_to_playlist()` ends with an `if song.shared_by != added_by_user_id:` check and a `create_notification()` call. `rate_song()` saves the rating and returns — there's no equivalent block. This isn't a broken condition; the notification step was never written for this path.

**The root cause:** `rate_song()` is missing the "notify the sharer" step that `add_to_playlist()` already has.

**My fix and side-effect check:** Added the same "notify unless the sharer is the one acting" pattern to `rate_song()`, creating a `song_rated` notification. While making this change I initially pasted the new code into the wrong function by mistake, which broke `add_to_playlist()`'s existing notification — caught by re-running `pytest tests/` and inspecting the file directly rather than assuming the edit worked. After correcting both functions, verified with a before/after notification count test (0 → 1) and confirmed the full suite (13/13) still passes.

---

### Issue #4 — Friends Listening Now shows people from yesterday

**How I reproduced it:** Inserted a `ListeningEvent` for a friend timestamped 23.9 hours before "now" and called `get_friends_listening_now()`. The friend showed up — something that happened almost a full day ago isn't a reasonable definition of "now."

**How I found the root cause:** The query logic (the `>= cutoff` filter, the dedup-by-most-recent-song loop) all looked correct once traced. The actual cause was the module-level constant the whole function is built on: `RECENT_THRESHOLD = timedelta(hours=24)`. Testing the boundary directly (23.9h → shown, 24.0h → not shown) confirmed the filter works exactly as written — the *window* itself is just too wide for a feature meant to mean "right now."

**The root cause:** A 24-hour rolling window means anyone who listened at any point in the previous day still counts as "currently listening" today. The seed data's own comments describe "recent" events as ones within the past 30 minutes — the intended window was always meant to be short.

**My fix and side-effect check:** Changed `RECENT_THRESHOLD` to `timedelta(minutes=30)` — on the first attempt I actually typed `timedelta(hours=30)` by mistake, which made the window even wider than before. My own reproduction test caught this immediately (23.9h still showed `True`), which is exactly why the brief has you verify with concrete before/after inputs rather than trusting that an edit did what you meant. After correcting it to `minutes=30`, re-ran the boundary test (23.9h → False, 1h → False, 0.4h → True) and the full suite (13/13 pass). Checked `get_activity_feed()` in the same file — it doesn't use `RECENT_THRESHOLD` at all (explicitly documented as unfiltered by recency), so it's unaffected.

---

### Issue #5 — The same song keeps showing up twice in search

**How I reproduced it:** First attempt — searching for a 3-tag song ("Crown Heights Anthem") through `search_songs()` returned exactly 1 result, not 3. The existing `test_search.py` suite also passed already. I didn't stop there, since "the obvious bug doesn't reproduce" is itself worth verifying rather than assuming. I compared two things side by side: a raw SQL query for the same join, and the ORM-level `search_songs()` call. The raw SQL genuinely returned the same song ID 3 times (once per tag). The ORM call still returned exactly 1.

**How I found the root cause:** The `outerjoin` against `song_tags` (a many-to-many table) creates one row per matching tag in raw SQL — a song with 3 tags produces 3 rows sharing the same song ID. This version of SQLAlchemy's legacy `Query()` API happens to auto-deduplicate full-entity results by primary key before `.all()` returns them, so the duplication doesn't reach the caller here — but that's version-specific ORM behavior, not something the query itself guarantees.

**The root cause:** The query relies on an undocumented, version-dependent auto-dedup behavior instead of explicitly asking for distinct rows. It happens not to manifest as a user-facing bug in this environment, but the query is still fragile — a different SQLAlchemy version or a different way of writing the same join could bring the duplicates back.

**My fix and side-effect check:** Added `.distinct()` to the query so it's correct on its own terms regardless of ORM version. Re-ran `search_songs('Crown')` (still returns exactly 1 result with all 3 tags) and the full test suite (13/13 pass).

---

## Commits

All fixes are on `bugfix/mixtape`, one commit per fix:

```
98ac4f7 fix: add .distinct() to search query to prevent duplicate results
c96e163 fix: shrink Friends Listening Now window from 24h to 30 minutes
92a0e6d fix: send a notification when a friend rates a shared song
4daa5db fix: stop dropping the last song from playlist results
966ee83 fix: remove spurious Sunday check that reset listening streaks
2dfdeaa Add .gitignore file and update README with setup instructions
7b64551 initial commit
```