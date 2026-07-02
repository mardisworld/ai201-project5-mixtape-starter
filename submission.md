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

1. feed.py, with route: GET/feed/user_id/listening-now     -->  feed_service.get_friends_listening_now(user_id) 
2. feed.py, with route: GET/feed/user_id/activity          -->  feed_service.get_activity_feed(user_id)   

### Playlists 

1. playists.py, with route: GET/playists/post                -->  playlist_service.create_playlist(name, created_by, is_collaborative)
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
3. users.py, with route:    GET//users/user_id/notifications  --> notification_service.get_notifications(user_id)
4. users.py, with route:    POST//users/notifications/notification_id/read  --> notification_service.read_notification(notification_id) and try notification_service.mark_as_read(notification_id)