# DJFED: A proposal for a federated online DJing platform

Request for comments.
J. Woods, 2022
MIT License
Rev. 2022-06-18

## Introduction

In the early 2010s, turntable.fm provided a whimsical, novel idea:  users would join chat rooms with an added “DJing” feature, joining a queue where they took turns to play songs.  Each user could vote on whether or not they liked the song, and enough thumbs-down votes would cause the song to be skipped.

turntable.fm attempted an approach which involved direct licensing with record companies; this proved, at the time, to not be financially viable.  The emergence of turntable also coincided with an increase in the popularity of online music streaming services, which have deeply changed the relationship between individuals and their music.  Now, a surprising number of turntable-inspired online DJing platforms exist, the vast majority of which are powered by a specific music streaming service, like YouTube or Spotify.

This type of platform is deeply important for creating communities.  Humans have a deep, personal relationship with music, so it’s no wonder that individual groups quickly spring up around this concept when given the opportunity to.  These communities transcend the particular DJing platform they’ve chosen:  time and time again, they’ve shown the ability to move between platforms when necessary.  One notable example is Indie Discotheque, a group centered around a very specific subgenre of music, who created their own DJing platform after turntable.fm was initially closed.

Over and over, internet users have developed a vested interest in closed platforms, which have been forced to close their doors, or changed direction in ways which alienate communities.  Novel approaches to communication protocols have sprung up to combat this:  namely, federation combined with open source software.  Communities should own their own platforms.  Communities should control their own “home bases,” without needing to be concerned about contingency planning or unexpected closures.

This document proposes a common architecture which facilitates the creation of individual online DJing rooms which can elect to share information with each other.  The hope is that many open-source implementations of DJFED will be created, allowing for a music-loving user to find their own home online with minimal friction.

## Document status

This RFC is non-normative and not on a standards track.  It is not currently adopted by any standards-governing body.  Please consider this document flexible, and in “early stages.”

Contributions to this document are highly encouraged.  Right now, the primary means for submitting feedback is to submit a GitHub issue, pull request, or send an email to djfed@joewoods.dev.

## Document scope

This proposal provides a higher-level framework for the creation and federation of online DJing communities.  It does not cover:

* Specifics regarding the implementation of playback mechanisms, merely the API to engage them,
* Concrete suggestions for the UX of a DJFED implementation, or
* Recommendations for hosting, deliverability, or uptime.

Monetization can be trivially implemented within the confines of this specification, but monetization of a specific DJFED instance should only serve to ensure the sustainability of a community. 

## Other considerations

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all capitals, as shown here.

All notation or endpoints described in this document MUST conform to the Internet JSON (I-JSON) format as defined in [RFC7493].

This document is heavily inspired by the ActivityPub W3C Recommendation, but the implementation has subtle differences.  Knowledge of ActivityPub is not required to implement this RFC. 

DJFED uses HTTP [@!RFC7230] to expose its resources. All HTTP requests MUST use the https:// scheme (HTTP over TLS [@!RFC2818]).

## Implementation

1. Server information
1.1. Well-known URIs
1.2. Retrieving server state
2. User registration, authentication, and settings
2.1. User registration
2.2. User authentication 
2.3. Admin actions
3. Users
4. Server federation
4.1. Federated subscription
4.2. Federated messaging
4.3. Account synchronization between servers
5. Events
5.1. State updated
5.2. Music-related events
5.3. User interaction
5.4. Server messages
5.5. Cross-server messages
5.6. Latency, user timeouts
6. Integrating music services
6.1. NowPlaying object
7. Bots
8. Anti-abuse
8.1. Rate-limiting

### 1. Server information

#### 1.1. Well-known URIs

Each DJFED server MUST provide a well-known URI as described in RFC8615 with the prefix `djfed`.

This endpoint provides URLs for the following, as a key-value JSON object:

* **auth**: initiating user registration and authentication as described in 2.1, 2.2.; for registering 
* **event**: authenticated endpoint to receive events from the server as described in sections 4, 5, and 6
* **state**: for receiving information about the state of a server as described in 1.2.  May be authenticated or authenticated; may provide different information depending on the authentication status of the user.
* **inbox**: authenticated endpoint for federation as described in section 3.

#### 1.2. Retrieving server state

The **state** endpoint provides information about the server.  This endpoint should always return a JSON object, but any number of these properties MAY be returned dependent on whether or not the user is authenticated.  If the user is authenticated, every property MUST be returned.

* **contact**: an email address to contact a representative of the server
* **messages**: recent Messages from users
* **description**: a plaintext description of the server
* **genre**: a short string describing the music genre that this instance plays
* **playing**: a NowPlaying object
* **service**: a string describing the streaming service used by this instance
* **online**: an array of online User objects

This object may be extended; if the client receives keys that it does not understand, they MUST be ignored.

This information can be used by aggregators which list rooms for prospective users to join.

### 2. User registration, authentication, and settings

This uses the **auth** endpoint as described in section 1.1.

#### 2.1. User registration

Posting to `/register` path creates a registration attempt.  

The body of the registration attempt is:

* **email**
* **password**
* **username**: initial username to use in the chat; this may be changed at any time.
* **homeserver**: Optional URI. If provided, mark this user as a guest from another server, and keep their information up to date as provided by that server.

Future implementations may wish to extend the registration object with, for example, a “reason” field, if an instance is invite-only.

A successful registration request MUST return a 200 response with a JSON body:

* **active**: Boolean.  Indicates whether or not the user’s account was made immediately active; if this is false, the user must be approved by the instance administrators.

#### 2.2. User authentication

DJFED uses Authorization headers with “Basic” authentication as described in RFC 7235.

Posting to `/status` will inform whether or not the user is currently logged in.

#### 2.3. Admin actions

Administrators may elect to mark users as unable to log in.  Instances may elect to implement any degree of privacy or openness that they desire, and may elect to change their status over time based on individual need.

### 3. Users

`User` objects are sent and received by the server.  These contain at least the following keys and values:

* **id**: a unique identifier for the user which is not presented by the user interface.
* **username**
* **avatar** (optional) an image representing the user.

### 4. Server federation

Discoverability and cross-communication between other communities is vital for both community growth and entertainment purposes.  DJFED instances can be either completely closed, without the ability to communicate with other servers, or completely open, allowing users to freely migrate and send messages between servers.

#### 4.1. Federated subscription

Servers may post to the **auth** endpoint to receive a Basic auth token through which to post messages to the **inbox** endpoint.

#### 4.2. Federated messaging

Successfully authenticated servers may post plaintext messages to each other’s **inbox** endpoints, which may be treated as system-wide server messages and passed on to online users.

#### 4.3. Account synchronization between servers

Servers may post user information to each other via the **inbox**.  This allows users to register accounts with servers as guests, marking them as “from” another community, and using their settings from that server.

### 5. Events

A user is considered logged in if they have an active connection to the **event** endpoint.

The **event** endpoint may be implemented in any real-time communication protocol, but, at time of writing, the use of WebSockets is highly encouraged.

This section describes the events that a DJFED instance should provide, but each instance MAY extend this protocol.  A client MUST ignore any messages it does not understand.

#### 5.1. State updated

If a user updates information about their account, e.g. their username or an avatar, or the server updates information about itself, e.g. its description, the server should send a `{stateUpdated: true}` message.  This instructs the client to re-request the **state** information of the server and update the interface as necessary.

#### 5.2. Music-related events

Though the NowPlaying object (section 5.1.) should be returned in the state, the server should also send updates to this object via the event connection.  This is a JSON object, where the key is `nowPlaying`, and the value is the new NowPlaying object.

#### 5.3. User interaction

Users may send information to the server via the event endpoint.  A user-sent event must be a JSON object with exactly one key.  These include:

* **vote**: String representing a new vote from a user on the currently playing track.  Ignored by the server if invalid
* **chat**: A chat message to be sent to other users
* **leave**: Boolean; user is quitting their session.

The server should send vote and chat messages from users to other users.  The server will send a message which is a JSON object with a single key describing the action, but the value is a tuple (Array) of \[UserId, message\], where UserId is the id of the User, and the message is the value as provided above.

#### 5.6. Latency, user timeouts

May reconnect by querying the state of the server and making a new connection to the **event** endpoint

## License

(MIT license goes here)