
# API sketch in Golang {#apisketch}

The following sketch is a snapshot of an API currently under development in Go,
available at https://github.com/mami-project/postsocket. The details of the API
are still under development; once the API definition stabilizes, this will be
expanded into prose in a future revision of this draft.

~~~~~~~~
// The interface to path information is TBD
type Path interface{}

// An association encapsulates an endpoint pair and the set of paths between them.
type Association interface {
    Local() Local
    Remote() Remote
    Paths() []Path
}

// A message together with with metadata needed to send it
type OutMessage struct {
    // The niceness of this message. 0 is highest priority.
    Niceness uint
    // The lifetime of this message. After this duration, the message may expire.
    Lifetime time.Duration
    // Pointers to messages that must be sent before this one.
    Antecedent []*OutMessage
    // True if the message is safe to send such that it may be received multiple times (i.e. for 0-RTT).
    Idempotent bool
}

// A message received from a stream
type InMessage struct {

}

// A Carrier is a transport protocol stack-independent interface for sending and
// receiving messages between an application and a remote endpoint; it is roughly
// analogous to a socket in the present sockets API.
type Carrier interface {
    // Send a message on this Carrier. The optional onSent function will be
    // called when the protocol stack instance has sent the message. The
    // optional onAcked function will be called when the receiver has
    // acknowledged the message. The optional onExpired function will be
    // called if the message's lifetime expired before the message coult be
    // sent. If the Carrier is not active, attempt to activate the Carrier
    // before sending.
    SendMessage(content []byte, msg *OutMessage, complete bool, onSent func(), onAcked func(), onExpired func()) error

	// Send a byte array on this Carrier as a message with default metadata
	// and no notifications.
	Send(content []byte) error

    // Signal that the application is ready to receive a single atomic message via
    // the given callback.
    // This will deliver a single message, or an error if the carrier failed before
	// a message could be delivered.
    ReceiveMessage(receive func(content []byte, InMessage, error) bool) error

	// Signal that the application is ready to receive a complete or partial message
	// via the given callback. The minimum incomplete bytes specifies a length of the
	// message content that is expected before delivery, and the maximum bytes specifies
	// the number of content bytes that can be received at once. If the message is complete
	// with fewer than the minimum incomplete bytes, the message will be delivered.
	// This will deliver a single callback, with the available content, the message associated
	// with the content, whether or not the content is complete, and an error.
	Receive(minimumIncompleteBytes uint, maximumBytes uint, receive func(content []byte, InMessage, complete bool, error) bool) error

    // Retrieve the Association over which this Carrier is running.
    Association() *Association

    // Retrieve the active Transients over which this carrier is running, if active.
    Transients() []Transient

    // Determine whether the Carrier is currently active
    IsActive() bool

    // Ensure that the Carrier is active and ready to send and receive messages.
    // Attempts to bring up at least one Transient.
    Activate(isActive func()) error

    // Terminate the Carrier
    Close()

    // Attempt to fork a new Carrier for communicating with the same Remote
    Fork() (Carrier, error)

    // Signal that an application is ready to accept forks via a given callback.
    // Forked carriers will be given to the callback until it returns false or
    // until the Carrier is closed.
    Accept(accept func(Carrier) bool) error
}

// Initiate a Carrier from a given Local to a given Remote. Returns a new
// Carrier, which may be bound to an existing or a new Association. The
// initiated Carrier is not yet active.
func Initiate(local Local, remote Remote) (Carrier, error)

type Listener interface {
    // Signal that an application is ready to accept forks via a given callback.
    // Accept will terminate when the callback returns false, or until the
    // Listener is closed.
    Accept(accept func(Carrier) bool) error

    // Terminate this Listener
    Close()
}

// Create a Listener on a given Local which will pass new Carriers to the
// given channel until that channel is closed.
func Listen(local Local) (Listener, error)

// A Source is a unidirectional, send-only Carrier.
type Source interface {
    // Send a byte array on this Source as a message with default metadata
    // and no notifications.
    Send(buf []byte) error

    // Send a message on this Source. The optional onSent function will be
    // called when the protocol stack instance has sent the message. The
    // optional onAcked function will be called when the receiver has
    // acknowledged the message. The optional onExpired function will be
    // called if the message's lifetime expired before the message coult be
    // sent. If the Source is not active, attempt to activate the Source
    // before sending.
    Sendmsg(msg *OutMessage, onSent func(), onAcked func(), onExpired func()) error

    // Retrieve the Association over which this Source is running.
    Association() *Association

    // Determine whether the Source is currently active
    IsActive() bool

    // Ensure that the Source is active and ready to send messages.
    // Attempts to bring up at least one Transient.
    Activate() error

    // Terminate the Source
    Close()
}

// Initiate a Source from a given Local to a given Remote. Returns a new
// Source, which may be bound to an existing or a new Association. The
// initiated Source is not yet active.
func NewSource(local Local, remote Remote) (Source, error)

// A Sink is a unidirectional, receive-only Carrier, bound only to a local.
type Sink interface {
    // Signal that an application is ready to receive messages via a given callback.
    // Messages will be given to the callback until it returns false, or until the
    // Sink is closed.
    Ready(receive func(InMessage) bool) error

    // Retrieve the Association over which this Sink is running.
    Association() *Association

    // Terminate the Sink
    Close()
}

// Initiate a Sink on a given Local. Returns a new
// Sink, which may be bound to an existing or a new Association.
func NewSink(local Local) (Sink, error)

// Initiate a Responder on a given Local. For each incoming Message, calls the
// respond function with the Message and a Sink to send replies to. Calls the
// Responder until it returns False, then terminates
func Respond(local Local, respond func(msg InMessage, reply Sink) bool) error

// A local identity
type Local struct {
    // A string identifying an interface or set of interfaces to accept messages and new carriers on.
    Interface string
    // A transport layer port
    Port int
    // A set of zero or more end entity certificates, together with private
    // keys, to identify this application with.
    Certificates []tls.Certificate
}

// Encapsulate a remote identity. Since the contents of a Remote are highly
// dependent on its level of resolution; some examples are below.
type Remote interface {
    // Resolve this Remote Identity to a
    Resolve() ([]RemoteIdentity, error)
    // Returns True if the Remote is completely resolved; i.e., cannot be resol
    Complete() bool
}

// Remote consisting of a URL
type URLRemote struct {
    URL string
}

// Remote encapsulating a name and port number
type NamedEndpointRemote struct {
    Hostname string
    Port     int
}

// Remote encapsulating an IP address and port number
type IPEndpointRemote struct {
    Address net.IP
    Port    int
}

// Remote encapsulating an IP address and port number, and a set of presented certificates
type IPEndpointCertRemote struct {
    Address      net.IP
    Port         int
    Certificates []tls.Certificate
}
~~~~~~~~