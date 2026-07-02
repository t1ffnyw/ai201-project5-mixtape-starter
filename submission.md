# Submission

## Summary

Mixtape is a Flask REST API for a social music app. Users share songs, build collaborative playlists, rate tracks, track daily listening streaks, and see what friends are listening to. The app uses SQLAlchemy with SQLite, follows a routes → services → models layout, and ships with pytest tests and seed data. This project is a bug-hunt exercise: business logic lives in `services/`, and endpoints delegate to those modules.

---

## AI Usage

1. Codebase Map
I used AI to understand the codebase structure and the overall functions of each file. I asked it to give me a summary of the codebase, each file and its purpose. This helped me understand the data flow and identify organizational patterns. 

2. Reproducing Bugs
I used AI to help reproduce bugs. For each bug, I asked AI how can this bug be reproduced and what are the edge cases that cause this bug to appear. This helped me verify that the fixes worked beyond the given pytests. Furthermore, I used AI to explain specific functions and blocks of code (like how outerjoin works) in order to identify the root cause for the bugs. After the fix, I double checked with AI and asked it if the fix caused any side effects or unintentionally broke any of the functionality. 

---

## Codebase Map

### Root files

| File | Purpose |
|------|---------|
| `app.py` | Application factory (`create_app`), Flask/SQLAlchemy setup, blueprint registration, and `db.create_all()` on startup |
| `models.py` | SQLAlchemy models (`User`, `Song`, `Tag`, `Playlist`, `Rating`, `ListeningEvent`, `Notification`) and join tables for friendships, tags, and ordered playlist entries |
| `seed_data.py` | Drops/recreates the DB and loads test users, songs, playlists, listening history, and sample notifications |
| `requirements.txt` | Dependencies: Flask, Flask-SQLAlchemy, SQLAlchemy, python-dotenv, pytest |

### `routes/` — HTTP layer

Thin controllers that parse requests, call services, and return JSON.

| File | Prefix | Endpoints |
|------|--------|-----------|
| `songs.py` | `/songs` | Search, song detail, rate a song, record a listen |
| `playlists.py` | `/playlists` | Create playlist, get playlist, list songs, add a song |
| `users.py` | `/users` | User profile, streak, notifications, mark notification read |
| `feed.py` | `/feed` | Friends listening now, activity feed |

### `services/` — Business logic

Where feature behavior is implemented (and where the known bugs live).

| File | Responsibility |
|------|----------------|
| `streak_service.py` | Records listening events and updates consecutive-day streaks |
| `feed_service.py` | Builds “friends listening now” and activity feeds from recent events |
| `search_service.py` | Searches songs by title/artist and returns results with tags |
| `notification_service.py` | Creates notifications, handles playlist-add side effects, ratings, and read state |
| `playlist_service.py` | Creates playlists and retrieves ordered song lists |

### `tests/`

| File | Covers |
|------|--------|
| `test_streaks.py` | Listening streak logic |
| `test_search.py` | Song search behavior |
| `test_playlists.py` | Playlist song retrieval |

---

## Data Flow: Adding a Song to a Playlist → Notification

When a friend adds someone else's shared song to a playlist, the original sharer gets notified.

1. **Request** — `POST /playlists/<playlist_id>/songs` with JSON `{ "song_id", "added_by" }`
2. **Route** — `routes/playlists.py` validates input and calls `notification_service.add_to_playlist()`
3. **Service** — `add_to_playlist()` in `notification_service.py`:
   - Loads the `Song`, `User` (adder), and `Playlist` from the DB
   - Appends the song to the playlist if not already present and commits
   - If `song.shared_by != added_by_user_id`, calls `create_notification()` with type `song_added_to_playlist`
4. **Persistence** — `create_notification()` inserts a `Notification` row and commits
5. **Retrieval** — The sharer can fetch it via `GET /users/<user_id>/notifications`, which routes to `get_notifications()` and returns ordered notification dicts

```
Client → playlists route → notification_service.add_to_playlist()
                              → create_notification() → Notification model → DB
Client → users route → notification_service.get_notifications() → JSON response
```

---

## Organizational Patterns

- **App factory** — `create_app()` in `app.py` wires config, DB, and blueprints; tests and `seed_data.py` both call it to get a configured app context.
- **Blueprint per domain** — Routes are split by feature area (`songs`, `playlists`, `users`, `feed`) with URL prefixes set at registration time.
- **Thin routes, fat services** — Routes handle HTTP concerns (params, status codes, JSON); services own validation, queries, and side effects.
- **Models expose `to_dict()`** — API responses are built from model serialization methods rather than separate serializers.
- **Cross-service calls** — Some services import from others (e.g. `notification_service` touches playlist logic when adding songs), so tracing a bug often means following imports across the `services/` layer.
- **SQLite by default** — `DATABASE_URL` env var can override; otherwise the app uses `sqlite:///mixtape.db`.


---

## Bug Fixes

### Bug 1: Listening streak resets on Sunday (Issue #1)

**Service:** `streak_service.py`

**How I reproduced it:**

1. **Required state:** A user with an existing streak whose `last_listened_at` is **yesterday** (Saturday) and whose next listen happens **today** (Sunday). 

2. Ran the targeted pytest:
   ```bash
   pytest tests/test_streaks.py::test_streak_increments_on_sunday -v
   ```
   The test simulates this by calling `update_listening_streak()` twice:
   - Saturday 2024-06-15 → streak becomes `1`
   - Sunday 2024-06-16 → streak should become `2`, but stays `1`
Observed failure: `assert u.listening_streak == 2` fails with `assert 1 == 2`


**Root cause:** 
To find the root cause, I looked at `streak_service.py` and the `update_listening_streak()` function. In `update_listening_streak()`, the consecutive-day branch requires `today.weekday() != 6`, which excludes Sunday. Listening on Sunday after listening yesterday falls through to the `else` branch and resets the streak to `1`, even though only one day has passed.

**Fix and side-effect check:**

- **What I changed:** Removed `and today.weekday() != 6` from the consecutive-day branch in `update_listening_streak()`, so the condition is now simply `elif days_since_last == 1:`.
- **Why this fixes the root cause:** The streak rules only care about calendar-day gaps, not the day of the week. The Sunday check incorrectly sent Saturday→Sunday listens to the reset branch. With it gone, any listen exactly one day after the last one increments the streak — including Sunday.
- **What I checked afterward:**
  - `pytest tests/test_streaks.py -v` — all 5 tests pass, including `test_streak_increments_on_sunday` (the failing case) and `test_streak_resets_after_skipped_day` (confirms skipped days still reset).
  - Verified same-day listens still do not double-count (`test_streak_does_not_double_count_same_day`).
  - Verified new users still start at streak 1 and Mon→Tue consecutive increments still work.

---

### Bug 2: Duplicate songs in search (Issue #3)

**Service:** `search_service.py`

**How I reproduced it:**

1. **Required state:** At least one song with **multiple tags** in the `song_tags` join table. Seed data includes five such songs (e.g. **Crown Heights Anthem** with tags `rap`, `hip-hop`, `boom bap`). Songs with 0 or 1 tag do not hit this path.

2. Ran the targeted pytest:
   ```bash
   pytest tests/test_search.py::test_search_no_duplicates_multi_tag_song -v
   ```
   The test creates "Crown Heights Anthem" with 3 tags, searches for `"Crown Heights"`, and asserts the song appears exactly once.

3. **SQL-level reproduction** (shows the join multiplying rows regardless of ORM deduplication):
   ```sql
   SELECT s.title, st.tag_id
   FROM song s
   LEFT JOIN song_tags st ON s.id = st.song_id
   WHERE s.title LIKE '%Crown Heights%';
   ```
   This returns **3 rows** for one song — one per tag.

**Root cause:** 
To find the root cause I looked at `search_service.py` and the `search_songs()` function. `search_songs()` outer-joins `song_tags` to attach tag data, which attaches a song multiple times if it has more than one tag (shows up once per tag). 

**Fix and side-effect check:**

- **What I changed:** Removed the `.outerjoin(song_tags, ...)` from the `search_songs()` query. The query now filters directly on the `Song` table by `title` and `artist` only.
- **Why this fixes the root cause:** The join was multiplying result rows — one per tag association — even though tags were never used in the `WHERE` clause. Removing it eliminates row duplication at the SQL level. Tags still appear in results because `song.to_dict()` loads them via the `Song.tags` relationship on the model.
- **What I checked afterward:**
  - `pytest tests/test_search.py -v` — all 5 tests pass, including multi-tag, single-tag, and no-tag duplicate checks.
  - Confirmed search still matches by title and artist (`test_search_returns_matching_songs`).
  - Confirmed empty results for no-match queries (`test_search_returns_empty_for_no_match`).
  - Manually searched seed data for `Crown Heights` — returns 1 result with all 3 tags present in the `tags` array.
  - Confirmed `get_song()` (single-song lookup) is unchanged and unaffected.

---

### Bug 3: Last song missing from playlist (Issue #5)

**Service:** `playlist_service.py`

**How I reproduced it:**

1. **Required state:** A playlist with **at least one song** in `playlist_entries`. Seed data populates three playlists with 7 songs each (e.g. **Late Night Vibes**).

2. Ran the targeted pytest:
   ```bash
   pytest tests/test_playlists.py::test_playlist_returns_all_songs -v
   ```
   Creates a playlist with 5 songs (Track 1–5). **Observed failure:** `assert len(songs) == 5` fails with `4 == 5`; Track 5 is missing.

3. **API reproduction:**
   ```http
   GET /playlists/<playlist_id>/songs
   ```
   Use any seeded playlist ID (or create a playlist, add 3+ songs via `POST /playlists/<id>/songs`, then fetch). `count` in the response is always one less than the number of rows in `playlist_entries`.

**Root cause:** 
To find the root cause I looked at `playlist_service.py`. I first checked `create_playlist()` to see if the playlist was being created correctly and the song wasn't being dropped during creation. Then I looked at `get_playlist_songs()` which is responsible for returning the songs in the playlist. I noticed after correctly querying songs ordered by `position`, the return statement slices off the final element with `songs[:-1]`, dropping the last song every time. This was the root cause of the missing last song. 

**Fix and side-effect check:**

- **What I changed:** Changed `return [song.to_dict() for song in songs[:-1]]` to `return [song.to_dict() for song in songs]` in `get_playlist_songs()`.
- **Why this fixes the root cause:** The query already returned the full ordered list; `[:-1]` was an off-by-one slice that discarded the last element on every call. Returning the full list includes the song at the highest `position`.
- **What I checked afterward:**
  - `pytest tests/test_playlists.py -v` — all 3 tests pass, including `test_playlist_returns_all_songs` (5 of 5 returned) and `test_playlist_returns_songs_in_order` (Track 1 through Track 5 in correct order).
  - Verified empty playlists still return `[]` without error (`test_empty_playlist_returns_empty_list`).
  - Confirmed `create_playlist()` and `get_playlist()` (metadata-only) are unchanged — only song retrieval was affected.
  - Manually verified against seed data: "Late Night Vibes" returns 7 songs (was 6), with the last song ("Free Throws") now present.