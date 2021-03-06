QueueUp Server
===========

QueueUp is a collaborative playlist streaming service. Anybody can create a playlist, use Spotify to stream from QueueUp, and invite friends to contribute in real time.

[**Get it on Google Play**](https://play.google.com/store/apps/details?id=org.louiswilliams.queueupplayer)

[**Get it on the App Store**](https://itunes.apple.com/us/app/queueup-player/id1028056241?mt=8)

A QueueUp *Player* is requried to stream from QueueUp. This repository is for the QueueUp *Server*. Read below about the Android and iOS *Players*.

### [Live Site](http://queueup.io/)

![Playlist](public/images/screen1small.png)
![Playlist](public/images/screen3small.png)

## Contents

* [Setup](#setup)
* [Runnning](#running)
* [Players](#players)
    * [Implementation](#implementation)
* [REST API](#rest-api)
    * [Authenticated Routes](#authenticated-routes)
    * [Authentication](#authentication)
        * [Step 1](#step-1-obtaining-a-user_id-and-client_token-secret)
        * [Step 2](#step-2-authenticating)
    * [Search Routes](#search-routes)
    * [Playlist Routes](#playlist-routes)
    * [User Routes](#user-routes)
* [Socket.io API](#socketio-api)
    * [Step 1](#step-1-authenticate-to-gain-access)
    * [Step 2](#step-2-register-as-a-client-or-player)
        * [Register as a Client](#register-as-a-client-read-only-updates)
        * [Register as a Player](#register-as-a-player-one-per-playlist)
* [Objects](#objects)

## Setup

The `/spotify.key` configuration file is required to run the server  properly. An example configuration file is located in `/spotify.key.example`. Most of the requried parameters can be obtained by creating Spotify Developer account, and then a [Spotify Application](https://developer.spotify.com/my-applications). The `encryptionSecret` is your password to encrypt refresh tokens on the server side.

The `/env.json` configuration file is required (env.json.example is provided)with the fields:
  - *name* (environment)
  - *host* (current hostname, like queueup.louiswilliams.org)
  - *port* (server listen port)
  - *database* (name of MongoDB database to use)

A MongoDB Server should be running on `localhost:27017`. This is configurable in `server.js`.

    npm install

## Running

    npm start


## Players

A QueueUp Player is required to play from [QueueUp](http://qup.louiswilliams.org). It connects to the server, subscribes to a playlist, and updates automatically to play music from a playlist.

Available Players:

  - [Android Player](https://play.google.com/store/apps/details?id=org.louiswilliams.queueupplayer): Get it on the Google Play Store
  - [iOS](https://itunes.apple.com/us/app/queueup-player/id1028056241?mt=8): Get it on the iOS App store
  - [Node.js Player](https://github.com/extrakt/queueup-spotify-client): Requires some setup, but effectively the same as the Android player, just on a desktop platform.

Notes:

  - All players require Spotify Premium accounts. This is a result of Spotify's streaming licensing, and there is no legal way around it. Consider buying one. As a student ($5/mo), it is one of the best decisions I've made in my adult life.
  - No web streaming API exists, again, because of music licensing issues with Spotify. Currently, the streaming APIs are limited to Android, iOS, and C (personal use developer accounts only).

### Implementation

A Player can be implemented using a mixture of REST and Socket.IO APIs.

In terms of the API, a **Client** is a read-only listener that subscribes to playlist updates. A **Player** is a **Client** that can also send updates about the current state of the playing track. Only one **Player** is allowed to play at a time for a given playlist.


## REST API

For requests that do not require event-based socketed connections, like searching for and updating playlist information. See **Objects** section for schema.

*Note: All responses send 200 codes on success, 400 on client errors, 403 on unauthorized access, and 500 on server errors. 4xx errors contain an `error` attribute, with an error description, `error.message`*

### Authenticated Routes

Authenticated routes require the HMAC scheme described below. An attempt to access an authenticated route without the `Authorization` header or if the user isn't found return 403 errors, all other bad requests return 400 (like an invalid hash). 


### Authentication

#### Step 0: Requesting anonymous user credentials

Beacause of the the desire to have anonymous users, call this route to obtain a `user_id` and `client_token`, which must be used in step 1.

- POST `/api/v2/auth/init`: Register an anonymous account
    - **Input**: `{device: {id: String}}`: Register with a unique device identifier
    - **Returns**: `{user_id: String, client_token: String}`: **Save these for later requests**

#### Step 1: Registering for the first time

*Note: a request to both of these routes REASSIGNS a `client_token` and invalidates the current one, if it exists.*

- POST `/api/v2/auth/register`: Register an account for the first time (without Facebook)
    - **Input**: `{user_id: String, client_token: String, email: String, password: String, name: String}`: Register with an name/email/password
    - **Returns**: `{user_id: String, client_token: String}`: **Save these for API requests**
- POST `/api/v2/auth/login`: Log in to receive a `client_token` for API requests
    - **Input**: Choose ONE:
        - `{email: String, password: String}`: Log in with an existing email/password
        - `{user_id: String, client_token: String, facebook_access_token: String}`: Log in with a valid FB access token.
          - *Note: a user_id/client_token are only required for the very first login attempt*
    - **Returns**: `{user_id: String, client_token: String}`: **Save these for API requests**


#### Step 2: Authenticating

To authenticate, the server uses an HMAC-SHA1 scheme. There are 2 requried HTTP headers:

* `Date`: RFC2822 or ISO 8601 formatted date
* `Authorization`: Basic HTTP authentication using the base64 encoded string in the form `user_id:HMAC_HASH`

Where `user_id` is received from logging in. The `HMAC_HASH` is the output of using `client_token` as the key of the HMAC algorithm with the following as input: 

    HTTP_METHOD+HOSTNAME+URI+UNIX_SECONDS

##### Example

Assume the following request by the user_id `cafebabecafebabe`, and client_token `secret` sent *Saturday, 11-Jul-15 21:00:03 UTC* (RFC2822 time):

    POST http://queueup.louiswilliams.org/api/v2/playlists/c0ffeec0ffee/rename

This message is hashed:

    POST+queueup.louiswilliams.org+/api/v2/playlists/c0ffeec0ffee/rename+1436648403

which yields `2871715b0c9fbf688de5104f83d6c800f30cbe34`. The string

    cafebabecafebabe:2871715b0c9fbf688de5104f83d6c800f30cbe34

is Base64 encoded to yield

    Y2FmZWJhYmVjYWZlYmFiZToyODcxNzE1YjBjOWZiZjY4OGRlNTEwNGY4M2Q2YzgwMGYzMGNiZTM0

The appropriate headers are then:

    Date: Saturday, 11-Jul-15 21:00:03 UTC
    Authorization: Basic Y2FmZWJhYmVjYWZlYmFiZToyODcxNzE1YjBjOWZiZjY4OGRlNTEwNGY4M2Q2YzgwMGYzMGNiZTM0

*Note: Dates must be withing 5 minutes of server time to prevent replay*

**Spotify Token Routes** (to obtain access tokens)

- POST `/api/v2/spotify/swap`: Swap an authorization code for access tokens
    - **Input**: `{code: String}`: Authorization code
    - **Returns**: `{access_token: String, refresh_token: String, expires_in: Number}`: An access token and (encrypted) refresh token to be stored for later retrieval.
- POST `/api/v2/spotify/refresh`: Exchange encrypted refresh token for an access token
    - **Input**: `{refresh_token}`: Encrypted refreh token obtained from the swap step
    - **Returns**: `{access_token: String, expires_in: Number}`: New access token


### Search Routes

- GET `/api/v2/search/tracks/:query/[:offset]`: Search for tracks with a page offset
    - **Returns**: `{tracks: [Track]}`: Array (max 10) of Spotify *Track* objects. Use the offset at multiples of 10 to get more results.
- GET `/api/v2/search/playlists/:query`: Search for playlists
    - **Returns**: `{playlists: [Playlist]}`: Array of top 10 matches to *Playlist* objects (by name)

### Playlist Routes

- GET `/api/v2/playlists`: Get a list of playlists
    - **Returns**: `{playlists: [Playlist]}`: Array of *Playlist* objects (without tracks).
- GET `/api/v2/playlists/:playlist_id`: Get details for a playlist, by `_id`.
    - **Returns**: `{playlist: Playlist}`: A *Playlist* object. 
- POST `/api/v2/playlists/nearby`: Show playlists within 2.0 mi radius
    - **Input**: `{location: {latitude: Double, longitude: Double}}`: Coordinates of the user creating the playlist
    - **Returns**: `{playlist: Playlist}`: New *Playlist* object. This object contains the `distance` field, which is equal to the distance to the playlist in meters.
- POST `/api/v2/playlists/new`: Create new playlist
    - **Input**: `{playlist: {name: String}}`: New playlist object (with name)
    - **Returns**: `{playlist: Playlist}`: New *Playlist* object.
- POST `/api/v2/playlists/:playlist_id/rename`: Rename the current track
    - **Input**: `{name: String}`: New name of playlist
    - **Returns**: `{playlist: Playlist}`: An updated *Playlist* object.
- POST `/api/v2/playlists/:playlist_id/vote`: Vote on a track
    - **Input**: `{track_id: String, vote: Boolean}`: True to vote, false to unvote
    - **Returns**: `{playlist: Playlist}`: An updated *Playlist* object.   

### User Routes

- GET `/api/v2/users/:user_id`: Get User information
    - **Returns**: `user: User`: A *User* object.
- GET `/api/v2/users/:user_id/playlists:`: Get User playlists
    - **Returns**: `playlists: [Playlist]`: Arraw of *Playlist* Objects (without tracks).
- POST `/api/v2/users/friends/playlists:`: Get a list of a user's friends' playlists
    - **Input**: `{fbids: [String]}`: An array of Facebok Ids
    - **Returns**: `playlists: [Playlist]`: Arraw of *Playlist* Objects (without tracks).

Socket.io API
---
For clients and players subscribing to playlist updates

### Step 1: Authenticate to gain access

- on `auth`: Initialize authentication by passing API credentials
    - **Parameters**:
        - `client_token`: Client token from the REST API `/auth/login`
        - `user_id`: UserId of client from Step 1
    - **Emits**: `auth_response`: On result. No error is a success.
        - `error`: Sent only if there was an error
            - `message: String`: Description of problem

### Step 2: Register as a *Client* or *Player*

#### Register as a Client (Read-only updates)

- on `client_subscribe`: Subscribe to updates from a playlist
    - **Parameters**:
        - `playlist_id: String`: Playlist ID to subscribe to
    - **Emits**: `state_change`: On every playlist update until disconnect or unsubscribe
        - *State* object
- on `client_unsubscribe`: Stop receiving state change updates
    - **Parameters**: None
    - **Emits**: `client_unsubscribe_response`: Stops receiving `state_change`
        - `error`: Sent only if there was an error
            - `message: String`: Description of problem


#### Register as a Player (One per playlist)
This registers as a client inherently

- on `player_subscribe`: Subscribe to updates to play from a playlist
    - **Parameters**:
        - `playlist_id: String`: Playlist ID to play from
    - **Emits**: `state_change`: (every playlist update until disconnect or unsubscribe)
        - *State* object
    - **Emits**: `player_subscribe_response`: Result of subscription
        - `error`: Sent only if there was an error
            - `message: String`: Description of problem
- on `player_unsubscribe`: Stop acting as a player.
    - **Parameters**: None
    - **Emits**: `player_unsubscribe_response`: Stops receiving `state_change`
        - `error`: Sent only if there was an error
            - `message: String`: Description of problem

**As a Player, the following events are now available (and should be implemented):**

- on `track_finished`: The local track finished.
    - **Parameters**: None
    - **Broadcasts**: `state_change: State`: *State* object with new track.
- on `track_progress`: An ratio to update to the track's progression
    - **Parameters**: Send this no less frequently than once per second
        - `progress: Number`: Track progress (ms)
        - `duration: Number`: Track duration (ms)
- on `track_play_pause`: `{playing: true/false}`: The track was paused
    - **Parameters**:
        - `play: Boolean`: Play state to update the server with (true = playing)
    - **Broadcasts**: `state_change: State`: *State* object

Objects
=======
- *Playlist*: Playlist object that represents the entire playlist. Only used in the REST API.
    -  `_id` *String*: Internal ID. Used for Player authentication.
    -  `name` *String* Name of the playlist
    -  `current` *Track*: Currently playing Track
    -  `play` *Boolean*: `true` if playing, `false` otherwise
    -  `volume` *Number [0-100]*: Volume percentage
    -  `tracks` *[QueueItem]*:  Ordered items in the queue.
    -  `admin` Interal ID associate with the Adminisator user (creator)
    -  `admin_name` Display name of admin
    -  `date_created` *Number*: Date created (UNIX)
    -  `last_updated` *Number*: Date last updated (UNIX)
    -  `key` *String*: Non-unique short name for the playlist
    -  `distance` *Double*: If a location was sent to `/playlists/nearby`, this field exists and is the distance in meters to the playlist 

- *User*: User object that stores basic  information
    - `_id`: *String*: Internal ID. 
    - `name`: *String*: Full name
    - `facebook` (If user is connected with Facebook)
        - `id`: *String*: Fabook profile ID
    - `spotify` (If user is connected with Spotify)
        - `id`: *String*: Spotify profile ID

- *State*: The following fields are always sent:
    - `play` *Boolean*: `true` if playing, `false` otherwise
    - `track` *Track*: Currently playing track.
    - `queue` *[QueueItem]*: Ordered Array of *QueueItem*s.
    - `trigger` *String*: Mostly for debugging. Identifies what action caused this broadcast.

- *Track*: Simplified version of [Spotify's Track (full)](https://developer.spotify.com/web-api/object-model/#track-object-full).
    -  `votes` *Number*: Number of votes on the track
    -  `voters` *[User]*: Array of *User* objects, with *_id* and *name*parameters
    -  `name` *String*: Track name
    -  `id` *String*: Spotify ID
    -  `uri` *String*: Spotify URI
    -  `artists` *[Object]*: Array of Spotify's Artist objects
        -  `id` *String*: Spotify ID
        -  `name` *String*: Artist name
        -  `type` *String*: Artist type
        -  `uri` *String*: Spotify URI
        -  `href` *String*: Spotify URLs
        -  `external_urls` *[String]*: Extra URLs
    -  `album` *Object*: Spotify's Album object
        - `id` *String*: Spotify ID
        - `name` *String*: Album name
        - `uri` *String*: Spotify URI
        - `images` *[Object]*: Array of Images
            - `height`: *Number*: Image height
            - `width`: *Number*: Image width
            - `url`: *String*: Image URL
- *QueueItem*: Item in the queue
    - `_id`: *String*: Internal ID
    - `track`: *[Track]*: Array of *Track* objects

Remarks
-------
Because I only get to spend so much of my time on this, I am opening this project up to contribution.

If there is a feature you want to see, you have a problem with the app, or you have a problem with me, send me message or open an Issue, thanks!
