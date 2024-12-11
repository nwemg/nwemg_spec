The Design and Specification of the Protocol
============================================

NWEmgis a protocol designed to make offloading EMG processing from
intelligent prostheses to the edge cloud simple, efficient and reliable.
The protocol was designed to achieve the best possible performance with
the lowest possible overhead both in terms of computations and network
resources.

While the guarantees provided by the URLLC 5G communications would make
the protocol seem extremely robust, the protocol does not require the
use of URLLC, and was designed to operate in all common modern wireless
networking environments (5G/Wi-Fi6), and a fallback mechanism is
expected to be available locally. Most classifier algorithms are
stateless, therefore packet (and as a direct result message) loss is
tolerable to a certain extent. The potential reordering of messages
should be accounted for by the implementation.

The protocol exchanges binary data with messages of predefined formats
specified in (rather than using a semi-structured data interchange
format like JSON or BSON). The byte ordering of data types used by the
protocol is little-endian (LE) regardless of the platform.

An implementation was created for the protocol that adheres to the
specification of the protocol and is presented in .

Inspiration and Alternatives {#sec:inspalt}
----------------------------

Before designing a new protocol, a natural step was to look into
existing protocols that could be applied to EMG data transmission.

MQTT [^1] and CoAP [@rfc7252] were examined as alternative application
layer protocols. MQTT is used in multiple papers in the field of MEC
offloading, but it uses transmission control protocol (TCP) which is
impractical due to the requirement , as in real-time and near real-time
use cases it might suffer extra latency due to head-of-line
blocking[@holb]. In the case of CoAP, the extra functionality it would
provide is not worth the overhead compared to native UDP.

Stream Control Transmission Protocol (SCTP) [@rfc2960], a
message-oriented, but connection-based transport layer protocol provides
an ideal subset of features of both UDP (no HOLB, optional reliability)
and TCP (connection-oriented). My reason for not choosing SCTP was that
most home routers which need to perform NAT in case of IPv4 traffic
still do not support SCTP (this unfortunately includes the customer
premise equipments (CPEs) used with the private 5G network used in ),
and while SCTP over UDP would have been an option, with some additional
work UDP alone can meet the requirements of the protocol.

The takeaway was that even though solutions exist both for IoT
communications (CoAP and MQTT) and real-time streaming (RTP) none of
these target low-bandwidth and low-latency use cases such as the
transmission of EMG data, and a protocol specifically designed for this
purpose could achieve lower overhead both in terms of additional data
transmitted, and computational requirements.

Using UDP has the advantage that it has the lowest possible overhead and
even the application layer protocol can be kept rather simple as the
exact structure/size of datagrams is always known before
transmission/reception.

The drawbacks of using UDP are that the protocol needs to handle
connections, ensure the detection of reordering, packet loss or multiple
reception and guarantee the consistency of the messages (as in IPv4, the
UDP checksum is optional, and the 16-bit checksum is rather weak).

As per UDP Usage Guidelines [@rfc8085] Datagram Transport Layer Security
(DTLS) is used to secure communications over the protocol.

Sessions
--------

Even though UDP is connectionless, the protocol uses the concept of
sessions. Sessions (in this case) are a temporary, stateful series of
communications between the client and the server. Sessions are
identified by 128-bit UUIDs but sessions are directly associated with
the IP addresses and ports of clients.

The possible states and legal state transfers are shown with black
arrows in . State transfer requests are idempotent and illegal state
transfers must always result in an error reply.

![The state machine representation of a session's
state](figures/diagrams/Statemachine.png){#fig:statemachine
width="0.99\\linewidth"}

The following paragraphs describe the lifecycle of a session over the
NWEmgprotocol.

Both the server and the client are initialised with the state . This
state indicates that the processes were started, but no communication
occurred, and the server is waiting for a client to connect.

The client sends an message to the server to establish a new session.
When the init message is received by the server, it transfers its state
to a hidden *Initialising* state. This state is necessary because
initialization of the classifier model, or other setup actions that
happen during the -\>state transfer could last longer than the timeout
value, and initialization (as all other state transfers) can only happen
once regardless of the number of requests.

After initialization, the client may proceed to the states or . If the
client establishes a new session with the server, the user has to train
the classifier model, and the session must proceed to state. If a
context with a trained model can be restored, for the user (e.g. in case
of reconnection caused by a network error) the -\> state transfer is
legal and should be used.

In both and states the protocol transmits and accepts messages related
to EMG processing. Data-related communications happen asynchronously,
and data messages may not alter the state of either party.

The client may terminate the session in any state. When the server
receives the request, the server informs the client that the session
state is , the session is terminated, and all server-side resources
related to the session are freed up.

The state transfer to is legal because upon reconnection the server
might be able to restore a previous session or training set (or even a
trained model depending on the implementation).

Definitions
-----------

Message: A message is the unit transmitted by the protocol. On IPv4
usually 1 IP packet, on IPv6 strictly one IP packet.

Numeric types: The specification uses the Rust type names[^2] as they
concisely represent numeric types. For the following chapters, the
format is (u\|i\|f)\<number of bits\>, where u means unsigned integer, i
means signed integer, and f represents a floating point value type.
(E.g. u64 is a 64-bit unsigned integer, and f32 is a 32-bit floating
point number.)

Version triplet: At first glance a SemVer version, but in reality a
triplet of u64 values: major, minor, and patch version numbers. When
used as a string, the major, minor and patch values are separated by
dots (e.g. \"0.4.2\").

Payload or message types: Each message type is introduced starting with
a table, showing the fields of a structure along with their types. As an
example, the table describing *version triplet* would be:

    +------------+------------+------------+
    | major: u64 | minor: u64 | patch: u64 |
    +------------+------------+------------+

This format was chosen as it represents structures more efficiently than
pseudocode structure definitions, but unlike tables in RFCs rows are not
of a fixed size in bytes.

The protocol encapsulates all messages into an NWEmgDatagram structure.
The structure represents the data in its final format that is sent over
the DTLS and UDP layers.

    +--------------------+---------+---------------------+----------------+
    | type: DatagramType | id: u32 | message: u8[1..MMS] |  checksum: u32 |
    +--------------------+---------+---------------------+----------------+

The maximum message size (MMS) is determined by the configured maximal
UDP datagram size. The default maximal datagram size the protocol
supports is 1500 bytes, as path maximum transmission units (MTUs) are
typically not more than 1500 bytes.

DatagramType is a one-byte enumerated type, providing unified message
type values. Message types are specified in the next section. New
message types may be added in later versions, but the 1-byte type
identifier is far from exhaustion.

The protocol uses 32-bit IDs that wrap on overflow by design. This ID
length is used, as even at a relatively high (1000 messages/s) message
rate, it would still take the counters more than 1000 hours to wrap. If
sessions need to be reconstructible (for analysis or debugging
purposes), limiting the session length to around 1000 hours does not
constrain the protocol (at least in its current use case).

The sequence numbers provided by the DTLS layer are not used (in upper
layers), as NWEmgsessions are not bound to DTLS sessions.

Message Types {#sec:msgtypes}
-------------

The protocol has two different kinds of messages, control messages, and
data messages. Control messages can be further divided into three
categories based on the possible sender: (i) client (ii) server (iii)
both parties.

All messages (regardless of their type) are sent through the same UDP
sockets, but depending on their type the way a given message is handled
might be entirely different. The implementation should be capable of
processing multiple messages asynchronously, regardless of the type of
incoming messages.

Control messages carry information required to operate the protocol
itself, ensure that the client and server-side state is synchronized,
and take care of error handling. Control messages are prone to packet
loss, thus control messages should be resent by the client in case the
reply to the request times out. Only the client should send messages
repeatedly, control messages have idempotent effects on the server's
state, therefore with each client-initiated retransmission results in a
new reply being sent to the client.

To help implementations reduce or even eliminate the need for dynamic
allocations, all messages are either of a fixed length or have their
maximum sizes specified. The only message type that has a non-fixed size
is the *Data* message, but a safe upper limit to its length always
exists.

### Client-Initiated Control Messages

These messages must be sent from a client. When a client receives a
client-initiated message, the result should be an *Invalid Message*
error.

#### InitRequest {#sec:initreq}

    +------------------+---------------+-----------------------+
    | version: Version | no_items: u32 |  no_bytes_item: u32   |
    +------------------+---------------+-----------------------+
    | batch_size: u32  | user_id: uuid | prev_session_id: uuid |
    +------------------+---------------+-----------------------+

The client sends an Init message when it wishes to establish a session.
The message contains the version triplet of the client. The server sends
a *mismatched version* error if the client's version is not compatible
with the server. The following 3 values specify the size and format of
data messages for the entire length of the session. (Dynamic
reconfiguration might have benefits in the longer run, more on this in
).

The client ID field identifies the user, and the previous\_session\_id
optionally holds the id of a session to be restored if a new session is
initiated due to a network error or a previous failure.

Control messages should be sent using a (large enough) timeout that
allows the server to complete long-running initialisation or clean-up
tasks such as loading or training a complex classification model.

#### DisconnectRequest

    +-------------+
    | reason: u32 |
    +-------------+

The client must send a disconnect request when it is shutting down or is
in a state in which it won't send any further messages in the session.
The disconnect request has a single field, a 32-bit ID reserved for
disconnect reasons. Currently, the reason field is only used for logging
but it could be used to inform the server to retain or instantly remove
the resources allocated for a specific session.

#### StartTrainingRequest

The client must send a *StartInferenceRequest* when it needs the server
to enter training mode. Currently, it has an empty payload, see for
ideas on increasing its flexibility. This message type can only be sent
in state .

#### StartInferenceRequest

The client must send a *StartInferenceRequest* when it needs the server
to enter inference mode. Currently, it has an empty payload, see for
ideas on increasing its flexibility. This message type can only be sent
in states , or .

### Server-Initiated Control Messages

These messages must be sent from a server. When a server receives a
server-initiated message, the result should be an *Invalid Message*
error.

#### InitResponse

    +------------------+------------------+
    | version: Version | session_id: Uuid |
    +------------------+------------------+

The response sent to an *InitRequest*. Informs the client that the
server-side state transfer to was successful, and the server is ready to
accept further messages. The version field contains the version of the
server, and a 128-bit session ID is generated for each session.

#### StateReport

    +------------------+----------------+
    | from: NWEmgState | to: NWEmgState |
    +------------------+----------------+

The message used for state synchronization. The client follows the
server-side state upon receiving a StateReport message. The message
contains the state in which the state transfer was initiated (from) and
the state after the state change (to).

State change messages with identical *from* and *to* states should be
used to indicate when a state change request did not affect the
server-side state. This might be required in various situations i.e. the
loss of the initial response.

State transfer messages are not necessarily the result of
StateChangeRequests.

### Data Messages

These messages handle the transmission of EMG data and classification
results.

#### Data

The message contains the elements of the data vector packed without
padding with LE byte ordering. Requirements for delivering data messages
are unique, with the most important aspect being the up-to-date delivery
of control information. Data messages are never retransmitted ( messages
should be used when the loss of messages can not be tolerated).

#### DataResponse

    +-----------------+---------------+---------------+
    | originator: u32 | velocity: f32 | action id: u8 |
    +-----------------+---------------+---------------+

The response sent to Data and DataGuaranteed messages. Contains
information required to control the prostheses. The originator field is
used to connect the response to the request (and as a result detect
reordering). This message type is specific to EMG processing, the
action\_id field specifies the result of the classification, along with
a corresponding velocity.

#### DataGuaranteed {#sec:dataguaranteed}

The message kind carrying data semi-reliably. This message kind is an
optional element of implementations. The delivery of these messages is
guaranteed using redundancy based on confirmations.

When an implementation does not provide this functionality, error
reports should inform the client that the functionality is not
available.

### Utility Control Messages

Both the client and server may send and/or receive utility messages.

#### ErrorReport

    +----------+-------------+---------------------+
    | code: u8 | length: u16 | message: [u8; 1024] |
    +----------+-------------+---------------------+

The message carries information regarding an error that may or may not
be fatal to the session. Error reports should be of the following kind:

-   Invalid message

-   Critical error

-   Invalid state transfer

-   Mismatched version

-   Custom error

Error reports allow a variable length (max. 1024 byte long) message to
be sent detailing the error. The length field holds the length of the
message, the message field holds the UTF-8 encoded bytes of the error
message.

#### Unknown

Under normal circumstances, it is impossible for the protocol to send or
receive a message of type *unknown*, however, it seemed logical to
create a message type that represents all messages received with an
unknown message type.

Reasons for a server or a client receiving unknown messages could be
undetected data corruption (though the chances of that happening are
extremely low provided the 32-bit checksum), human error, or mismatched
protocol versions (in case a new message type is introduced on one side,
but not the other), but this case is accounted for during
initialisation, as a compatibility check occurs.

The reception of messages of *unknown* kind should be treated as errors.

[^1]: <https://mqtt.org/mqtt-specification/>

[^2]: https://doc.rust-lang.org/beta/book/ch03-02-data-types.html
