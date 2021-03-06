API
===

The Wissl server exposes a REST API that allows interaction with any client
able to send simple HTTP requests and parse JSON responses.

This document describes this API so that you may write such a client.

Summary
-------

* Data types
  * [user](#user)
  * [session](#session)
  * [artist](#artist)
  * [album](#album)
  * [song](#song)
  * [playlist](#playlist)
* API methods
  * [`/login`](#login)
  * [`/logout`](#logout)
  * [`/hasusers`](#hasusers)
  * [`/users`](#users)
  * [`/user/{user_id}`](#useruser_id)
  * [`/user/add`](#useradd)
  * [`/user/password`](#userpassword)
  * [`/user/remove`](#userremove)
  * [`/playlist/create`](#playlistcreate)
  * [`/playlist/create-add`](#playlistcreate-add)
  * [`/playlist/random`](#playlistrandom)
  * [`/playlist/{playlist_id}/add`](#playlistplaylist_idadd)
  * [`/playlist/{playlist_id}/remove`](#playlistplaylist_idremove)
  * [`/playlist/{playlist_id}/songs`](#playlistplaylist_idsongs)
  * [`/playlist/{playlist_id}/song/{song_pos}`](#playlistplaylist_idsongsong_pos)
  * [`/playlists/remove`](#playlistsremove)
  * [`/playlists`](#playlists)
  * [`/playlists/{user_id}`](#playlistsuser_id)
  * [`/artists`](#artists)
  * [`/albums/{artist_id}`](#albumsartist_id)
  * [`/songs/{album_id}`](#songsalbum_id)
  * [`/song/{song_id}`](#songsong_id)
  * [`/song/{song_id}/stream`](#songsong_idstream)
  * [`/search/{query}`](#searchquery)
  * [`/recent/{number}`](#recentnumber)
  * [`/art/{album_id}`](#artalbum_id)
  * [`/folders`](#folders)
  * [`/folders/listing`](#folderslisting)
  * [`/folders/add`](#foldersadd)
  * [`/folders/remove`](#foldersremove)
  * [`/indexer/status`](#indexerstatus)
  * [`/indexer/rescan`](#indexerrescan)
  * [`/edit/artist`](#editartist)
  * [`/edit/album`](#editalbum)
  * [`/edit/song`](#editsong)
  * [`/edit/artwork/{album_id}`](#editartworkalbum_id)
  * [`/logs`](#logs)
  * [`/stats`](#stats)
  * [`/info`](#info)
  * [`/shutdown`](#shutdown)

Preamble
--------

* All communications with the server use the HTTP/1.1 protocol.
* For all communications with the server, the response will be encoded
  in UTF-8 and use the media-type `application/json`.
* Except when said otherwise, all requests will need to provide authentication
  using a parameter named `sessionId` which will contain the current session
  UUID. This can be achieved with either:
  * an HTTP header, ie:
   `curl -H "sessionId: af0ee222-6ed1-409d-9d99-5654c7802df1" http://localhost/wissl/req`
  * an HTTP query parameter, ie:
  `curl http://localhost/wissl/req?sessionId=af0ee222-6ed1-409d-9d99-5654c7802df1`  
  This sessionId will be provided once upon login.

Data types
----------

Here are defined the data types that will appear in the responses of the server.
Those types will be referenced later to avoid redundancy.

### <a id="user"></a>user

Each registered account is represented by an user.

    {
      // unique user id
      "id": INT,
      // user name
      "username": STRING,
      // 1: admin; 2: regular user
      "auth": INT,
      // total number of bytes downloaded
      "downloaded": INT
    }

### <a id="session"></a>session

When an user is connected, a Session is bound to it.
An unique user can have multiple active concurrent sessions.

    {
      // unique user id
      "user_id": INT,
      // user name
      "username": STRING,
      // milliseconds since last server activity
      "last_activity": INT,
      // milliseconds since session creation
      "created_at": INT,
      // ADMIN ONLY: milliseconds since last access for each client IP
      "origins": [
	     {
           "ip": STRING,
           "user_agent": STRING,
		   "time": INT
         }, ...
	  ],
      // last played song, or empty
      "last_played_song": SONG
    }

### <a id="artist"></a>artist

An artist contains several albums.

    {
      // unique artist id
      "id": INT,
      // artist name
      "name": STRING,
      // number of albums
      "albums": INT,
      // number of songs
      "songs": INT,
      // playtime for all songs in seconds
      "playtime": INT,
      // time since added in DB in ms
      "date_added": LONG
    }

### <a id="album"></a>album

An album contains several songs, and is contained by one artist.

    {
      // unique album id
      "id": INT,
      // album name
      "name": STRING,
      // unique artist id
      "artist": INT,
      // artist name
      "artist_name": STRING,
      // album release date
      "date": STRING,
      // musical genre
      "genre": STRING,
      // number of songs
      "songs": INT,
      // playtime for all songs in seconds
      "playtime": INT,
      // time since added in DB in ms
      "date_added": LONG,
      // true if the server has an artwork for this album
      "artwork": BOOL
      // will change if artwork changes, use it for cache control
      "artwork_id": STRING
    }

### <a id="song"></a>song

A song is contained by one album and one artist.
Each song can be added to multiple playlists.

    {
      // Unique song id
      "id": INT,
      // song title
      "title": STRING,
      // song position in album
      "position": INT,
      // disc number when multiple volumes
      "disc_no": INT,
      // song duration in seconds
      "duration": INT,
      // audio mimetype, ie 'audio/mp3'
      "format": STRING,
      // unique album id
      "album_id": INT,
      // album name
      "album_name": STRING,
      // unique artist id
      "artist_id": INT,
      // artist name
      "artist_name": STRING,
    }

### <a id="playlist"></a>playlist

A playlist is a list of songs created by an user.

    {
      // unique playlist id
      "id": INT,
      // playlist name
      "name": STRING,
      // unique user id, playlist owner
      "user": INT,
      // number of songs
      "songs": INT,
      // total duration of all songs in seconds
      "playtime": INT
    }

API methods
-----------

### <a id="login"></a>`/login`
* method: `POST`
* param: `username` a valid non-empty user name
* param: `password` cleartext password matching the username
* ex: `curl -d 'username=user&password=pwd' http://localhost/wissl/login`
* returns:<pre>
    {
      // unique user id
      "userId": INT,
      // session UUID
      "sessionId": STRING,
      // 1: admin; 2: regular user
      "auth": INT
    }
</pre>

The password is sent in clear-text.
To prevent eavesdropping, configure the server to use SSL.

Use the session id provided in the response to authenticate for each further call.
The session id can be used either as an HTTP header or an HTTP query parameter,
both named `sessionId`.

If a session already exists for this user, a new session will be created.
Each session is identified by an unique identifier.

When a session is not used for some time, it is removed.

### <a id="logout"></a>`/logout`
* method: `POST`
* no parameter
* ex: `curl -X POST -H 'sessionId:UUID' http://localhost/wissl/logout`
* does not return any reponse

Destroys the session associated with the UUID in the request.

### <a id="hasusers"></a>`/hasusers`
* method: `GET`
* no parameter
* does not requires authentication
* ex: `curl http://localhost/wissl/hasusers`
* returns:<pre>
    {
      "hasusers": BOOL
    }
</pre>

Checks whether the server already has an user registered.

If it does (`hasusers === true`), then only the registered users can log in.

If it does not, then any client can call `/user/add` to create the initial user without authentication.

### <a id="users"></a>`/users`
* method: `GET`
* no parameter
* ex: `curl -H 'sessionId:UUID' http://localhost/wissl/users`
* returns:<pre>
    {
	  // all known users
      "users": [
	    USER, ...
      ],
	  // currently active sessions
	  "sessions": [
        SESSION, ...
	  ]
    }
</pre>

List all users and all sessions.

Allows clients to list users and check which is currently logged in.

### <a id="useruser_id"></a>`/user/{user_id}`
* method: `GET`
* param: `user_id` unique user id
* ex: `curl -H 'sessionId:UUID' http://localhost/wissl/user/5`
* returns:<pre>
    {
      "user": USER,
      "sessions": [
         SESSION, ...
      ]
      "playlists": [
        PLAYLIST, ...
      ]
    }
</pre>

Retrieve information about a specific user.

The user id used as parameter can be found in the response of the `/users` request.

### <a id="useradd"></a>`/user/add`
* method: `POST`
* param: `username` name of the user to create
* param: `password` clear text password
* param: `auth` int authorization level. 1: admin, 2: user
* ex: `curl -H 'sessionId:UUID' -d 'username=user&password=pwd&auth=2' http://localhost/wissl/user/add`
* ex: `curl -d 'username=admin&password=pwd&auth=1 http://localhost/wissl/user/add`
* requires admin privileges
* returns:<pre>
    {
      // newly created user
      "user": USER
    }
</pre>

Create a new user account.
Clear-text password is sent as parameter. Configure the server to use SSL to prevent eavesdropping.

As shown by the examples, two distinct scenarios can be distinguished:

* Add the first user of the system right after a clean startup. No authentication is required.
  The first user must be admin (`auth === 1`).
* Add a new user. The client must authenticate as an admin user to perform this operation.

The provided password can be changed later using `/user/password`.

### <a id="userpassword"></a>`/user/password`
* method: `POST`
* param: `old_password` re-authenticate user with old password
* param: `new_password` new password to set
* ex: `curl -H 'sessionId:UUID' -d 'old_password=pwd&new_password=pwd2 http://localhost/wissl/user/password`
* does not return anything

Change the password for the user associated with the current session/
Clear-text password is sent as parameter. Configure the server to use SSL to prevent eavesdropping.

### <a id="userremove"></a>`/user/remove`
* method: `POST`
* param: `user_ids[]` array of unique user ids
* ex: `curl -H 'sessionId:UUID' -d 'user_ids=3&user_ids=5' http://locahost/wissl/user/remove`
* does not return anything

Remove one or more user accounts from the server.

You can not remove the user account that you are currently authenticated with.

You can not remove the last admin user account.

### <a id="playlistcreate"></a>`/playlist/create`
* method: `POST`
* param: `name` name of the new playlist
* ex: `curl -H 'sessionId:UUID' -d 'name=foo' http://localhost/wissl/playlist/create`
* returns:<pre>
    {
      "playlist": PLAYLIST
    }
</pre>

Create a new empty playlist.

If a playlist with the same name already exists for this user,
no playlist is created and the existing playlist is returned instead.

### <a id="playlistcreate-add"></a>`/playlist/create-add`
* method: `POST`
* param: `name` name of the new playlist
* param: `clear` optional boolean. If true, clears a pre-existing playlist with the same name
* param: `song_ids[]` array of song ids to add to the playlist
* param: `album_ids[]` array of album ids to add to the playlist
* ex: `curl -H 'sessionId:UUID' -d 'name=foo&songs_ids=5&songs_ids=7' http://localhost/wissl/playlist/create-add`
* returns:<pre>
    {
      // number of songs added to the playlist
      "added": INT,
      // playlist that was just created
      "playlist": PLAYLIST
    }
</pre>

Create a new playlist and add songs to it.

If the optional boolean parameter `clear` is set to true and a playlist with the same `name` already
exists, it will be cleared before adding the songs specified by the parameters.

Both parameters `song_ids` and `album_ids` are optional and can be used simultaneously.

Duplicate song records are not allowed.

### <a id="playlistrandom"></a>`/playlist/random`
* method: `POST`
* param: `name` name of the random playlist
* param: `number` number of random songs to add
* ex: `curl -H 'sessionId:UUID' -d 'name=random&number=20' http://localhost/wissl/playlist/random`
* returns:<pre>
    {
      // number of songs added to the random playlist
      "added": INT,
      // unique song id of the first song in the playlist
      "first_song": INT,
      // playlist that was created
      "playlist": PLAYLIST
    }
</pre>

Creates a new playlist that contains randomly picked songs.

If a playlist with that name already exists, it will be cleared.

The maximum number of songs that can be requested is 50.

The number of songs added to the playlist may be less than requested
if the library contains less than 50 songs.

### <a id="playlistplaylist_idadd"></a>`/playlist/{playlist_id}/add`
* method: `POST`
* param: `playlist_id` unique playlist id
* param: `clear` optional boolean. If true, clears the playlist
* param: `song_ids[]` array of song ids to add to the playlist
* param: `album_ids[]` array of album ids to add to the playlist
* ex: `curl -H 'sessionId:UUID' -d 'song_ids=5&song_ids=7' http://localhost/wissl/playlist/3/add`
* returns:<pre>
    {
      // number of songs added to the playlist
      "added": INT,
      "playlist": PLAYLIST
    }
</pre>

Add songs to an existing playlist.

If the optional boolean parameter `clear` is set to true,
the playlist will be cleared before adding the songs specified by the parameters.

Both parameters `song_ids` and `album_ids` are optional and can be used simultaneously.

An user can only alter a playlist that it has created.

Duplicate song records are not allowed.

### <a id="playlistplaylist_idremove"></a>`/playlist/{playlist_id}/remove`
* method: `POST`
* param: `playlist_id` unique playlist id
* param: `song_ids[]` array of song ids to remove from the playlist
* ex: `curl -H 'sessionId:UUID' -d 'song_ids=3&song_ids=7' http://localhost/wissl/playlist/3/remove`
* does not return anything

Remove songs from a playlist.

An user can only alter a playlist that it has created.

### <a id="playlistplaylist_idsongs"></a>`/playlist/{playlist_id}/songs`
* method: `GET`
* param: `playlist_id` unique playlist id
* ex: `curl -H 'sessionId:UUID' http://localhost/wissl/playlist/3/songs`
* returns:<pre>
    {
      // playlist name
      "name": STRING,
      // array of songs
      "playlist": [
        SONG, ...
      ]
    }
</pre>

List the content of a playlist.

The ordering of the `playlist` song array reflects the playlist play order.

### <a id="playlistplaylist_idsongsong_pos"></a>`/playlist/{playlist_id}/song/{song_pos}`
* method: `GET`
* param: `playlist_id` unique playlist id
* param: `song_pos` song position in playlist
* ex: `curl -H 'sessionId:UUID' http://localhost/wissl/{playlist_id}/song/{song_pos}`
* returns:<pre>
    {
      "song": SONG
    }
</pre>

Get a single song at a given position in a playlist.

### <a id="playlistsremove"></a>`/playlists/remove`
* method: `POST`
* param: `playlist_ids[]` array of playlist ids to remove
* does not return anything

Remove one or more playlists.

An user can only remove playlists that it has created.

### <a id="playlists"></a>`/playlists`
* method: `GET`
* ex: `curl -H 'sessionId:UUID' http://localhost/wissl/playlists`
* returns:<pre>
    {
      "playlists": [
        PLAYLIST, ...
      ]
    }
</pre>

Get all playlists created by the authenticated used.

### <a id="playlistsuser_id"></a>`/playlists/{user_id}`
* method: `GET`
* param: `user_id` unique user id
* ex: `curl -H 'sessionId:UUID' http://localhost/wissl/playlists/3`
* returns:<pre>
    {
      "playlists": [
        PLAYLIST, ...
      ]
    }
</pre>

Get all playlists created by a given user.

### <a id="artists"></a>`/artists`
* method: `GET`
* ex: `curl -H 'sessionId:UUID' http://localhost/wissl/artists`
* returns:<pre>
    {
      "artists": [
        {
          "artist": ARTIST,
          // list of album ids that have artworks
          "artworks": [
            {
              "album": INT,
              "id": STRING
            },
            ...
          ]
        },
        ...
      ]
    }
</pre>

Get all artists.

The returned object contains both the artist information and the ids of all albums that
have an artwork ready to be served by the method `/art/{album_id}`.

In the `artworks` array are described the ids of the albums that have artworks,
along with the `artwork_id` string that identifies an image and can be used to control
client cache.

### <a id="albumsartist_id"></a>`/albums/{artist_id}`
* method: `GET`
* param: `artist_id` unique artist id
* ex: `curl -H 'sessionId:UUID' http://localhost/wissl/albums/{artist_id}`
* returns:<pre>
    {
      "artist": ARTIST,
      "albums": [
        ALBUM, ...
      ]
    }
</pre>

Get all albums for a given artist.

### <a id="songsalbum_id"></a>`/songs/{album_id}`
* method: `GET`
* param: `album_id` unique album id
* ex: `curl -H 'sessionId:UUID' http://localhost/wissl/songs/5`
* returns:<pre>
    {
      "artist": ARTIST,
      "album": ALBUM,
      "songs": [
        SONG, ...
      ]
    }
</pre>

Get all songs for a given album.

### <a id="songsong_id"></a>`/song/{song_id}`
* method: `GET`
* param: `song_id` unique song id
* ex: `curl -H 'sessionId:UUID' http://localhost/wissl/song/10`
* returns:<pre>
    {
      "artist": ARTIST,
      "album": ALBUM,
      "song": SONG
    }
</pre>

Get a specific song.

### <a id="songsong_idstream"></a>`/song/{song_id}/stream`
* method: `GET`
* param: `song_id` unique song id
* ex: `mplayer http://localhost/wissl/song/13/stream?sessionId=UUID`
* ex: `curl -H 'sessionId:UUID' http://localhost/wissl/song/13/stream`
* returns: raw binary content of the song file

Read the music stream for a specific song.

The binary file is directly served, suitable to be read by any audio player that
understands the format.

This method supports seeking using HTTP range headers.
The reponse advertises remaiming byte length using the `Content-Length` header,
and will begin reading at the byte specified by the `range` request header.

### <a id="searchquery"></a>`/search/{query}`
* method: `GET`
* param: `query` search query string
* ex: `curl -H 'sessionId:UUID' http://localhost/wissl/search/foo`
* returns:<pre>
    {
      "artists": [
        ARTIST, ...
      ],
      "albums": [
        ALBUM, ...
      ],
      "songs": [
        SONG, ...
      ]
    }
</pre>

Search for songs, albums and artists with a single string query.
The search query will be matched against song titles, artist names and album names.
For example, the query 'bea' will match artist 'The Beatles' and song 'Heartbeat'.

### <a id="recentnumber"></a>`/recent/{number}`
* method: GET
* param: `number` number of elements to return
* ex: `curl -H 'sessionId:UUID' http://localhost/wissl/recent/5
* returns:<pre>
    {
	  "artists": [
	    ARTIST, ...
	  ],
	  "albums": [
	    ALBUM
	  ]
	}
</pre>

Returns the albums and artists that have been the most recently added into the DB.
The query parameter sets the maximum number of elements returned for both artists and albums.

### <a id="artalbum_id"></a>`/art/{album_id}`
* method: `GET`
* param: `album_id` unique album id
* does not require authentication
* ex: `curl http://localhost/wissl/art/6`
* returns: raw binary image content

Download the album art image.

If no artwork is available for this album, you will get a 404 Not Found error.

Authentication is not required because it would prevent caching for some browsers,
since they would try to download the image with a session id as query paremeter.

### <a id="folders"></a>`/folders`
* method: `GET`
* no parameter
* requires admin privileges
* ex: `curl -H 'sessionId:UUID' http://localhost/wissl/folders`
* returns:<pre>
    {
      // array of folder paths relative to the server filesystem
      "folders": [
        STRING, ...
      ]
</pre>

Lists the folders that are indexed as part as the music library.

These folders are relative to the server's local filesystem.

### <a id="folderslisting"></a>`/folders/listing`
* method: `GET`
* param: `directory` optional directory to list, or `$ROOT` to view filesystem root
* requires admin privileges
* ex: `curl -H 'sessionId:UUID' http://localhost/wissl/folders/listing?directory=/tmp/`
* returns:<pre>
    {
      // directory listed
      "directory": STRING.
      // platform path separator
      "separator": STRING
      // parent directory
      "parent": STRING,
      // directories contained
      "listing": [
        STRING, ...
      ]
    }
</pre>

List directory on the server's filesystem.

Does not display single files, only directories.

This method allows an administrator to browse the server's filesystem to look for
a new folder to add to the library indexer path using `/folders/add`.

If the optional `directory` parameter is empty, the directory listed will
be the home of the user running the server process.

If the query parameter is `$ROOT`, this method will list the filesystem root.
This is equivalent to `/` on unix systems, but allows listing all drives on Windows.

### <a id="foldersadd"></a>`/folders/add`
* method: `POST`
* param: `directory` absolute path of the directory to add
* requires admin privileges
* ex: `curl -H 'sessionId:UUID' -d 'directory=/music/ http://localhost/wissl/folders/add`
* does not return anything

Add a folder to the music library indexation system.

This folder will be regularly scanned and the music files it contains will be indexed by the server.

The parameter `directory` is a directory relative to the server's local filesystem.
This directory can be discovered by using the `/folders/listing` method.

### <a id="foldersremove"></a>`/folders/remove`
* method: `POST`
* param: `directory[]` array of absolute paths to remove from the library indexation system
* requires admin privileges
* ex: `curl -H 'sessionId:UUID' -d 'directory=/tmp&directory=/music' http://localhost/wissl/folders/remove`
* does not return anything

Remove one or more folders from the music library indexation system.

The folders currently indexed can be listed with the method `/folders`.

### <a id="indexerstatus"></a>`/indexer/status`
* method: `GET`
* no parameter
* requires admin privileges
* ex: `curl -H 'sessionId:UUID' http://localhost/wissl/indexer/status`
* returns:<pre>
    {
      // false when idle, true when indexing
      "running": BOOL,
      // when indexing, estimates percent done in [0.0;1.0]
      "percentDone": DOUBLE,
      // when indexing, estimates time left in seconds
      "secondsLeft": INT,
      // when indexing, total songs indexed
      "songsDone": INT,
      // when indexing, total songs to index
      "songsTodo": INT
    }
</pre>

### <a id="indexerrescan"></a>`/indexer/rescan`
* method: `POST`
* no parameter
* requires admin privileges
* ex: `curl -X POST -H 'sessionId:UUID' http://localhost/wissl/indexer/rescan`
* does not return anything

Forces library indexer to rescan all folders now.

If it is currently already running, it will be stopped and restarted.

### <a id="editartist"></a>`/edit/artist`
* method: `POST`
* param: `artist_ids[]` array of unique artist id to edit
* param: `artist_name` new artists name
* requires admin privileges
* ex: `curl -H "sessionId:UUID" -d "artist_ids[]=2&artist_name=foo" http://localhost:8080/wissl/edit/artist`
* does not return anything

Edit metadata for one or more artists.

The only field that can be edited at the artist level is the artist name.

This method will write the new metadata to the actual music file on the filesystem
and update the DB synchronously: request might take up to a few minutes to complete
depending the number of files and disk performances.

### <a id="editalbum"></a>`/edit/album`
* method: `POST`
* param: `album_ids[]` array of unique album id to edit
* param: `album_name` new album name
* param: `artist_name` new artist name
* param: `date` new date
* param: `genre` new genre
* requires admin privileges
* ex: `curl -H "sessionId:UUID" -d "album_ids[]=2&genre=metal" http://localhost:8080/wissl/edit/album`
* does not return anything

Edit metadata for one or more albums.

Each parameter is optional; one can call this method and only change the `genre` field for
a list of albums.

This method will write the new metadata to the actual music file on the filesystem
and update the DB synchronously: request might take up to a few minutes to complete
depending the number of files and disk performances.

### <a id="editsong"></a>`/edit/song`
* method: `POST`
* param: `song_ids[]` array of unique song ids to edit
* param: `song_name` new song name
* param: `position` new position in album
* param: `disc_no` new disc number
* param: `album_name` new album name
* param: `artist_name` new artist name
* requires admin privileges
* ex: `curl -H "sessionId:UUID" -d "song_ids[]=2&position=4" http://localhost:8080/wissl/edit/song`
* does not return anything

Edit metadata for one or more songs.

Each parameter is optional; one can call this method and only change the `disc_no` field for
a list of songs.

This method will write the new metadata to the actual music file on the filesystem
and update the DB synchronously: request might take up to a few minutes to complete
depending the number of files and disk performances.

### <a id="editartworkalbum_id"></a>`/edit/artwork/artwork_id`
* method: `POST`
* param: `album_id` unique album id
* param: `file` image file sent as part using `multipart/form-data`
* requires admin privileges
* ex: `curl -H "sessionId:UUID" -F 'file=@/path/to/image.jpg' http://localhost/wissl/edit/artwork/142`
* does not return anything

Edit album artwork for a given album.

This method will write the new metadata to the actual music file on the filesystem
and update the DB synchronously: request might take up to a few minutes to complete
depending the number of files and disk performances.

### <a id="logs"></a>`/logs`
* method: `POST`
* no parameter
* requires admin privileges
* ex: `curl -H 'sessionId:UUID' http://localhost/wissl/logs`
* returns: log file content as plain text

List log file content.

Logs are rotated after a number of lines have been appended to it.
This method only returns the content of the last file, to investigate on recent events.

### <a id="stats"></a>`/stats`
* method: `GET`
* no parameter
* ex: `curl -H 'sessionId:UUID' http://localhost/wissl/stats`
* returns:<pre>
    {
      "stats": {
        // total number of songs indexed
        "songs": INT,
        // total number of albums indexed
        "albums": INT,
        // total number of artists indexed
        "artists": INT,
        // total number of playlists
        "playlists": INT,
        // total number of users
        "users": INT,
        // cumulated duration of all songs in seconds
        "playtime": INT,
        // cumulated downloaded bytes since startup
        "downloaded": INT,
        // time since server startup in seconds
        "uptime": INT
      }
    }
</pre>

Return a short list of runtime statistics properties for the server.

### <a id="info"></a>`/info`
* method: `GET`
* path: `/info`
* no parameter
* ex: `curl -H 'sessionId:UUID' http://localhost/wissl/info`
* returns:<pre>
    {
      // server release version
      "version": STRING,
      // server build number
      "build": STRING,
      // java servlet container
      "server": STRING,
      // operating system
      "os": STRING,
      // java version
      "java": STRING
    }
</pre>

Returns various information about the server runtime environment.

### <a id="shutdown"></a>`/shutdown`
* method: `POST`
* path: `/shutdown`
* no parameter
* requires admin privileges
* ex: `curl -H 'sessionId:UUID' -X POST http://localhost/wissl/shutdown`
* does not return anything

Shutdown the server.
