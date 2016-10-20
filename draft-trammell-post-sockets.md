---
title: Post Sockets, An Abstract Programming Interface for the Transport Layer
abbrev: Post Sockets
docname: draft-trammell-post-sockets-00
date: 
category: info

ipr: trust200902
area: Transport
workgroup: TAPS Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: B. Trammell
    name: Brian Trammell
    org: ETH Zurich
    email: ietf@trammell.ch
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland
  -
    ins: C. Perkins
    name: Colin Perkins 
    org: University of Glasgow
    street: School of Computing Science
    city: Glasgow  G12 8QQ
    country: United Kingdom
    email: csp@cperkins.net
  -
    ins: T. Pauly
    name: Tommy Pauly
    org: Apple Inc.
    street: 1 Infinite Loop
    city: Cupertino, California 95014
    country: United States of America
  -
    ins: M. Kuehlewind
    name: Mirja Kuehlewind
    org: ETH Zurich
    email: mirja.kuehlewind@tik.ee.ethz.ch
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland

normative:

informative:
    RFC0793:
    RFC4960:
    RFC6824:
    RFC7258:
    I-D.trammell-plus-abstract-mech:
    I-D.trammell-plus-statefulness:
    I-D.hamilton-quic-transport-protocol:
    minimalt:
    minion:

--- abstract

[EDITOR'S NOTE: write me]

--- middle

# Introduction

The BSD Unix Sockets API's SOCK_STREAM abstraction, by bringing network
sockets into the UNIX programming model, allowing anyone who knew how to write
programs that dealt with sequential-access files to also write network
applications, was a revolution in simplicity. It would not be an overstatement
to say that this simple API is the reason the Internet won the protocol wars
of the 1980s. SOCK_STREAM is tied to the Transmission Control Protocol (TCP),
specified in 1981 {{RFC0793}}. TCP has scaled remarkably well over the past
three and a half decades, but its total ubiquity has hidden an uncomfortable
fact: the network is not really a file, and stream abstractions are too
simplistic for many modern application programming models.

In the meantime, the nature of Internet access is evolving. Many end-user
devices are connected to the Internet via multiple interfaces, which suggests
it is time to promote the "path" by which a host is connected to a first-order
object; we call this "path primacy". 

Implicit multipath communication is available for these multihomed nodes in
the present Internet architecture with the Multipath TCP extension (MPTCP)
{{RFC6824}}. Since many multihomed nodes are connected to the Internet through
access paths with widely different properties with respect to bandwidth,
latency and cost, adding explicit path control to MPTCP's API would be useful
in many situations. Path primacy for cooperation with path elements is also
useful in single-homed architectures, such as that proposed by the Path Layer
UDP Substrate (PLUS) effort (see {{I-D.trammell-plus-statefulness}} and 
{{I-D.trammell-plus-abstract-mech}}).

Another trend straining the traditional layering of the transport stack
associated with the SOCK_STREAM interface is the widespread interest in
ubiquitous deployment of encryption to guarantee confidentiality,
authenticity, and integrity, in the face of pervasive surveillance
{{RFC7258}}. Layering the most widely deployed encryption technology,
Transport Layer Security (TLS), strictly atop TCP (i.e., via a TLS library
such as OpenSSL that uses the sockets API) requires the encryption-layer
handshake to happen after the transport-layer handshake, which increases
connection setup latency on the order of one or two round-trip times, an
unacceptable delay for many applications. Integrating cryptographic state
setup and maintenance into the path abstraction naturally complements efforts
in new protocols (e.g. QUIC {{I-D.hamilton-quic-transport-protocol}}) to
mitigate this strict layering.

From these three starting points -- more flexible abstraction, path primacy,
and encryption by default -- we define the Post-Socket Application Programming
Interface (API), described in detail in this work. Post is designed to be
language, transport protocol, and architecture independent, allowing
applications to be written to a common abstract interface, easily ported among
different platforms, and used even in environments where transport protocol
selection may be done dynamically, as proposed in the IETF's Transport Services
wotking group (see https://datatracker.ietf.org/wg/taps/charter).

Post replaces the traditional SOCK_STREAM abstraction with an Object
abstraction. Objects can be small (e.g. messages in message-oriented
protocols) or large (e.g. an HTTP response containing header and body). It
replaces the notions of a socket address and connected socket with an
Association with a remote endpoint via set of Paths. Implementation and wire
format for transport protocol(s) implementing the Post API are explicitly out
of scope for this work; these abstractions need not map directly to
implementation-level concepts, and indeed with various amounts of shimming and
glue could be implemented with varying success atop any sufficiently flexible
transport protocol.

For compatibility with situations where only strictly stream-oriented
transport protocols are available, applications with data streams that cannot
be easily split into Objects at the sender, and and for easy porting of the
great deal of existing stream-oriented application code to Post, Post also
provides a SOCK_STREAM compatible abstraction, unimaginatively named Stream.

The key features of Post as compared with the existing sockets API are:

- Explicit Object orientation, with framing and atomicity guarantees for
  Object transmission.

- Asynchronous reception, allowing all receiver-side interactions to be 
  event-driven. 

- Explicit support for multipath transport protocols and network architectures. 

- Long-lived Associations, whose lifetimes may not be bound to underlying \
  transport connections. This allows associations to cache state and 
  cryptographic key material to enable fast (0-rtt) resumption of communication.

This work is the synthesis of many years of Internet transport protocol
research and development. It is heavily inspired by concepts from the Stream
Control Transmission Protocol (SCTP) {{RFC4960}}, TCP Minion [EDITOR'S NOTE:
cite], MinimaLT [EDITOR'S NOTE: cite, and various bulk object transports.
While much of the work for building parts of the protocols needed to implement
Post are already ongoing in other IETF working groups (e.g. TAPS, MPTCP, QUIC,
TLS), we argue that an abstract programming interface unifying access all
these efforts is necessary to fully exploit their potential.
