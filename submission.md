# Mixtape — Bug Hunt Submission

## AI Usage

I did this entire project in conversation with Claude (Claude Code), so essentially all
of the reading, tracing, and hypothesis-testing below was AI-assisted. What follows is
specifically about *how* it was used — where it accelerated things legitimately, and
where its first answer was incomplete, over-confident, or flat wrong until checked
against actually running the code.

**Navigation / orientation phase.** Before opening any service file, I had it read
`app.py`, `models.py`, and every file in `routes/`/`services/` and summarize
responsibilities, then trace two concrete call chains end-to-end (rating a song →
notification path; adding a song to a playlist → notification path). This is the "file
summary" and "data flow trace" pattern — it's reliable because the answer is checkable
against the actual imports and function bodies right there in the file, not a guess
about behavior. The "pattern noticed" writeup (every route is a thin adapter; all logic
lives in `services/`) came out of comparing all four route files side by side, which
would have taken much longer to notice file-by-file.

**Where structural diffing worked well.** For Issue #4, instead of asking "what's wrong
with `rate_song`", I had it line up `rate_song()` against `add_to_playlist()` — two
functions in the same file handling the same shape of event (a friend interacts with
your shared song). That comparison is what surfaced the missing `create_notification()`
call directly, rather than pattern-matching on the word "notification" somewhere in the
code. This matches the "give AI two similar code paths, ask what's structurally
different" approach — much more reliable than asking it to find the bug cold.

**Where the first hypothesis was wrong and had to be checked by running code.** Two
notable cases:

1. *Issue #3 (search duplicates).* Reading `search_songs()` in isolation, the natural
   (and textbook-correct-sounding) diagnosis is "joining `Song` to `song_tags` without
   `.distinct()` causes row fan-out for multi-tag songs → duplicate results." That's
   exactly what I expected going in, and it's *true at the raw-SQL level* (verified 3
   raw rows for a 3-tag song). But actually running `search_songs()` and counting the
   returned list showed 1, not 3 — SQLAlchemy's legacy `session.query(Song).all()`
   deduplicates full-entity results via its identity map, something the "obvious" fan-out
   explanation doesn't account for. I only caught this by executing the real function
   and a raw Core query side by side and comparing counts; if I'd trusted the
   pattern-matched explanation and gone straight to writing a `.distinct()` fix, I'd have
   "fixed" a bug that wasn't actually live, and the existing `test_search.py` suite
   already passing should have been the tell I checked *before* concluding anything.

2. *Issue #2 (feed shows stale friends).* My first pass over `get_friends_listening_now()`
   suspected the naive-vs-timezone-aware `datetime` comparison (SQLite stores/returns
   naive datetimes; the `cutoff` bound parameter is timezone-aware) as the culprit —
   a real inconsistency I confirmed exists (`entry['listened_at']` round-trips with no
   `+00:00` suffix). But testing it against a friend (`nova`, as seen by `darius`) whose
   only event was legitimately recent didn't expose any incorrect filtering — the 24h
   cutoff worked correctly for the timestamps in the seeded data. I had to try a
   *different* user than my first pick (checked `aaliya`'s 34-hour-old event as seen by
   `kenji`) before concluding the tz mismatch isn't actually the live bug path; the more
   likely real issue is that `RECENT_THRESHOLD = timedelta(hours=24)` is just too
   generous for a feature called "listening now" — a 2-hour-old listen showing up as
   "now" reproduced immediately and needed no clock-mocking at all. I didn't fix this one
   (it wasn't in my chosen three), but I wouldn't have caught that my first theory was
   the wrong one without actually running both scenarios and comparing.

**Where I verified rather than trusted.** For the Sunday streak bug (Issue #1), I asked
for the difference between Python's `datetime.weekday()` and `isoweekday()` once I'd
already narrowed the suspicious line down myself — a good use, since I'd already read
the code and just needed the stdlib semantics confirmed (`weekday()` returns 6 for
Sunday, which is what the buggy `!= 6` check was keying off). I didn't ask "why is this
broken" cold; I asked a narrow factual question about a function I'd already identified
as suspicious, then verified the fix against constructed Saturday/Sunday datetimes myself
rather than taking "removing the clause should fix it" on faith.

**General discipline followed throughout:** every claimed root cause was checked by
actually running the relevant function (via a Python one-off script or the live Flask
app) before writing it down, and every fix was re-run through the full test suite plus at
least one boundary case not already covered by existing tests. The one bonus discovery
(the `POST /playlists/<id>/songs` 500 due to `position`/`added_by` not being set by the
ORM `secondary=` relationship) came from actually trying to reproduce Issue #5 through
the live endpoint, not from reading — it's the kind of thing that's easy to miss by
inspection since `seed_data.py` and the test fixtures both sidestep it by inserting into
`playlist_entries` directly.

## Codebase Map

### Main files

- **`app.py`** — Flask application factory (`create_app`). Initializes the shared `db`
  (Flask-SQLAlchemy) instance, reads `DATABASE_URL`/`SECRET_KEY` from env, registers the
  four blueprints (`songs`, `playlists`, `users`, `feed`), and calls `db.create_all()` on
  startup. No business logic lives here.

- **`models.py`** — All SQLAlchemy models plus three association tables:
  - `User` — has `listening_streak` + `last_listened_at` (for streaks), a self-referential
    many-to-many `friends` relationship via the `friendships` table (rows are inserted in
    both directions — friendship is stored symmetrically, not derived), and backrefs to
    `Song` (`shared_songs`), `Rating`, `ListeningEvent`, `Notification`, `Playlist`.
  - `Song` — has `shared_by` (the original sharer) and a many-to-many `tags` relationship
    via `song_tags`, loaded with `lazy="subquery"` (a separate query, not a SQL join).
  - `ListeningEvent` — one row per "user listened to song at time T". This is the
    source-of-truth event log that both streaks and feeds are derived from.
  - `Rating` — one row per (user, song), enforced by a `UniqueConstraint`. Score 1–5.
  - `Playlist` — many-to-many `songs` via `playlist_entries`, which (unlike `song_tags`)
    carries extra columns: `position` (explicit ordering, not insertion order),
    `added_by`, `added_at`.
  - `Notification` — generic `user_id` + `notification_type` + `body` + `read` flag. No
    polymorphism — every notification type is just a pre-rendered string in `body`.

- **`routes/`** — one blueprint per resource area. Every route does request parsing
  (pull fields out of `request.get_json()`/`request.args`), calls exactly one service
  function, and maps `ValueError` → a 4xx JSON response. No route contains business logic.
  - `songs.py` — search, get-by-id, rate, record-a-listen.
  - `playlists.py` — create, get metadata, list songs, add a song.
  - `users.py` — get profile, get streak, list/mark-read notifications.
  - `feed.py` — "listening now" and general activity feed.

- **`services/`** — where all the actual logic (and all five bugs) lives:
  - `streak_service.py` — `record_listening_event()` writes a `ListeningEvent` and calls
    `update_listening_streak()`, which compares `today` to the user's `last_listened_at`
    date and increments/resets `listening_streak`.
  - `feed_service.py` — `get_friends_listening_now()` (24h-windowed, deduped to one song
    per friend) and `get_activity_feed()` (last N events, no time filter at all).
  - `search_service.py` — `search_songs()` (title/artist `ILIKE`, joined to `song_tags`
    for tag data) and `get_song()`.
  - `notification_service.py` — `create_notification()` (generic writer),
    `add_to_playlist()` (adds a song to a playlist *and* notifies the sharer),
    `rate_song()` (writes/updates a `Rating` — notably does **not** notify anyone),
    plus `get_notifications()`/`mark_as_read()`.
  - `playlist_service.py` — `create_playlist()`, `get_playlist_songs()` (position-ordered
    song list), `get_playlist()`, `get_user_playlists()`.

- **`seed_data.py`** — Builds 5 users/friendships, 25 songs (deliberately split into
  0-tag / 1-tag / 3+-tag groups), 3 playlists, and both "recent" (last 30 min) and
  "older" (hours-to-days old) `ListeningEvent` rows. The comments in this file are a
  strong hint about which scenarios expose which bug (e.g. it explicitly calls out
  the 3+-tag songs as the ones that "expose Issue #3").

- **`tests/`** — `test_streaks.py`, `test_search.py`, `test_playlists.py` correspond to
  Issues #1, #3, #5 and already contain the exact assertion that should pass once each
  bug is fixed (e.g. `assert u.listening_streak == 2  # Should increment, not reset`).
  There is **no** test file for `feed_service.py` (Issue #2) or `notification_service.py`
  (Issue #4) — those two need to be verified manually/by writing new tests.

### Pattern noticed

Every route is a thin adapter: parse input → call one service function → catch
`ValueError` → `jsonify`. All validation ("does this user/song/playlist exist"),
all persistence, and all cross-entity side effects (e.g. "adding a song to a playlist
also creates a notification") live in `services/`. This means every bug in the tracker
is reachable by reading a single service function — you never have to chase logic
back into a route or model. It also means the fix for each issue is a service-layer
change, not a routing or schema change.

### Data flow traces

**1. A friend adds your shared song to a playlist → you get notified**

```
POST /playlists/<playlist_id>/songs   {song_id, added_by}
  routes/playlists.py: add_song()
    → services/notification_service.py: add_to_playlist(playlist_id, song_id, added_by)
        - loads Song, User (adder), Playlist — 404s (ValueError) if any is missing
        - if song not already in playlist.songs: appends it (via the playlist_entries
          association table) and commits
        - if song.shared_by != added_by:
              create_notification(user_id=song.shared_by,
                                   notification_type="song_added_to_playlist",
                                   body=f"{adder.username} added your song '...' to '...'")
  → 201 {"message": "Song added to playlist"}
```

The notification is only created for the **original sharer**, and only if they aren't
the one who did the adding. Later, that sharer's client calls
`GET /users/<id>/notifications` → `notification_service.get_notifications()`, which just
filters `Notification` by `user_id` (and optionally `read=False`) ordered by recency.

**2. Compare: a friend rates your shared song → (should, but doesn't) notify you**

```
POST /songs/<song_id>/rate   {user_id, score}
  routes/songs.py: rate()
    → services/notification_service.py: rate_song(user_id, song_id, score)
        - validates score is 1-5
        - loads Song, User(rater) — 404s if missing
        - upserts a Rating row (unique on user_id+song_id) and commits
        - returns the Rating
  → 201 {rating fields}
```

This is the same shape of side effect as playlist-adding (another user interacting with
your shared song), but `rate_song()` never calls `create_notification()`. This is Issue
#4 — the fix is almost certainly "add the same `if rater_id != song.shared_by:
create_notification(...)` block that `add_to_playlist()` already has."

---

## The Five Issues — Investigation Notes

I read all five issue titles/affected files before picking, and ran the existing
`pytest` suite plus a few manual scripts against the seeded DB to see which bugs
actually reproduce today (not just which lines look suspicious).

```
3 failed, 10 passed
FAILED tests/test_playlists.py::test_playlist_returns_all_songs        (5 songs → 4)
FAILED tests/test_playlists.py::test_playlist_returns_songs_in_order   (missing last track)
FAILED tests/test_streaks.py::test_streak_increments_on_sunday         (2 → 1)
```

| # | Title | File | Status | Root cause found while reading |
|---|-------|------|--------|---------------------------------|
| 1 | Streak keeps resetting | `streak_service.py` | **Confirmed** (test fails) | `update_listening_streak()` only increments on a consecutive day `and today.weekday() != 6` — so listening Sat→Sun always resets to 1 instead of incrementing, any week, every week. |
| 2 | Friends Listening Now shows people from yesterday | `feed_service.py` | **Reproduced manually** | `RECENT_THRESHOLD = timedelta(hours=24)`. A feature literally named "listening **now**" treats anything in the last 24 hours as current — seeding a friend's only listen ~2 hours ago still surfaces them as "now." The window is just too wide for what the feature promises; no test file exists for this service yet. |
| 3 | Same song shows up twice in search | `search_service.py` | **Did NOT reproduce** | `search_songs()` joins `Song` to `song_tags` without `.distinct()`, so the raw SQL fans out one row per tag (verified 3 raw rows for a 3-tag song). But because the code calls legacy `session.query(Song)...all()` (not the 2.0-style `select()`), SQLAlchemy's ORM identity map deduplicates full-entity results automatically — confirmed empirically (`len(rows) == 1`) and all 5 `test_search.py` cases pass as-is. Worth re-checking with fresh eyes before spending a fix slot here — the described bug may already be latent/dormant rather than live. |
| 4 | Notified for playlist-add but not for rating | `notification_service.py` | **Confirmed by inspection** | `add_to_playlist()` calls `create_notification()` when adder ≠ sharer; `rate_song()` has no equivalent call at all. No test file covers this — will need to add one. |
| 5 | Last song in a playlist never shows up | `playlist_service.py` | **Confirmed** (test fails) | `get_playlist_songs()` queries songs in correct position order, then returns `songs[:-1]` — an explicit off-by-one slice that drops the last element every time, even though the docstring says "returns all songs in the playlist." |

### Rough plan for which three to fix first

Picking **#1 (streak Sunday bug), #5 (playlist last-song slice), and #4 (missing
rating notification)** first:

- #1 and #5 already have failing tests pinpointing the exact expected behavior, so
  they're fast, low-risk, high-confidence fixes (single-line logic errors).
- #4 has a clear, obvious fix by mirroring the existing `add_to_playlist()` pattern,
  though it'll need a new test since none exists.
- #2 is next in line after those three — the fix is more of a product/behavior judgment
  call (what should "now" actually mean? shrink `RECENT_THRESHOLD`, or key off the
  most-recent event more strictly?) rather than a pure logic-error fix, so it needs a
  bit more thought.
- #3 is deliberately last/lowest priority: as investigated above, the described
  duplicate-row bug does not currently reproduce against this repo's SQLAlchemy usage
  (legacy `Query` auto-dedupes). Before touching it I'd want to confirm whether there's
  a code path that *does* still duplicate (e.g. if a future refactor moves to 2.0-style
  `select()`), or whether `.distinct()`/`.unique()` should be added defensively anyway
  even though today's tests already pass.

---

## Reproduction Log (before any fix code)

For each of the three chosen bugs, I triggered the actual reported behavior against a
running instance of the app (`flask run` + the seeded `mixtape.db`) before touching any
service code. Full commands are below; no fixes have been applied yet.

### Bonus find while reproducing: `POST /playlists/<id>/songs` currently 500s

Before I could reproduce Issue #5 through the live "add a song" endpoint, the endpoint
itself crashed:

```
sqlalchemy.exc.IntegrityError: (sqlite3.IntegrityError) NOT NULL constraint failed: playlist_entries.position
[SQL: INSERT INTO playlist_entries (playlist_id, song_id, added_at) VALUES (?, ?, ?)]
```

Root cause: `add_to_playlist()` in `notification_service.py` adds a song via the ORM
relationship (`playlist.songs.append(song)`), which only knows how to populate the two
foreign-key columns on the `playlist_entries` association table. It has no way to fill in
`position` or `added_by`, which are both `nullable=False` with no default — so every
live call to this route fails. `seed_data.py` and the `test_playlists.py` fixture both
avoid this by inserting into `playlist_entries` directly instead of going through the
ORM relationship, which is why it wasn't obvious from the tests alone.

This isn't one of the five tracked issues and I'm not fixing it as part of this pass,
but it's notable because Issue #4 is framed as "playlist-add notifications already work,
rating notifications don't" — and right now, via the real API, playlist-add doesn't
work either (it never reaches the notification code, since the crash happens first, on
the `db.session.commit()` right after `playlist.songs.append(song)`). Flagging for
after the three chosen fixes land.

## Root Cause Analyses (fixed bugs)

### Issue #5 — last song in a playlist never shows up

**How I reproduced it:** Since the real `POST /playlists/<id>/songs` endpoint crashes
(see the "bonus find" above), I inserted directly into the `playlist_entries` table —
the same technique `seed_data.py` already uses — to build a clean playlist with 4 songs
at positions 1–4, then hit the *real* `GET /playlists/<id>/songs` endpoint (the code
path Issue #5 is actually about) over HTTP:

```
POST /playlists/                                  → created playlist f8eecd5c-...
(direct insert into playlist_entries: 4 songs, position=1..4, all added_by=nova)
GET /playlists/f8eecd5c-.../songs
  → {"count": 3, "songs": [Midnight Drive, Still Waters, First Light]}
```

"Block Party" (position 4, the last song inserted) was missing from the response —
`count` was 3 instead of 4.

**How I found the root cause:** The route (`routes/playlists.py: get_songs()`) just
calls and returns `playlist_service.get_playlist_songs(playlist_id)`, so the whole
investigation is inside that one function in `services/playlist_service.py`. Reading it
top to bottom: it loads the `Playlist` (404s if missing), then runs a query joining
`Song` to `playlist_entries` filtered by `playlist_id` and ordered ascending by
`position` — that part is correct and matches the docstring ("Songs are returned in the
order they were added"). The very next line is where confidence landed:
`return [song.to_dict() for song in songs[:-1]]`. The query variable `songs` already
holds the fully correct, correctly ordered list; `[:-1]` then unconditionally slices off
the last element before returning. That one line is the entire bug — nothing upstream
of it (the query, the join, the ordering) is wrong.

**The root cause:** `get_playlist_songs()` builds the song list correctly but then
returns `songs[:-1]` instead of `songs`. Python's `list[:-1]` slice always drops the
final element of whatever list it's given, regardless of the list's length — so no
matter how many songs a playlist has, the last one (by position order) is silently
excluded from every response. The function's own docstring even says "this function
returns all songs in the playlist," directly contradicting the code beneath it.

**My fix and side-effect check:** Changed the return line from
`[song.to_dict() for song in songs[:-1]]` to `[song.to_dict() for song in songs]` —
a one-token change, since the query above it was already correct and needed no
modification. To check for side effects: I grepped the repo for other `[:-1]`/slicing
patterns on song lists (`playlist_service.py`, `notification_service.py`, `seed_data.py`)
and found this slice was not duplicated anywhere else. I re-ran the full test suite
(`pytest tests/`) — `test_playlists.py`'s three tests now all pass, and the other two
test files are unaffected (the only remaining failure at this point is the not-yet-fixed
Sunday streak test). I also specifically checked the boundary case not covered by the
existing tests — a playlist with exactly **one** song — since a bug in a "drop the
last item" slice is exactly the kind of thing that behaves differently at small sizes:
before the fix this would have returned 0 songs instead of 1; after the fix it correctly
returns the single song.

### Issue #4 — notified for playlist-add but not for rating

**How I reproduced it:** Used the seeded song "Midnight Drive" (shared by `nova`) and
had her friend `darius` rate it, checking `nova`'s notifications before and after via
the real HTTP endpoints:

```
GET  /users/<nova_id>/notifications                        → count: 1 (seeded
                                                                "song_added_to_playlist")
POST /songs/<midnight_drive_id>/rate {user_id: darius, score: 5}
                                                             → 201, rating created
GET  /users/<nova_id>/notifications                         → count: 1 (unchanged)
```

The rating is saved successfully (the POST returns 201 with the new `Rating` row), but
`nova`'s notification count doesn't move.

**How I found the root cause:** `routes/songs.py: rate()` calls
`notification_service.rate_song()`, so — despite the module being named
`notification_service.py` — that's where the logic lives. Reading `rate_song()` top to
bottom: it validates the score, loads `song` and `rater`, upserts a `Rating` row, commits,
and returns. There is no call to `create_notification()` anywhere in the function —
not a wrong condition, an absent one. Confidence came from diffing this function
structurally against `add_to_playlist()` directly above it in the same file: both
functions load the acting user and the target `Song`, both know `song.shared_by`, both
have a natural "the sharer should hear about this" moment — but only `add_to_playlist()`
ends with the `if song.shared_by != added_by_user_id: create_notification(...)` block.
`rate_song()` has every piece of data that block would need (`song.shared_by`,
`rater.username`, `song.title`) and simply never uses it to notify anyone.

**The root cause:** `rate_song()` never calls `create_notification()`. It's not a logic
error in an existing condition — the notification side effect for "someone interacted
with your shared song" was implemented for the playlist-add path and never added for the
rating path, even though both handlers have identical data available to build the same
kind of notification.

**My fix and side-effect check:** Added a `create_notification()` call at the end of
`rate_song()`, guarded by `song.shared_by != user_id` (mirroring `add_to_playlist()`'s
guard exactly, so a user rating their own shared song doesn't notify themselves), using
`notification_type="song_rated"`. Since `notification_service.py` had zero test coverage
before this, I added `tests/test_notifications.py` with two cases: rating a friend's
song creates exactly one `song_rated` notification for the sharer, and rating your own
song creates none. Side-effect check: ran the full suite (`pytest tests/`) — all
previously-passing tests still pass, including `test_search.py`/`test_playlists.py`
(untouched) and the pre-existing `rate_song` behavior (score validation, upsert-on-rerate
via the `existing` branch) which I didn't change. I also re-verified live against the
running app with a fresh rating: notification count went from 1 → 2 with the new
`song_rated` entry appearing, while the original seeded `song_added_to_playlist`
notification was left untouched — confirming I only added a new code path rather than
altering the existing one.

### Issue #1 — listening streak keeps resetting (Sunday boundary)

**How I reproduced it:** This bug only triggers when "today" is a Sunday and the user
listened on the immediately preceding day (Saturday) — a real calendar condition, not
something reachable on demand through the live API today (the actual server clock is
currently a Monday in UTC). Waiting for an actual Sunday isn't a reasonable way to
verify a date-boundary bug, so — the same way `test_streaks.py` already does — I called
the real production function, `update_listening_streak()` (the exact function
`record_listening_event()` invokes from the `/songs/<id>/listen` route), directly with
a constructed clock:

```python
u = User(username="repro_sunday_user", ...)   # fresh user, no prior listening history
saturday = datetime(2026, 7, 4, 20, 0, tzinfo=timezone.utc)   # weekday() == 5
sunday   = datetime(2026, 7, 5,  9, 0, tzinfo=timezone.utc)   # weekday() == 6

update_listening_streak(u, saturday)   # -> streak = 1
update_listening_streak(u, sunday)     # -> streak = 1   (expected: 2, consecutive day)
```

Listening on two consecutive calendar days should increment the streak (as it does for
every other day-of-week pair — confirmed by the existing Mon→Tue test already passing).
It doesn't when the second day is a Sunday: `update_listening_streak()` returned
`streak = 1` after the Sunday listen instead of `2`.

**How I found the root cause:** `routes/songs.py: listen()` calls
`streak_service.record_listening_event()`, which writes the `ListeningEvent` row and then
delegates the actual streak math to `update_listening_streak()` — that's the whole
call chain, so the investigation stayed inside `streak_service.py`. Reading the function
against its own docstring (lines 46–50: "if the user listened yesterday: streak
increments by 1... if more than one day has passed: streak resets to 1") — the documented
rule is purely about *how many calendar days elapsed*, with no mention of which weekday
it is. The moment of confidence was lining up `days_since_last = (today - last_date).days`
(correct — computes elapsed calendar days) against the very next line,
`elif days_since_last == 1 and today.weekday() != 6:`. The `and today.weekday() != 6`
clause is not described anywhere in the docstring and has nothing to do with "how many
days elapsed" — it's an extra condition bolted onto the correct one, and it's the only
thing standing between the correctly-computed `days_since_last == 1` and the increment.

**The root cause:** Python's `datetime.weekday()` returns `6` for Sunday. The condition
`days_since_last == 1 and today.weekday() != 6` means: increment the streak on a
consecutive day, *unless that day happens to be a Sunday* — in which case fall through to
the `else` branch and reset the streak to 1, even though exactly one day elapsed since the
last listen. This has nothing to do with correctly detecting a skipped day; it's a
spurious extra condition that misfires specifically on the weekly boundary the code
author was presumably trying to reason about (e.g. "does the streak carry across a
week?") but implemented as an outright block on incrementing rather than any real
week-aware logic.

**My fix and side-effect check:** Removed the `and today.weekday() != 6` clause, leaving
`elif days_since_last == 1:` — the streak now increments purely based on elapsed calendar
days, matching the docstring's stated rules exactly, with no day-of-week special case
anywhere. Ran the full suite (`pytest tests/`) — all 15 tests pass, including
`test_streak_increments_on_sunday` (previously the one persistent failure across all
three fixes) and the four other streak tests (new user, consecutive day, same-day
no-op, skip-a-day reset), none of which I touched. Because this was a boundary-condition
bug, I specifically checked both sides of the Sunday boundary that the existing tests
don't cover: (1) Sunday → Monday (consecutive, crossing the boundary the *other*
direction) now correctly increments 1 → 2, and (2) Friday → Monday (a genuine two-day
skip that happens to span a Sunday) still correctly resets to 1 — confirming the fix
removed the erroneous special case without weakening the real skipped-day reset logic.
