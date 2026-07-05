# Mixtape — Submission Doc
## AI Usage

I used Claude as an AI assistant throughout this project in the following ways:

**Codebase orientation:**
I shared the contents of each service file with Claude and asked it to
summarize what each module was responsible for and what its main functions
did. This helped me build a mental model of the app quickly without
reading every line myself. I then verified my understanding by tracing
the call chains manually.

**Data flow tracing:**
I asked Claude to trace how a user rating a song flows through the
codebase — from the route to the service — and to identify what each
step returns. This confirmed my understanding of how routes delegate
to services and helped me write the data flow section of my codebase map.

**Bug investigation:**
For each bug, I read the suspicious code myself first, then asked Claude
to explain specific functions or conditions I didn't fully understand.
For example:
- Bug #1: I asked Claude to explain what Python's `weekday()` returns
  for each day of the week, which confirmed my hypothesis about the
  Sunday boundary condition.
- Bug #4: I asked Claude to compare the structure of `add_to_playlist()`
  and `rate_song()` to identify the structural difference between the
  working and broken notification paths.
- Bug #5: I identified the `songs[:-1]` slice myself and asked Claude
  to confirm what it returns in Python.

**What I verified myself:**
I reproduced every bug manually before touching any code, and verified
every fix by calling the relevant API endpoint and checking the response.
Claude's explanations pointed me in the right direction, but I confirmed
each root cause by reading the code myself.
## Codebase Map

### Main Files and Their Responsibilities

**`models.py`**
Defines 7 database entities:
- `User`: Stores username, email, listening streak count (`listening_streak`),
  and last listened timestamp (`last_listened_at`)
- `Song`: Stores title, artist, album, genre, sharer, and share note
- `Tag`: Song tags (e.g. genre keywords), linked to Song via the
  `song_tags` association table in a many-to-many relationship
- `ListeningEvent`: Records who listened to which song and when
- `Rating`: Stores a user's 1–5 score for a song; a unique constraint
  ensures each user can only rate each song once
- `Playlist`: Stores name, creator, and whether the playlist is collaborative
- `Notification`: Stores notification type (e.g. `song_rated`),
  body text, and read status

Association tables:
- `friendships`: Symmetric many-to-many relationship between Users
- `song_tags`: Many-to-many between Song and Tag
- `playlist_entries`: Many-to-many between Song and Playlist, with extra
  columns `position` (explicit ordering) and `added_by` (who added the song) —
  song order in a playlist is managed explicitly, not by insertion order

---

**`routes/songs.py`**
Handles HTTP requests for sharing, searching, and rating songs.
Contains no business logic — delegates immediately to
`search_service` and `notification_service`.

**`routes/playlists.py`**
Handles playlist creation, retrieval, and song management requests.
Delegates to `playlist_service` and `notification_service`.

**`routes/users.py`**
Handles user profile, listening streak, and notification requests.
Delegates to `streak_service` and `notification_service`.

**`routes/feed.py`**
Handles "Friends Listening Now" and activity feed requests.
Delegates to `feed_service`.

---

**`services/streak_service.py`**
Manages user listening streaks. Core logic lives in
`update_listening_streak()`:
- Listened today already → no change
- Listened yesterday → streak +1
- More than one day gap → reset to 1

**`services/feed_service.py`**
Provides two feed types:
- `get_friends_listening_now()`: Returns friends with listening events
  in the past 24 hours; only the most recent song per friend is shown
- `get_activity_feed()`: Returns the most recent N listening events
  across all friends, with no time filter

**`services/search_service.py`**
Performs case-insensitive fuzzy search on song title and artist
using `ilike`. Uses `outerjoin` on `song_tags` to include tag data.

**`services/notification_service.py`**
Creates and retrieves notifications. Also contains two business operations:
- `add_to_playlist()`: Adds a song to a playlist and notifies the
  song's original sharer
- `rate_song()`: Creates or updates a Rating record

**`services/playlist_service.py`**
Handles playlist creation and retrieval.
`get_playlist_songs()` JOINs the `playlist_entries` table and returns
songs ordered ascending by `position`.

---

### Data Flow: A User Rates a Song
```
User sends POST /songs/<song_id>/rate

↓

routes/songs.py

Parses user_id and score from request body

↓

notification_service.rate_song(user_id, song_id, score)

Validates score is between 1 and 5
Looks up Song — raises ValueError if not found
Looks up User — raises ValueError if not found
Checks for existing Rating record

→ exists: updates score field

→ not found: creates new Rating and adds to session
Commits to database

↓

Returns Rating object

routes/songs.py formats it as JSON and returns to client
```
Note: `rate_song()` never calls `create_notification()` after saving
the rating — the song's original sharer receives no notification
that their song was rated.

---

### Patterns Observed

**Strict Route → Service layering**
Every route function parses the request, then immediately hands off
to a service function. Routes contain no business logic; services
contain no HTTP handling. This means: if a feature is broken,
the root cause will be in `services/`, not in `routes/`.

**Services own their own database transactions**
Each service function calls `db.session.commit()` itself after
completing its work. Callers do not need to commit.

**Consistent validation pattern**
Every service function validates that key objects exist (User, Song,
Playlist) before proceeding, raising `ValueError` if not found.

## Bug #5 — The last song in a playlist never shows up

**Issue:** #5 — The last song in a playlist never shows up

**How I reproduced it:**
Called `GET /playlists/0d9cabf3-1050-47be-9f4f-e979bfbb4d7c/songs` on a
playlist with 7 songs in the database. The response returned only 6 songs.
The last song (highest position) was missing from every playlist tested.

**How I found the root cause:**
Traced the request from `routes/playlists.py` to
`playlist_service.get_playlist_songs()`. The function queries songs ordered
by position correctly, but the final return statement immediately caught
my attention.

**Root cause:**
In `playlist_service.get_playlist_songs()`, the return statement used
`songs[:-1]` instead of `songs`. In Python, `songs[:-1]` returns all
elements except the last one. This meant the song with the highest
position value — always the last song in the ordered list — was silently
dropped from every response, regardless of playlist size.

**Fix and side-effect check:**
Changed `songs[:-1]` to `songs` in the return statement of
`get_playlist_songs()`. Verified the fix by calling the same

## Bug #4 — No notification when a song is rated

**Issue:** #4 — I got notified when a friend added my song to a playlist
but not when they rated it

**How I reproduced it:**
Sent `POST /songs/<song_id>/rate` as user darius rating nova's song
"Midnight Drive" with a score of 5. Then called
`GET /users/<nova_id>/notifications` and confirmed nova received no
notification about the rating — only a pre-existing playlist notification
was present.

**How I found the root cause:**
Traced the request from `routes/songs.py` to
`notification_service.rate_song()`. Compared its structure to
`add_to_playlist()`, which handles the working notification.
`add_to_playlist()` calls `create_notification()` after committing to
the database. `rate_song()` commits and returns immediately with no
equivalent call.

**Root cause:**
`rate_song()` in `notification_service.py` saves the rating and commits
to the database, but never calls `create_notification()`. The
`add_to_playlist()` function follows the correct pattern — after
committing, it checks whether the adder is the original sharer and
creates a notification if not. `rate_song()` was missing this entire
block, so the song's original sharer never received a `song_rated`
notification regardless of who submitted the rating.

**Fix and side-effect check:**
Added a `create_notification()` call after

## Bug #1 — Listening streak resets on Sundays

**Issue:** #1 — My listening streak keeps resetting

**How I reproduced it:**
Set nova's `last_listened_at` to yesterday (Saturday) with a streak of 7
via a direct database update. Then sent `POST /songs/<song_id>/listen`
with nova's user_id on a Sunday. Expected the streak to increment to 8,
but it reset to 1 instead.

**How I found the root cause:**
Traced the request from `routes/users.py` to
`streak_service.update_listening_streak()`. The function has three
branches: same day (no change), yesterday (increment), and anything else
(reset). The increment branch had an extra condition:
`days_since_last == 1 and today.weekday() != 6`.
The `weekday() != 6` check immediately stood out as unrelated to
streak logic.

**Root cause:**
Python's `datetime.weekday()` returns 6 for Sunday. The streak increment
condition included `today.weekday() != 6`, which means "today is not
Sunday." This caused the increment branch to be skipped entirely on
Sundays — even when the user had listened the day before. The code would
fall through to the `else` branch and reset the streak to 1. There is no
streak rule that involves the day of the week — the only condition for
incrementing should be that the user listened yesterday.

**Fix and side-effect check:**
Removed `and today.weekday() != 6` from the elif condition in
`update_listening_streak()`, leaving only `days_since_last == 1`.
Verified the fix by setting nova's last_listened_at to yesterday and
confirming the streak incremented from 7 to 8 on a Sunday.
Checked `record_listening_event()` and `get_streak()` — neither is
affected by this change.