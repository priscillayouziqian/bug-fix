# Mixtape — Submission Doc

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

**How I reproduced it:**
Called `GET /playlists/0d9cabf3-1050-47be-9f4f-e979bfbb4d7c/songs` on a
playlist with 7 songs in the database. The response returned only 6 songs.
The last song (highest position) was missing from every playlist tested.

## Bug #4 — No notification when a song is rated

**How I reproduced it:**
Sent `POST /songs/<song_id>/rate` as user darius (cbdc0790) rating nova's
song "Midnight Drive" with a score of 5. Then called
`GET /users/<nova_id>/notifications` and confirmed nova received no
notification about the rating — only a pre-existing playlist notification
was present. The `song_rated` notification type never appears.

## Bug #1 — Listening streak resets on Sundays

**How I reproduced it:**
Set nova's `last_listened_at` to yesterday (Saturday) with a streak of 7
via a direct database update. Then sent `POST /songs/<song_id>/listen`
with nova's user_id on a Sunday. Expected the streak to increment to 8,
but it reset to 1 instead. The bug only triggers when today is Sunday
(weekday() == 6).