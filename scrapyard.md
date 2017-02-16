
# Implementation Notes 

[EDITOR'S NOTE: most of this goes into the FIT paper instead?]

## From Policies to Paths

[EDITOR'S NOTE: write a section on the pathfinder]

## Supporting Stack Agility

[EDITOR'S NOTE: write a section on protocol stack instances]

## Playing with PostSockets: Golang implementation

[EDITOR'S NOTE: short howto on the Go implementation]

# Scrapyard

[EDITOR'S NOTE: Here are a few paragraphs that don't fit in the draft but that haven't been deleted yet, in case they are useful after the reorg of the doc is complete]


Messages larger
than the MTU on the Path on which they are sent will be segmented into
multiple frames. Multiple Messages that will fit into a single frame may be
concatenated into one frame. There is no preference for transmitting the
multiple frames for a given Message in any particular order, or by default,
that Messages will be delivered in the order sent by the application. 

When an application has hard semantic requirements that all the frames of a
given Message be sent down a given Path or Paths, these hard constraints can
also be expressed by the application.



## Carrier

A carrier is a transport-independent 

[EDITOR'S NOTE: terminology question: is our "Stream" really a "Carrier" or a "Channel"? I like "carrier"; it doesn't collide in Layer 3 or 4 terminology and fits nicely with "Post" (i.e. "letter carrier"). "Bytestream" then turns back into "Stream". Thoughts?]

Messages are sent and received over Streams, which represent a networks'  Messages sent on a Stream will be
received at the other end, atomically, but not necessarily reliably or in
order. An application may use one or more Streams to communicate with a remote
application; the semantics of which Messages belong on which Streams are, in
this case, application-specific.



## Listener

[EDITOR'S NOTE: possibly rewrite me, encapsulates any initial establishment
and cryptographic state setup to create an Association from a Local and a
not-previously-known Remote.]

In many applications, there is a distinction between the active opener (or
connection initiator, often a client), and the passive opener (often a
server). A Listener represents an endpoint's willingness to start
Associations in this passive opener/server role. It is, in essence, a
one-sided, Path-less Association from which fully-formed Associations can
be created.

Listeners work very much like sockets on which the listen(2) call has
been called in the SOCK_STREAM API.

## Association

An Association is... [EDITOR'S NOTE: work pointer]

Note that, in contrast to current SOCK_STREAM sockets, Associations are meant
to be relatively long-lived. 

Transients may be dynamically added or removed from an association, as well, as
connectivity between the endpoints changes. An Association may exist even if
there are no currently active paths available between them. Cryptographic
identifiers and state for endpoints may also be added and removed as necessary
due to certificate lifetime, key rollover, revocation, and so on.

## Path

A Path represents a local and remote endpoint address, an optional set of
intermediary path elements between the local and remote endpoint addresses,
and a set of properties associated with the path.


## Pathfinder

[EDITOR'S NOTE: write me, encapsulates any re-establishment and rendezvous
protocol. might be equivalent to connect(), might also need to use something
like ICE. Connection racing also fits behind the Pathfinder. Notes from Seoul:
add a Pathfinder abstraction for rendezvous, especially in peer-to-peer
situations. A Pathfinder encapsulates a method for reconnecting to a specific
remote (e.g., underlying transport connect() call in the case of client-
server, something like ICE in peer-to-peer). Add a pathfind() call to ensure
an association has paths; this *must* be called before Messages can be sent.
send() should *not* bring a dormant path up by default, it should fail. ]



listener, association, and
stream), four entry points (listen(), associate(), send(), and
open_stream()) and a set of callbacks for handling events at each endpoint.
The details are given in the subsections below.

## Active Association Creation

Associations can be created two ways: actively by a connection initiator, and passively by a Listener that accepts a connection. Connection initiation uses the associate() entry point:

association = associate(local, remote, receive_handler)

where: 

- local: a resolved Local (see {{address-resolution}}) describing the local identity and interface(s) to use
- remote: a resolved Remote (see {{address-resolution}}) to associate with
- receive_handler: a callback to be invoked when new objects are received; see  {{receiving-objects}} 

The returned association has the following additional properties:

- paths: a set of Paths that the Association can currently use to transport Objects. When the underlying transport connection is closed, this set will be empty. For explicitly multipath architectures and transports, this set may change dynamically during the lifetime of an association, even while it remains connected.

Since the existence of an association does not necessarily imply
current connection state at both ends of the Association, these objects are
durable, and can be cached, migrated, and restored, as long as the mappings to
their event handlers are stable. An attempts to send an object or open a
stream on a dormant, previously actively-opened association will cause the
underlying transport connection state to be resumed.

## Listener and Passive Association Creation

In order to accept new Association requests from clients, a server must create a Listener object, using the listen() entry point:

listener = listen(local, accept_handler)

where: 

- local: resolved Local (see {{address-resolution}}) describing the local identity and interface(s) to use for Associations created by this listener.
- accept_handler: callback to be invoked each time an association is requested by a remote, to finalize setting the association up. Platforms may provide a default here for supporting synchronous association request handling via an object queue.

The accept_handler has the following prototype:

accepted = accept_handler(listener, local, remote)

where:

- local: a resolved Local on which the association request was received.
- remote: a resolved Remote from which the association request was received. 
- accepted: flag, true if the handler decided to accept the request, false otherwise.

The accept_handler() calls the accept() entry point to finally create the association:

association = accept(listener, local, remote, receive_handler)

## Sending Objects 

Objects are sent using the send() entry point:

send(association, bytes, [lifetime], [niceness], [oid], [antecedent\_oids], [paths])}

where:

- association: the association to send the object on
- bytes: sequence of bytes making up the object. For platforms without bounded byte arrays, this may be implemented as a pointer and a length.
- lifetime: lifetime of the object in milliseconds. This parameter is optional and defaults to infinity (for fully reliable object transport).
- niceness: the object's niceness class. This parameter is optional and defaults to zero (for lowest niceness / highest priority)
- oid: opaque identifier for an object, assigned by the application. Used to refer to this object as a subsequently sent object's antecedent, or in an ack or expired handler (see {{events}}). Optional, defaults to null.
- antecedent_oids: set of object identifiers on which this object depends and which must be sent before this object. Optional, defaults to empty, meaning this object has no antecedent constraints.
- paths: set of paths, as a subset of those available to the association, to explicitly use for this object. Optional, defaults to empty, meaning all paths are acceptable.

Calls to send are non-blocking; a synchronous send which blocks on remote
acknowledgment or expiry of an object can be implemented by a call to send()
followed by a wait on the ack or expired events (see {{events}}).

## Receiving Objects

An application receives objects via its receive_handler callback, registered
at association creation time. This callback has the following prototype:

receive_handler(association, bytes)

where:
- association: the association the object was received from.
- bytes: the sequence of bytes making up the object.

For ease of porting synchronous datagram applications, implementations may
make a default receive handler available, which allows messages to be
synchronously polled from a per-association object queue. If this default is
available, the entry point for the polling call is:

bytes = receive_next(association)

## Creating and Destroying Streams

A stream may be created on an association via the open_stream() entry point:

stream = open_stream(association, [sid])

where:

- association: the association to open the stream on
- sid: opaque identifier for a stream. For transport protocols which do not support multiple streaming, this argument has no effect.

A stream with a given sid must be opened by both sides before it can be used.

The stream object returned should act like a file descriptor or bidirectional
I/O object, according to the conventions of the platform implementing Post.

## Events

Message reception is a specific case of an event that can occur on an
association. Other events are also available, and the application can register
event handlers for each of these. Event handlers are registered via the
handle() entry point:

handle(association, event, handler) or

handle(oid, event, handler)

where

- association: the association to register a handler on, or
- oid: the object identifier to register a handler on
- event: an identifier of the event to register a handler on
- handler: a callback to be invoked when the event occurs, or null if the event should be ignored.

The following events are supported; every event handler takes the association
on which it is registered as well as any additional arguments listed:

- receive (bytes): an object has been received
- path_up (path): a path is newly available
- path_down (path): a path is no longer available
- dormant: no more paths are available, the association is now dormant, and the connection will need to be resumed if further objects are to be sent
- ack (oid): an object was successfully received by the remote
- expired (oid): an object expired before being sent to the remote

Handlers for the ack and expired events can be registered on an association
(in which case they are called for all objects sent on the association) or on
an oid (in which case they are only called for the oid).

## Paths and Path Properties

As defined in {{path}}, the properties of a path include both the addresses of
elements along the path as well as measurement-derived latency and capacity
characteristics. The path_up and path_down events provide access to
information about the paths available via the path argument to the event
handler. This argument encapsulates these properties in a platform and
transport-specific way, depending on the availability of information about the
path.

## Address Resolution

Address resolution turns the name of a Remote into a resolved Remote object,
which encapsulates all the information needed to connect (address, certificate
parameters, cached cryptographic state, etc.); and an interface identifier on
a local system to information needed to connect. Remote and local resolvers
have the following entry points:

remote = resolve(endpoint_name, configuration)

local = resolve_local(endpoint_name, configuration)

where:

- endpoint_name: a name identifying the remote or local endpoint, including port
- configuration: a platform-specific configuration object for configuring certificates, name resolution contexts, cached cryptographic state, etc. 

## Configuration

[EDITOR'S NOTE: add entry points for configurability, and make configuability
consistent. system level and application level configuration. probably wrap
all this in a configuration object.]