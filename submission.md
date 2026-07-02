Mixtape
A social music app where friends share songs, build collaborative playlists, and track listening stats.

This is the starter repo for Project 5: Mixtape Bug Hunt. The app has five open issues in its tracker. Your job is to find, fix, and document at least three of them.

## App Structure

```
ai201-project5-mixtape-starter/
├── app.py                      # Flask app factory and DB setup
├── models.py                   # SQLAlchemy models for all entities
├── routes/
│   ├── songs.py                # Song sharing, search, and rating routes
│   ├── playlists.py            # Playlist creation and song management
│   ├── users.py                # User profiles, streaks, notifications
│   └── feed.py                 # Friends listening now, activity feed
├── services/
│   ├── streak_service.py       # Listening streak logic
│   ├── feed_service.py         # Friends listening now feed logic
│   ├── search_service.py       # Song search logic
│   ├── notification_service.py # Notification creation and retrieval
│   └── playlist_service.py     # Playlist retrieval logic
├── tests/
│   ├── test_streaks.py
│   ├── test_search.py
│   └── test_playlists.py
├── seed_data.py                # Populates DB with test data
├── requirements.txt
└── .gitignore
```

## Codebase Orientation 

## Data Flow

### Feed Routes

1. feed.py, with route: GET/feed/user_id/listening-now        -->  feed_service.get_friends_listening_now(user_id) 
2. feed.py, with route: GET/feed/user_id/activity             -->  feed_service.get_activity_feed(user_id)   

### Playlists 

1. playists.py, with route: POST/playists                    -->  playlist_service.create_playlist(name, created_by, is_collaborative)
2. playists.py, with route: GET/playists//playlist_id        -->  playlist_service.get_detail(playlist_id):
3. playists.py, with route: GET/playists//playlist_id>/songs -->  playlist_service.get_playlist_songs(playlist_id)
4. playists.py, with route: POST/playists/playlist_id/songs  -->  playlist_service.dd_to_playlist(playlist_id, song_id, added_by)


### Songs

1. songs.py, with route:    GET/songs/search                 --> search_service.search_songs(query: str)
2. songs.py, with route:    GET/songs/song_id                --> search_service.get_song(song_id)
3. songs.py, with route:    POST/songs/song_id/rate          --> notification_service.rate_song(user_id: str, song_id: str, score: int)
4. songs.py, with route:    POST/songs/song_id/listen        --> streak_service.record_listening_event(user_id:, song_id)

### Users
1. users.py, with route:    GET/users/user_id                --> will call db.session.get(User, user_id), and return jsonify({"error": "User not found"}) or jsonify(user.to_dict())
2. users.py, with route:    GET/users/user_id/streak         --> streak_service.get_streak(user_id)
3. users.py, with route:    GET/users/user_id/notifications --> notification_service.get_notifications(user_id)
4. users.py, with route:    POST/users/notifications/notification_id/read  --> notification_service.read_notification(notification_id) and try notification_service.mark_as_read(notification_id)

## Services

### feed_service.py

This module is the feed-aggregation service for social listening activity.

1. It is responsible for: Building a “friends listening now” feed via get_friends_listening_now(...) in feed_service.py:14:
    A. Building a “friends listening now” feed via get_friends_listening_now(...) in feed_service.py:14:
    B. Validates the requesting user exists.
    C. Looks at only that user’s friends.
    D. Filters events to a recent window (24 hours via RECENT_THRESHOLD).
    E. Sorts by newest first.
    F. Deduplicates so each friend appears at most once (their latest event only).
    G. Returns normalized dicts with friend, song, and listened_at.

2. Building a broader activity stream via get_activity_feed(...) in feed_service.py:62:
    A. Validates user exists.
    B. Pulls friends’ listening events (no recency cutoff).
    C. Sorts newest first and applies a configurable limit (default 20).
    D. Returns event-by-event dicts with friend, song, and listened_at.

In short: it’s the read-side service that queries listening events and shapes them into API-ready social feed payloads.

### notification_service.py

This module is mainly responsible for managing user notifications and the specific social interactions that trigger them.

1. Core Notification CRUD:
    A. create_notification(): Generates a new notification for a user.
    B get_notifications(): Fetches a user's notifications, with an option to filter for unread only.
    C. mark_as_read(): Updates a specific notification's status to read.
2. Social Interaction Workflows (The Triggers):
    A. add_to_playlist(): Adds a song to a playlist and automatically notifies the user who originally shared the song ("User X added your song 'Y' to playlist 'Z'").
    B. rate_song(): Saves or updates a user's 1-5 score for a particular song.
    C. Essentially, it acts as the business logic layer for recording interactions between friends and alerting users when those happened (where applicable) alerting them about those interactions.

### playlist_service.py

This module handles playlist creation and retrieval. Its four functions:

1. Core Notification CRUD:
    A. create_playlist() — Validates the user exists, then creates and persists a new Playlist (with an optional is_collaborative flag).
    B. get_playlist_songs() — Fetches a playlist's songs ordered by their position in the playlist_entries join table.
    C. get_playlist() — Returns just a playlist's metadata (no songs).
    D. get_user_playlists() — Returns all playlists created by a given user.

    <!--TODO: Bug spotted: In get_playlist_songs() on playlist_service.py:63:. The docstring says "returns all songs in the playlist," but songs[:-1] drops the final song. It should be songs (no slice). -->

### search_service.py

This module is responsible for song lookup. It has two functions:

1. Core Notification CRUD:
    A. search_songs(query) — Searches songs by title or artist using a case-insensitive LIKE match (%query%). It outer-joins the song_tags table so tag data is available, and returns a list of song dicts.
    B. get_song(song_id) — Fetches a single song by its UUID, raising a ValueError if not found.

That's its entire scope — no writes, no side effects, just read-only song retrieval.

### streak_service.py

This module handles daily listening streak tracking. It has three functions:

1. Core Notification CRUD:
    A. record_listening_event() — Creates a ListeningEvent record for a user/song pair, then calls update_listening_streak to update the user's streak before committing both changes together.
    B. update_listening_streak() — Applies the streak rules based on days_since_last listened:
        - First time ever → streak = 1
        - Already listened today → no change
        - Listened yesterday → streak + 1 (with a caveat — see below)
        - Gap of 2+ days → streak resets to 1
    C. get_streak() — Simple read returning the user's current listening_streak integer.

<!--TODO: Bug spotted: In update_listening_streak() on streak_service.py:67: elif days_since_last == 1 and today.weekday() != 6: The today.weekday() != 6 condition skips the streak increment on Sundays (weekday 6), which means listening on a Sunday after Saturday never extends the streak. This looks unintentional — the streak rules in the docstring make no mention of Sunday being excluded. -->

### models.py

models.py defines the entire database schema — every SQLAlchemy model and association table used across the app. Here's the breakdown:

Association Tables (join tables with no model class of their own):

    A.friendships — many-to-many self-referential User↔User relationships
    B. song_tags — many-to-many Song↔Tag relationships
    C. playlist_entries — many-to-many Playlist↔Song with extra columns (position, added_by, added_at)


Model Classes (each has a to_dict() method that serializes it for API responses):

| Model	            | Key fields	                                        | Relationships
| User	            |username, email, listening_streak, last_listened_at	| friends (self-ref), shared songs, ratings, listening events, notifications, playlists
| Tag	            | name	                                                | (no to_dict)
| Song	            | title, artist, album, genre, shared_by, share_note	| ratings, listening events, tags
| ListeningEvent	| user_id, song_id, listened_at	                        | —
| Rating	        | user_id, song_id, score (1–5)	                        | unique constraint per user+song pair
| Playlist	        | name, created_by, is_collaborative	                | songs (via playlist_entries)
| Notification	    | user_id, notification_type, body, read	            | —

All primary keys are UUID strings generated by the generate_uuid() helper at the top of the file.

## Bug Fixes

<!--For Issue #3 (duplicates), the bug is conditional — think about what conditions trigger the second code path that produces the duplicate.

For Issue #4 (notifications), the root cause is architectural, not a typo. Look at the pattern used for the working notification and compare it line-by-line to the missing one.-->




## AI Usage
1. As directed by the project instructions, I gave AI the contents of a service file to summarize what is this module responsible for? and What are its main functions and wht does each one do? 

**Full disclosure:** In doing this, the AI tool identified two bugs it identified: the first in playlist_service.py": 

"Bug spotted: In get_playlist_songs() on playlist_service.py:63:. The docstring says "returns all songs in the playlist," but songs[:-1] drops the final song. It should be songs (no slice)."

The second bug it identified was in streak_service.py.

"Bug spotted: In update_listening_streak() on streak_service.py:67: elif days_since_last == 1 and today.weekday() != 6: The today.weekday() != 6 condition skips the streak increment on Sundays (weekday 6), which means listening on a Sunday after Saturday never extends the streak. This looks unintentional — the streak rules in the docstring make no mention of Sunday being excluded."

In both cases, it asked if it wanted me to fix the bug, but I delcined in order to follow the project instructions. 

2.  I asked AI to validate that all three test files (test_playlists.py, test_search.py, and test_streaks.py) were correct and complete. 

### test_playlists.py — Correct, but incomplete coverage
The 3 tests are well-structured. test_playlist_returns_all_songs correctly expects 5 songs and will fail due to the songs[:-1] bug identified earlier. 

**Missing coverage:** 
    1. get_playlist_songs with an invalid playlist_id (should raise ValueError)
    2. create_playlist is imported but never tested

### test_search.py — Has a bug in the service, tests are correct
The 5 tests are well-written and will correctly catch the duplicate-results bug in search_service.py. This identified another bug (the duplicate results bug in search_service.py, which had not been identified before.). 

The issue: search_songs uses .outerjoin(song_tags, ...) without a .distinct(), so songs with multiple tags appear multiple times in results. test_search_no_duplicates_multi_tag_song will fail until that's fixed. 

Missing coverage:
    - Case-insensitive matching (e.g. query "borough" should match "Borough Kings")
    - get_song is not tested at all (not even imported)


### test_streaks.py
For test_streaks.py, it said all 5 bugs are are well-written and cover the key cases: new user, consecutive days, same-day double count, skipped day, and the Sunday edge case. Notably, test_streak_increments_on_sunday directly tests the bug flagged in streak_service.py (the weekday() != 6 condition) — so this test will fail until that bug is fixed.

