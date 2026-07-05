# Mixtape Bug Hunt — Submission

---

## AI Usage

During this project, I used AI tools as a support for understanding and navigating the codebase rather than for directly guessing or blindly fixing bugs.

I primarily used AI to:

- Help interpret unfamiliar Flask and SQLAlchemy code patterns in the services layer
- Trace request flow from routes → services → database models
- Explain what specific functions were doing step-by-step when the logic was not immediately clear
- Clarify debugging strategy for reproducing API bugs using `curl` instead of the frontend
- Help identify likely locations of bugs after I had already narrowed down the relevant files

For debugging, I always verified AI suggestions by:

- Reading the actual function implementation in the codebase
- Running `curl` commands to reproduce the behavior myself
- Checking database state using SQLite queries when needed

In some cases, AI helped point me toward likely causes (such as slicing errors or missing joins), but I always confirmed the issue manually before applying any fix. This ensured that changes matched the actual application logic rather than assumptions.

Overall, AI was used as a guidance tool for understanding and structuring debugging steps, while all final decisions were validated through direct inspection and testing.

---

## Codebase Map

The Mixtape application is structured as a Flask backend with clear separation between routes, services, and database models.

### Core Structure

- `app.py`  
  Flask application factory and SQLAlchemy database setup.

- `models.py`  
  Defines database models including `User`, `Song`, `Playlist`, `ListeningEvent`, `Rating`, and `Notification`. Also includes association tables such as `song_tags`, `playlist_entries`, and `friendships`.

- `routes/`  
  Handles HTTP endpoints and request/response formatting:
  - `songs.py` → song search, rating, and listening events
  - `playlists.py` → playlist creation and song retrieval
  - `feed.py` → listening activity feeds
  - `users.py` → user profiles, notifications, streaks

- `services/`  
  Contains business logic:
  - `search_service.py` → song search functionality
  - `playlist_service.py` → playlist creation and retrieval
  - `feed_service.py` → friends listening now + activity feed
  - `notification_service.py` → notifications and rating logic
  - `streak_service.py` → listening streak tracking

### Data Flow Example (Song Rating)

1. `POST /songs/<song_id>/rate` → `routes/songs.py`
2. Route validates input (`user_id`, `score`)
3. Calls `rate_song()` in `services/notification_service.py`
4. Rating is saved/updated in `Rating` table
5. (After fix) Notification is created for the original song uploader

### Pattern Observed

- Routes are thin controllers that only handle HTTP input/output
- Services contain all business logic and database interaction
- Models define schema and relationships only
- Clear separation of concerns across layers

---

## Milestone 2 – Bug Reproduction

---

### Issue #5 – The last song in a playlist never shows up

**How I reproduced it**

```bash
curl http://127.0.0.1:5000/playlists/49181625-0626-4ea2-85d7-0d90721ad8cc/songs
````

**Observed behavior**

The endpoint returned **6 songs instead of 7**, even though the seed data clearly inserts 7 songs into the playlist.

---

### Issue #4 – Missing notification when a song is rated

**How I reproduced it**

```bash
curl -X POST http://127.0.0.1:5000/songs/4f3677e9-3912-446a-88ba-f623c1d68185/rate \
-H "Content-Type: application/json" \
-d '{"user_id":"05c7417c-4ce4-4a86-9146-5a7f8fcc0118","score":5}'
```

**Observed behavior**

The rating was successfully created, but no notification was generated for the original uploader.

---

### Issue #2 – Friends Listening Now shows empty feed

**How I reproduced it**

```bash
curl http://127.0.0.1:5000/feed/43198414-a259-438c-98bf-9e21af1edba2/listening-now
```

**Observed behavior**

The endpoint returned `count: 0`. I was unable to reproduce the originally described bug with the current seeded dataset, but I confirmed the endpoint response and proceeded with investigation.

---

## Milestone 3 – Root Cause Analysis

---

### Issue #3 – Search does not return songs matching tags

**How I reproduced it**

```bash
curl "http://127.0.0.1:5000/songs/search?q=rap"
```

**How I found the root cause**

I traced `search_songs()` in `services/search_service.py` and inspected the query logic. I then checked how tags are stored in the database and confirmed they exist in a separate `Tag` table connected via `song_tags`.

**Root cause**

The search query only filters `Song.title` and `Song.artist`. It never joins or checks `Tag.name`, so tag-based searches could not return results.

**Fix and side-effect check**

I added a join through `song_tags` and included `Tag.name.ilike()` in the filter. I verified:

* tag searches now return correct results
* title/artist search still works correctly

---

### Issue #4 – Missing notification when a song is rated

**How I reproduced it**

```bash
curl -X POST http://127.0.0.1:5000/songs/4f3677e9-3912-446a-88ba-f623c1d68185/rate \
-H "Content-Type: application/json" \
-d '{"user_id":"05c7417c-4ce4-4a86-9146-5a7f8fcc0118","score":5}'
```

**How I found the root cause**

I traced the flow from `routes/songs.py → rate_song()` in `services/notification_service.py` and compared it with `add_to_playlist()`, which correctly triggers notifications.

**Root cause**

`rate_song()` never called `create_notification()`. Unlike playlist additions, rating a song had no notification logic.

**Fix and side-effect check**

I added notification logic inside `rate_song()` to notify the original uploader when someone rates their song. I confirmed:

* ratings still work correctly
* notifications only trigger when appropriate

---

### Issue #5 – The last song in a playlist never shows up

**How I reproduced it**

```bash
curl http://127.0.0.1:5000/playlists/49181625-0626-4ea2-85d7-0d90721ad8cc/songs
```

**Observed behavior**

The endpoint returned **6 songs instead of 7**, even though the seed data clearly inserts 7 songs into the playlist.

**How I found the root cause**

I traced `get_playlist_songs()` in `services/playlist_service.py` and inspected the return statement.

**Root cause**

The function returned `songs[:-1]`, which removes the last song from the list.

**Fix and side-effect check**

I replaced `songs[:-1]` with `songs`. I verified:

* all songs now appear
* ordering remains unchanged

## Git Log Screenshot

The required git log screenshot is included in this repository:

- `commit.png`
