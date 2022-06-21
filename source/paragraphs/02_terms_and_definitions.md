## Terms and Definitions
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP14 [@!RFC2119] [@!RFC8174] when, and only when, they appear in all capitals, as shown here.

Commonly used terms in this document:

Client: 
: An endpoint that participates in a stream and wants to receive a file.

Server: 
: An endpoint that participates in a stream and wants to send a file.

Stream:
: An ordered unidirectional (Client to Server) sequence of bytes. It is guaranteed that each byte will arrive and that it will arrive in exactly the same order it was sent out.