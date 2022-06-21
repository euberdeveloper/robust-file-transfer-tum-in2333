# Streams
The protocol is based on streams. Streams are ordered sequences of bytes. RFT guarantees that all messages in a stream will arrive in-order (or will be retransmitted). There is not happened-before relation between different messages in different streams. A stream represents exactly one filepath on the server.

A stream can be established between a client and a server. A stream is identified by a **Stream ID**, a unique U16 value greater than 0. A stream ID is chosen by the server. A server **MAY** choose its stream IDs however it sees fit. Each stream contains the following state:

* **Encryption**. Right now, we don't support any form of encryption.
* **Compression**. Right now, we don't support any kind of compression.
* **Flow control**. We support flow control with a sliding window. See (#FlowControl) for details.
* **Congestion control**. We support congestion control with an algorithm similar to TCP Reno. See (#CongestionControl) for details.
* **Connection migration**. We support migrating a stream. See (#StreamMigration).

## Establishing a stream
A stream is established by a three way handshake:

```

             +----------------------+
    +------->|       Initial        |
    |        +----------+-----------+
    |                   |
Error or                | CLIENT_HELLO
Timeout                 |
    |                   v
    |        +----------------------+
    +--------| Parse Client Request |
    |        +----------+-----------+
    |                   |
    |                   | SERVER_HELLO
    |                   |
    |                   v
    |        +----------------------+
    +--------| Parse Server Request |
             +----------+-----------+
                        |
                        | ACK
                        |
                        v
             +----------------------+
             |      Established     |
             +----------------------+

```
Figure: 3-way handshake of RFT

1.  The client sends a CLIENT_HELLO ((#ClientHello)) message. The request **MUST** be filled with an appropriately sized `WindowInMessages`. To allow for the resumption of previous transmissions the client **MAY** choose an offset (in the chunk's payload size) of the file.

2.  The server parses the CLIENT_HELLO message and tries to process the additional headers. Right now, none are defined, so they **MUST** be set to 0.
    1.  In case of success, the server responds with a SERVER_HELLO ((#ServerHello)) message. It assigns a unique stream ID. The request **MUST** be filled with an appropriately sized `WindowInMessages` and the stream ID.
    2.  In case of an error the server responds with an appropriate ERROR ((#Error)) message. The server **MAY** pose arbitrary constraints. For example, it may not allow two streams to be created for the same file.

3.  The client waits at most 5 seconds for the SERVER_HELLO message. If it was not received, it **MUST** retry to establish the stream (i.e., start at step 1). Any messages that are received later **SHALL** be discarded.

    The client parses the SERVER_HELLO message and tries to process the additional headers. Right now, none are defined, so they **MUST** be set to 0.

4.  If the client accepts the headers it sends an ACK ((#ACK)) message to the server. The `SequenceNumber` **MUST** be set to 0. This concludes the stream establishing process.

## Message sending {#ErrorHandling}
Both endpoints must validate all messages (e.g. by using the checksums). All valid messages in a stream except for ACKs themselves **MUST** be acknowledged using an ACK message. Each partner in a stream keeps track of the number of bytes the other side has sent. Both sides start out with a `SequenceNumber` of 0 after the initial handshake. Multiple messages **MAY** be acknowledged in one message (i.e., cumulative ACKs). ACKs **MUST** always contain sequence numbers that correspond to whole messages.

For each message, the `SequenceNumber` is increased by the size of the message in bytes (e.g., what you would get for the expression `sizeof(message)`). In other words: The sequence number of a message is the starting position (in bytes) of this particular message in the stream.

If a message arrives with a `SequenceNumber` larger than the one expected, the receiver **MUST** discard it and send an ACK with the last valid sequence number.

If the sender doesn't receive an ACK within 5 seconds of sending the message, the message is considered lost and **MUST** be retransmitted.

## Sending data
After the handshake completes, the server **MUST** start sending the required file chunks ((#Chunk)). A chunk of a file is 997 bytes. The server **MUST** respect the client's requested offset from the stream 3-way handshake. If less than 997 bytes of the file remain, the server **MUST** fill the rest of the message with 0x00. A client **MUST** ignore this padding. 

### File listing
A special case is the file with a path of "" (the empty string). In this case the server **MUST** produce a file listing, that prints the path to all available files separated by `\r\n`. Even though this file is purely virtual, the exact same mechanisms apply.

## Closing a stream
A stream can be closed at any time by either the client or the server. This can be done by sending a FIN ((#FIN)) message. The other endpoint **MUST** ACK the message before resources can be freed. Upon reception all messages still in-flight **MAY** be discarded by both client and server. A client **MAY** choose to process outstanding CHUNK messages, though.

## Stream Migration {#StreamMigration}
Streams are identified by a stream ID. If the IP address of either the client or the server changes they **MUST** inform the other party with a CONNECTION_MOVED ((#ConnectionMoved)) message. The receiver **MUST** use the information from the lower layers to change its state and send all further messages to the new address.

Any messages already in-flight **MAY** be processed by both the server and the client. If either opts not to receive them, normal error handling ((#ErrorHandling)) applies.

## Flow control {#FlowControl}
Each ACK message ((#ACK)) contains `WindowInMessages`, the amount of messages a client may receive at any time. A server **MUST** take care to never exceed this limit. If the window remains zero for five consecutive messages the sender **MUST** assume the receiver has failed and terminate the stream.

## Congestion control {#CongestionControl}
Congestion control is done with an algorithm similar to TCP's congestion control [@?RFC5681]. 

The amount of messages in-flight is limited by a congestion window `cwnd`. The following rules **SHALL** apply to determine the size of the window.

Slow Start:
: `cwnd` is set to 1. For each received ACK or SERVER_HELLO message the window should be determined as `cwndNew = min(cwndOld * 2, receiverWindow)`. This phase stops when the slow start threshold is reached.

Congestion Avoidance:
:   Once slow start has stopped, we reach this phase. Here we increase our `cwnd` by 1 if all messages in this window have been acknowledged. Our `cwnd` still may not grow larger than the advertised receiver window.
    
    If we receive three duplicate ACKs we assume the message is lost and retransmit it immediately (fast retransmit). We then set our `cwnd` to half of its current size and continue with the congestion avoidance phase.

    Once a timeout occurs `cwnd` is reset to 1 and the slow start threshold is set to half of the number of messages in-flight. We then go back to the slow start phase.