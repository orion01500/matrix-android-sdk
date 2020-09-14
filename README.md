[![Buildkite](https://badge.buildkite.com/c080f1de5e60c792fab17531dfda61ee0c3178ec616cfccb29.svg?branch=develop)](https://buildkite.com/matrix-dot-org/matrix-android-sdk)
[![Quality Gate](https://sonarcloud.io/api/project_badges/measure?project=matrix.android.sdk&metric=alert_status)](https://sonarcloud.io/dashboard?id=matrix.android.sdk)
[![Vulnerabilities](https://sonarcloud.io/api/project_badges/measure?project=matrix.android.sdk&metric=vulnerabilities)](https://sonarcloud.io/project/issues?id=matrix.android.sdk&resolved=false&types=VULNERABILITY)
[![Bugs](https://sonarcloud.io/api/project_badges/measure?project=matrix.android.sdk&metric=bugs)](https://sonarcloud.io/project/issues?id=matrix.android.sdk&resolved=false&types=BUG) 

Important Announcement
======================

This SDK is deprecated and the core team does not work anymore on it.

We strongly recommends that new projects use [the new Android Matrix SDK](https://github.com/matrix-org/matrix-android-sdk2).

We can provide best effort support for existing projects that are still using this SDK though.

matrix-android-sdk
==================
The [Matrix] SDK for Android wraps the Matrix REST API calls in asynchronous Java methods and provides basic structures for storing and handling data.

It is an Android Studio (gradle) project containing the SDK module.
https://github.com/vector-im/riot-android is the sample app which uses this SDK.

Overview
--------
The Matrix APIs are split into several categories (see [matrix api]).
Basic usage is:

 1. Log in or register to a home server -> get the user's credentials
 2. Start a session with the credentials
 3. Start listening to the event stream
 3. Make matrix API calls

Bugs / Feature Requests
-----------------------
Think you've found a bug? Please check if an issue
does not exist yet, then, if not, open an issue on this Github repo. If an issue already
exists, feel free to upvote for it.

Contributing
------------
Want to fix a bug or add a new feature? Check if there is an corresponding opened issue.
If no one is actively working on the issue, then please fork
the ``develop`` branch when writing your fix, and open a pull request when you're
ready. Do not base your pull requests off ``master``.

Logging in
----------
To log in, use an instance of the login API client.

```java
HomeServerConnectionConfig hsConfig = new HomeServerConnectionConfig.Builder()
    .withHomeServerUri(Uri.parse("https://matrix.org"))
    .build();
new LoginRestClient(hsConfig).loginWithUser(username, password, new SimpleApiCallback<Credentials>());
```

If successful, the callback will provide the user credentials to use from then on.

Starting the matrix session
---------------------------
The session represents one user's session with a particular home server. There can potentially be multiple sessions for handling multiple accounts.

```java
MXSession session = new MXSession.Builder(hsConfig, new MXDataHandler(store, credentials), getApplicationContext())
    .build();
```

sets up a session for interacting with the home server.

The session gives access to the different APIs through the REST clients:

```session.getEventsApiClient()``` for the events API

```session.getProfileApiClient()``` for the profile API

```session.getPresenceApiClient()``` for the presence API

```session.getRoomsApiClient()``` for the rooms API

For the complete list of methods, please refer to the [Javadoc].

**Example**
Getting the list of members of a chat room would look something like this:

```java
session.getRoomsApiClient().getRoomMembers(<roomId>, callback);
```

The same session object should be used for each request. This may require use
of a singleton, see the ```Matrix``` singleton in the ```app``` module for an
example.

The event stream
----------------
One important part of any Matrix-enabled app will be listening to the event stream, the live flow of events (messages, state changes, etc.).
This is done by using:

```java
session.startEventStream();
```

This starts the events thread and sets it to send events to a default listener.
It may be useful to use this in conjunction with an Android ```Service``` to
control whether the event stream is running in the background or not.

The data handler
----------------
The data handler provides a layer to help manage data from the events stream. While it is possible to write an app with no
data handler and manually make API calls, using one is highly recommended for most uses. The data handler :

 * Handles events from the events stream
 * Stores the data in its storage layer
 * Provides the means for an app to get callbacks for events
 * Provides and maintains room objects for room-specific operations (getting messages, joining, kicking, inviting, etc.)

```java
MXDataHandler dataHandler = new MXDataHandler(new MXMemoryStore());
```

creates a data handler with the default in-memory storage implementation.

### Registering a listener
To be informed of events, the app needs to implement an event listener.

```java
session.getDataHandler().addListener(eventListener);
```

This listener should subclass ```MXEventListener``` and override the methods as needed:

```onPresenceUpdate(event, user) ```
Triggered when a user's presence has been updated.

```onLiveEvent(event, roomState) ```
Triggered when a live event has come down the event stream.

```onBackEvent(event, roomState) ```
Triggered when an old event (from history), or back event, has been returned after a request for more history.

```onInitialSyncComplete() ```
Triggered when the initial sync process has completed. The initial sync is the first call the event stream makes
to initialize the state of all known rooms, users, etc.

### The Room object
The Room object provides methods to interact with a room (getting message history, joining, etc).

```java
Room room = session.getDataHandler().getRoom(roomId);
```

gets (or creates) the room object associated with the given room ID.

#### Room state
The RoomState object represents the room's state at a certain point in time: its name, topic, visibility (public/private), members, etc.
onLiveEvent and onBackEvent callbacks (see Registering a listener) return the event, but also the state of the room at the time of the event to
serve as context for building the display (e.g. the user's display name at the time of their message). The state provided is the one before
processing the event, if the event happens to change the state of the room.

#### Room history
When entering a room, an app usually wants to display the last messages. This is done by calling

```java
room.requestHistory();
```

The events are then returned through the ```onBackEvent(event, roomState)``` callback in reverse order (most recent first).

This does not trigger all of the room's history to be returned but only about 15 messages. Calling ```requestHistory()``` again will then
retrieve the next (earlier) 15 or so, and so on. To start requesting history from the current live state (e.g. when opening or reopening a room),

```java
room.initHistory();
```

must be called prior to the history requests.

The content manager
-------------------
Matrix home servers provide a content API for the downloading and uploading of content (images, videos, files, etc.).
The content manager provides the wrapper around that API.

```java
session.getContentManager();
```

retrieves the content manager associated with the given session.

### Downloading content
Content hosted by a home server is identified (in events, avatar URLs, etc.) by a URI with a mxc scheme (mxc://matrix.org/xxxx for example).
To obtain the underlying HTTP URI for retrieving the content, use

```java
contentManager.getDownloadableUrl(contentUrl);
```

where contentUrl is the mxc:// content URL.

For images, an additional method exists for returning thumbnails instead of full-sized images:

```java
contentManager.getDownloadableThumbnailUrl(contentUrl, width, height, method);
```

which allows you to request a specific width, height, and scale method (between scale and crop).

### Uploading content
To upload content from a file, use

```java
contentManager.uploadContent(filePath, callback);
```

specifying the file path and a callback method which will return an object on completion containing the mxc-style URI where the uploaded
content can now be found.

**See the sample app and Javadoc for more details.**

References
----------
- [Matrix home page](https://matrix.org)
- [Matrix api documentation](https://matrix.org/docs/spec/client_server/latest.html)
- [Matrix api](https://matrix.org/docs/api/client-server/)

License
-------
Apache 2.0

