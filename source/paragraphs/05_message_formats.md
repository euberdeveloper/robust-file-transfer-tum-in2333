# Message Formats
RFT knows two types of messages: `Control` and `Chunk` messages. Messages **MUST** have a little-endian format.

All RFT messages contain a stream ID as well as a message type. If there is no stream ID (e.g., because a stream is yet to be established) it **MUST** be set to `0`.

{#RftMessageFormat}
```
Message {
    U16 StreamId,
    U8 MessageType,
    U64 SequenceNumber,
    // More fields, depending on the message
}
```
Figure: All mandatory fields of any RFT message.

## Control Messages
Control messages are typically very small and aren't secured by themselves. Instead, they rely on UDP's checksum for error correction. A message is a control message if its MessageType is greater than 0.

### CLIENT_HELLO {#ClientHello}
A `CLIENT_HELLO` message is sent by a client to a server to establish a new stream. Since a stream is not established yet, the field **MUST** be set to 0 by the client. The message type **MUST** be set to 1.

The `Version` field carries information about the RFT version that the client can support. It **MUST** be set to the highest version a client can support. For RFT 0.2 the value **MUST** be 0x01.

The `NextHeaderType` and `NextHeaderOffset` fields aren't used in version 0.2 of RFT and **MUST** be set to 0 by the client. The server **MUST** reject all messages with different values. In the future they can be used for additional parameters like encryption and compression.

`WindowInMessages` tells the server the advertised receiver window.

`StartChunk` allows the client to specify an offset from which to start chunk transmission. The first chunk has an offset of `0`.

`Filename` contains the filename of the requested file. It **must** be absolute. It **MAY** be empty, in this case the server replies with a file listing.

```
ClientHello {
    U16 StreamId = 0x00,
    U8 MessageType = 0x01,
    U64 SequenceNumber = 0x00,
    U8 Version = 0x01,
    U8 NextHeaderType = 0x00,
    U8 NextHeaderOffset = 0x00,
    U16 WindowInMessages,
    U32 StartChunk,
    string Filename
}
```
Figure: A CLIENT_HELLO message.

### SERVER_HELLO {#ServerHello}
The `SERVER_HELLO` message is sent as a response by the server. 

* The `StreamId` field **MUST** contain the stream ID that the server has allocated for this stream.
* The `MessageType` **MUST** be set to 2.
* The `Version` field **MUST** contain `min(client version, maximum protocol version the server supports)`.
* The `NextHeaderType` and `NextHeaderOffset` fields aren't used in version 0.2 of RFT and **MUST** be set to 0 by the server. The client **MUST** reject all messages with different values. 
* The `WindowInMessages` field tells the client the advertised receiver window.
* The `Checksum` field contains the SHA-3 hash [@SHA3] of the entire file. 
[comment] # I think this one requires discussion: do we need to include the LastModified in the protocol?
* The `LastModified` field contains the number of seconds since 01. January 1970. It is in the server's timezone. Coordination about timezones between client and server is out of scope for this protocol.
* The `FileSizeBytes` contains the file's size.

```
ServerHello {
    U16 StreamId,
    U8 MessageType = 0x02,
    U64 SequenceNumber = 0x00,
    U8 Version,
    U8 NextHeaderType = 0x00,
    U8 NextHeaderOffset = 0x00,
    U16 WindowInMessages,
    U256 Checksum,
    I64 LastModified,
    U64 FileSizeBytes
}
```
Figure: A SERVER_HELLO message.

<reference anchor="SHA3" quoteTitle="true" target="https://doi.org/10.6028/NIST.FIPS.202">
    <front>
        <title>SHA-3 Standard: Permutation-Based Hash and Extendable-Output Functions</title>
        <seriesInfo name="DOI" value="10.6028/NIST.FIPS.202"/>
        <seriesInfo name="FIPS PUB" value="202"/>
        <author>
            <organization showOnFrontPage="true">National Institute of Standards and Technology</organization>
        </author>
        <date year="2015" month="August"/>
    </front>
</reference>


### ACK {#ACK}
An `ACK` message is sent anytime someone wants to acknowledge something.
[comment] # Do we need to send a WindowInMessages every single ACK?
* `WindowInMessages` corresponds to the receive window of the sender. It's measured in messages.
* `AckNumber` is the sequence number of the last message that should be acknowledged.

```
ACK {
    U16 StreamId,
    U8 MessageType = 0x03,
    U64 SequenceNumber,
    U16 WindowInMessages, //Number of chunk messages that can be held in the client's buffer
    U64 AckNumber
}
```
Figure: An ACK message.

[comment] # Do we need FIN anyway? The transmission could end when sending the empty data packet, and this one creates a new condition branch to consider.
### FIN {#FIN}
A `FIN` message can be sent to close a stream.

```
FIN {
    U16 StreamId,
    U8 MessageType = 0x04
    U64 SequenceNumber
}
```
Figure: A FIN message.

[comment] # According to discussion this thing could be taken care of by resending the last correct ACK from a new address instead - we know now where to start sending from again.
### CONNECTION_MOVED {#ConnectionMoved}
A `CONNECTION_MOVED` message can be sent by either the client or the server. It indicates that the sender's IP address has changed and any future communication should be sent to the new address.

```
ConnectionMoved {
    U16 StreamId,
    U8 MessageType = 0x07,
    U64 SequenceNumber = 0x00,
}
```
Figure: A CONNECTION_MOVED message.

### ERROR {#Error}
The `ERROR` message informs clients of an error. The tuple `(ErrorCategory, ErrorCode)` uniquely identifies each error. An optional `Message` field **MAY** give additional information, otherwise it **MUST** be empty ("").

```
Error {
    U16 StreamId,
    U8 MessageType = 0xFF,
    U64 SequenceNumber = 0x00,
    U8 ErrorCategory,
    U8 ErrorCode,
    string Message
}
```
Figure: A ERROR message.

| ErrorCategory | ErrorValue | Meaning |
|---------------|------------|---------|
| 0             | According to the macros in glibc's error codes (see https://www.gnu.org/software/libc/manual/html_node/Error-Codes.html). | Same meaning as the glibc error codes. |
Table: List of all valid error category/ error value combinations.
