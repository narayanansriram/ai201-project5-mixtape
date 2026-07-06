## AI Usage

I used Claude Code throughout this project mainly for two things: setting up and running reproduction/verification tests against the actual code, and getting detailed explanations of what each function does before I relied on it.

**Detailed function explanations.** Before working on any bug, I had Claude go through each service file and explain what every function does — not just a one-line summary, but what it returns, what it assumes about its inputs, and what could cause it to behave unexpectedly. This is how I understood things like the routes → services pattern (every route delegates immediately to a service function) and how `add_to_playlist()` in `notification_service.py` works, which became the reference pattern for finding the missing notification in Issue #4.

**Setting up and running tests.** For every bug, Claude wrote small reproduction scripts (Python shell / app-context scripts, not part of the committed test suite) to confirm the bug before any fix, and follow-up scripts to verify the fix afterward on both sides of the relevant boundary condition — e.g. Saturday vs. Sunday for the streak bug, 11:59pm vs. 12:01am UTC for the feed bug, and a 1-song playlist for the playlist bug. For Issue #3 (search duplicates), Claude's test scripts were also what surfaced a non-obvious detail: the obvious fix (`.distinct()`) didn't look necessary from a normal test, because SQLAlchemy's legacy Query API silently de-duplicates results — Claude had to write a raw-SQL test to prove the underlying duplicate rows were still there. I reviewed each script and its output myself before accepting any fix as verified, and ran the full `pytest tests/` suite after each change to confirm nothing else broke.

---

## Codebase Map

/instance folder
    mixtape.db - contains database and contents

/routes folder
    feed.py - contains routes for
                - user's listening now
                - user's activity

    playlists.py - contains routes for
                - POST to create a playlist
                - GET to get playlist by playlist_id
                - GET songs in a playlist
                - POST to add songs to a playlist

    songs.py - contains routes for
                - GET search
                - GET for song details
                - POST to rate song
                - POST to mark listening of a song

    users.py - contains routes for
                - GET user by user_id
                - GET user streak
                - GET notifications for a user
                - POST to mark notification as read

services/ folder
    feed_service.py - contains methods
                - get_friends_listening_now which returns list of friends' recent listenings
                - get_activity_feed which returns recent listening events from all friends
    notification_service.py - contains methods
                - create_notification which creates a notification for a user
                - add_to_playlist - records that a user adds songs to a playlist and notifies the sharer - this makes use of create_notification to alert the sharer
                - rate_song - saves user's rating for a song
                - get_notifications - retrieves user notifications
                - mark_as_read - marks notifications as read
    playlist_service.py - contains methods
                - create_playlist - creates a playlist
                - get_playlist_songs - gets playlist songs
                - get_playlist - gets playlist metadata without songs
                - get_user_playlists - gets playlists created by a user
    search_service.py - contains methods
                - search_songs - searches for songs by title or artist
                - get_song - fetches song by id
    streak_service.py - contains methods
                - record_listening_event - records the user and song and updates streak
                - update_listening_streak - updates user's listening streak based on their last listening date
                - get_streak - retrieves streak

models.py - houses all the database models
    - generate_uuid - generates unique user id
    Association tables
        - friendships - user to friend; many-to-many
        - song tags - song to tag; many-to-many
        - playlist entries - playlist to song; many-to-many
    Tables
        - User - columns are username, email, listening_streak, last_listened_at, created_at. Relations to songs, ratings, listening events, notifications, playlists, friends
        - Tag - columns are names
        - Songs - columns are title, artist, album, genre, shared_by, shared_at, shared_note
        - Relationships are ratings, listening events and tags
        - Listening events - columns are user_id, song_id, listened_at
        - Rating - columns are user_id, song_id, score, rated_at
        - Playlist - columns are name, created_by, created_at, is_collaborative
        - Notification - columns are user_id, notification_type, body, created_at, read
Traces
- GET /users/<user_id>/streak → calls get_streak in streak_service → queries db to get user's listening streak using user_id
- POST /playlists/ → calls create_playlist in playlist_service → creates a Playlist entry in db, returns a dict → dict jsonified to caller
- POST /playlists/<playlist_id>/songs → add_to_playlist() → creates the playlist entry → calls create_notification() → {adder.username} added your song '{song.title}' to the playlist '{playlist.name}'.

Pattern
Router catches requests like GET, POST. Services contain methods that perform database operations -- all called from router methods. Some service functions also trigger side effects like notifications by calling other service functions directly like add_to_playlist calling create_notification

---

## Root Cause Analysis

BUG 1:
Issue number and title — #1 — My listening streak keeps resetting

How you reproduced it — In a Python shell (app context), I set a user's `listening_streak` to 5 and `last_listened_at` to a Saturday (2026-07-04), then called `update_listening_streak(user, sunday)` with `sunday = 2026-07-05` (the very next calendar day). Since the two listens were on consecutive days, I expected the streak to increment to 6. Instead it reset to 1. Repeating the same test with a non-Sunday consecutive pair (e.g. Tuesday → Wednesday) incremented correctly, which told me the bug was specific to landing on a Sunday.

How you found the root cause — README pointed me at `streak_service.py` for this issue. I read `update_listening_streak()` top to bottom: it computes `days_since_last` from the date difference, then branches on `days_since_last == 0` (same day, no-op), `days_since_last == 1` (consecutive day, increment), and everything else (reset to 1). The `days_since_last == 1` branch had an extra clause: `and today.weekday() != 6`. That immediately looked suspicious because it made the "consecutive day" branch conditional on the day of the week, which has nothing to do with whether two listens were consecutive. I confirmed by checking `datetime(2026,7,5).weekday()` returned `6`, meaning that any listen on a Sunday fails this condition and falls through to the `else`, resetting the streak.

The root cause — Python's `datetime.weekday()` returns 6 for Sunday (Monday=0). The consecutive-day branch was written as `elif days_since_last == 1 and today.weekday() != 6`, which excludes Sundays from being treated as a valid "listened yesterday, increment streak" case. So any time a user's second consecutive day of listening happened to be a Sunday, the code fell into the `else` branch and reset `listening_streak` to 1 instead of incrementing it, even though the user had listened on consecutive days.

Fix and side-effect check — Removed the extraneous `and today.weekday() != 6` condition so the branch is simply `elif days_since_last == 1:`, matching the documented rule ("if the user listened yesterday: streak increments by 1") with no day-of-week exception. After the fix I re-ran four scenarios directly against `update_listening_streak`: Saturday→Sunday (now correctly 5→6), Sunday→Monday (6→7, confirming the other side of the boundary also works), a normal weekday pair Tuesday→Wednesday (2→3, unaffected), and a multi-day gap (correctly resets to 1, so the reset logic for real gaps still works). I also ran the full `pytest tests/` suite — the streak tests pass, and the only failures are the two pre-existing playlist tests tied to Issue #5 (unrelated to this fix).

BUG 2:
Issue number and title — #2 — Friends Listening Now shows people from yesterday

How you reproduced it — I read `get_friends_listening_now()` in `feed_service.py` and, at first, could not find a straightforward off-by-one — the code filters `ListeningEvent.listened_at >= cutoff` where `cutoff = now - timedelta(hours=24)`. I wrote a script (Python shell, app context) to test the boundary directly: I created a friend with only a single listening event and no recent one, then set that event's `listened_at` to 25 hours ago, 23h59m ago, and 30 hours ago in separate runs, and called `get_friends_listening_now()` for their friend. In every case the rolling 24-hour filter behaved exactly as a 24-hour window should — nothing was excluded/included incorrectly relative to that window. That told me the code wasn't miscounting hours; the window itself (rolling 24h) is the wrong definition for "Listening Now." A friend who listened 18 hours ago is still inside a 24-hour rolling window, but that listen happened "yesterday" by calendar date — which matches the exact complaint in the issue title, confirmed by constructing a friend whose only event was 1 minute before midnight UTC: under the old rolling-window logic, that friend still showed up as "listening now" up to 24 hours later, well into the next day.

How you found the root cause — Since a direct code-level defect (like a wrong comparison operator or an inverted condition) wasn't present, I compared what the function actually implements (`now - 24h` rolling window) against what "Listening Now" implies (current, same-day activity) and against the issue title itself, which specifically says "yesterday" — a calendar concept, not an hour count. The `RECENT_THRESHOLD = timedelta(hours=24)` constant was the tell: it encodes a sliding window rather than a day boundary, so anything from the trailing ~24 hours (which very often spans into "yesterday" depending on what time of day the request is made) qualifies as "now."

The root cause — `get_friends_listening_now()` used a rolling 24-hour cutoff (`datetime.now(timezone.utc) - timedelta(hours=24)`) to decide what counts as "recent" activity, instead of a calendar-day boundary. This means a friend who listened at any point in the last 24 hours — including late in the previous calendar day — is shown as currently listening, even though from the user's perspective that activity happened "yesterday." The bug isn't a miscount; it's the wrong boundary definition (rolling window vs. calendar day).

Fix and side-effect check — Changed the cutoff calculation to `datetime.combine(today, datetime.min.time(), tzinfo=timezone.utc)` (midnight UTC of the current calendar day) instead of `now - timedelta(hours=24)`, and removed the now-unused `RECENT_THRESHOLD` constant. This means only listens that occurred on today's UTC date are included, regardless of how many hours ago that was. I verified both sides of the new boundary: a listen at 11:59pm UTC "yesterday" is now correctly excluded, and a listen at 12:01am UTC "today" is correctly included, even though both are within an hour of each other. I also confirmed events from minutes ago (the normal case) still show up correctly, and re-ran `pytest tests/` — no new failures; the only failures are the two pre-existing playlist tests tied to Issue #5.

BUG 3:
Issue number and title — #3 — The same song keeps showing up twice in search

How you reproduced it — `search_service.search_songs()` joins `Song` to the `song_tags` association table with an `outerjoin` so it can eventually surface tags, but never groups or deduplicates by song. `seed_data.py` deliberately creates several songs with 3+ tags "to expose Issue #3." I called `search_songs("Crown Heights")` (a song seeded with 3 tags: rap, hip-hop, boom bap) directly and via the `/songs/search` HTTP endpoint — and got exactly 1 result, not 3. The existing pytest test for this (`test_search_no_duplicates_multi_tag_song`, comment: "Should be 1, bug causes it to be 3") also passed. At first this looked like the bug wasn't present at all.

How you found the root cause — I didn't trust that the query itself was actually clean just because the ORM result looked deduplicated, so I dropped to raw SQL against `instance/mixtape.db` for the exact join/filter `search_songs` builds: `SELECT song.id FROM song LEFT OUTER JOIN song_tags ON song.id = song_tags.song_id WHERE title LIKE ...`. That returned the same song's id 3 times — one row per tag — proving the join fans out rows exactly as expected without a DISTINCT. So why didn't `search_songs()` show it? `search_service.py` builds the query with `db.session.query(Song)`, SQLAlchemy's legacy `Query` API, which silently auto-deduplicates full-entity results as a backwards-compatibility shim — it collapses the 3 duplicate rows back into 1 Song object before `.all()` returns. I confirmed this by running the identical join/filter through the modern `session.execute(select(Song)...)` API instead, which does not carry that legacy dedup behavior: it returned all 3 duplicate rows. That's the moment I was confident I'd found the actual defect — the query itself is wrong (missing `.distinct()`); it merely isn't currently visibly broken through this one specific ORM call style in this SQLAlchemy version.

The root cause — `search_songs()` performs `db.session.query(Song).outerjoin(song_tags, ...)` without a `.distinct()` (or equivalent grouping). For any song with N tags, the join produces N duplicate rows for that song at the SQL level. Whether that duplication is visible to a caller is an accident of which SQLAlchemy API is used to run the query (legacy `Query.all()` happens to auto-uniquify ORM entity results in the installed version; the modern `select()`/`session.execute()` API does not) — so the correctness of the result silently depends on an ORM implementation detail rather than the query being correct by construction. That's the "conditional" part: the duplicate only appears for songs with 2+ tags, and whether it's masked or not depends on incidental ORM behavior, not anything the code deliberately guards against.

Fix and side-effect check — Added `.distinct()` to the query chain in `search_songs()`, right after the filter. I verified the fix at the level that actually exposed the bug: re-running the identical join/filter through `session.execute(select(Song)...` with `.distinct()` added now returns exactly 1 row for the 3-tag song, matching the intended behavior regardless of which SQLAlchemy query API is used. I also re-checked the 1-tag and 0-tag songs return exactly once (no over-collapsing), confirmed a basic search still returns matches, and ran `pytest tests/` — the same 5 search tests pass, no new failures beyond the two pre-existing Issue #5 playlist failures. I did not commit the scratch reproduction scripts I used to compare the legacy `Query` vs. modern `select()` behavior — they're throwaway diagnostics, not part of the fix.

BUG 4:
Issue number and title — #4 — I got notified when a friend added my song to a playlist but not when they rated it

How you reproduced it — In a Python shell (app context), I picked an existing song, looked up its sharer, and called `get_notifications(sharer_id)` to record the count before. I then called `rate_song(rater_id, song_id, 5)` where `rater_id` was a different user than the sharer, and called `get_notifications(sharer_id)` again. The notification count was identical before and after — no notification was created for the sharer even though a friend rated their song.

How you found the root cause — README pointed to `notification_service.py` for this issue, and the file already contains a working example of the pattern (`add_to_playlist`), which the hint told me to compare line-by-line against the broken one. Reading `add_to_playlist()`: after writing the playlist-song relationship, it checks `if song.shared_by != added_by_user_id` and calls `create_notification(user_id=song.shared_by, notification_type="song_added_to_playlist", body=...)` to alert the original sharer. Reading `rate_song()` immediately below it: it validates the score, saves or updates a `Rating` row, commits, and returns — there is no call to `create_notification` anywhere in the function, and no reference to `song.shared_by` at all. That absence, next to the working pattern one function above it, was the moment I was confident this was the actual defect and not just a symptom — `rate_song()` never attempts to notify anyone, so there's no faulty condition to trace further, just a missing step.

The root cause — `rate_song()` in `notification_service.py` saves a `Rating` record but never calls `create_notification()` to alert the song's original sharer, unlike the structurally identical `add_to_playlist()`, which does call `create_notification()` for the same kind of "someone interacted with your shared song" event. This isn't a typo or off-by-one — the notification side effect for the rating flow was simply never implemented, even though the exact pattern to follow already existed in the same file for the playlist-add flow.

Fix and side-effect check — After the existing `db.session.commit()` in `rate_song()`, added the same guarded notification call used in `add_to_playlist()`: `if song.shared_by != user_id: create_notification(user_id=song.shared_by, notification_type="song_rated", body=f"{rater.username} rated your song '{song.title}' {score}/5.")`. The `song` and `rater` objects were already fetched earlier in the function, so no extra queries were needed. I verified three cases directly against `rate_song()`: (1) a friend rates another user's song — notification count increases by 1 for the sharer; (2) the sharer rates their own song — no notification is created for themselves, matching the same self-notification guard `add_to_playlist` already uses; (3) the same rater re-rates the same song (hitting the "update existing rating" branch instead of "insert new rating") — still produces a notification each time, so re-rating correctly keeps notifying the sharer. I also ran `pytest tests/` — all previously-passing tests still pass, with no new failures beyond the two pre-existing Issue #5 playlist failures.

BUG 5:
Issue number and title — #5 — The last song in a playlist never shows up

How you reproduced it — In a Python shell, I queried the `playlist_entries` table directly for a playlist's rows (7 entries, ordered by `position`) and compared that against `get_playlist_songs(playlist.id)`, which returned only 6. I looked up each `song_id` from `playlist_entries` to get its title, and confirmed the missing one was "Free Throws" — the song at the highest `position`, i.e. the last song in the playlist.

How you found the root cause — README pointed to `playlist_service.py` for this issue. Reading `get_playlist_songs()`, the query itself correctly joins `playlist_entries` and orders by `position` ascending — I confirmed this independently by comparing the query's song order against my direct `playlist_entries` query, and they matched. The only transformation between the correctly-ordered query result and the returned value is the final line: `return [song.to_dict() for song in songs[:-1]]`. Since the query was already proven correct, and `[:-1]` slices off the last element of any list regardless of its contents, this was confirmed as the exact cause rather than just a suspicious-looking line.

The root cause — `get_playlist_songs()` queries and orders songs correctly, but the return statement applies `songs[:-1]`, which unconditionally drops the last item of the list before returning it. For a playlist with N songs, only N-1 are ever returned — the song in the last position is silently omitted every time, regardless of playlist length.

Fix and side-effect check — Changed `songs[:-1]` to `songs` so the full ordered list is returned. Verified on both sides of the boundary: an existing 7-song playlist now correctly returns 7 songs including "Free Throws" (previously missing), and a newly created 1-song playlist now correctly returns that 1 song instead of an empty list (previously the most severe case of this bug, where a single-song playlist appeared completely empty). Ran `pytest tests/` — all 13 tests pass, including `test_playlist_returns_all_songs` and `test_playlist_returns_songs_in_order`, which were specifically written to catch this bug and were failing before the fix.

---

## Commit History

Screenshot of `git log --oneline` on the `bugfix/mixtape` branch, showing one commit per bug fix:

![git log --oneline on bugfix/mixtape](./git%20log%20oneline.png)