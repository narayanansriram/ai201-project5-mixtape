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
    playlist_service - contains methods
                - create_playlist - creates a playlist
                - get_playlist_songs - gets playlist songs
                - get_playlist - gets playlist metadata without songs
                - get_user_playlists - gets playlists created by a user
    search_service - contains methods
                - search_songs - searches for songs by title or artist
                - get_song - fetches song by id
    streak_service - contains methods
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
- get request to streak, user id provided GET /users/<user_id>/streak -> calls get_streak in streak_service -> queries db to get user's listening streak using user_id
- create playlist where in post method is used POST /playlists/ -> calls create_playlist in playlist_service -> creates a Playlist entry in db, returns a dict -> dict jsonified to caller
- POST /playlists/<playlist_id>/songs → add_to_playlist() → creates the playlist entry → calls create_notification() → {adder.username} added your song '{song.title}' to the playlist '{playlist.name}'.

Pattern
Router catches requests like GET, POST. Services contain methods that perform database operations -- all called from router methods. Some service functions also trigger side effects like notifications by calling other service functions directly like add_to_playlist calling create_notification


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