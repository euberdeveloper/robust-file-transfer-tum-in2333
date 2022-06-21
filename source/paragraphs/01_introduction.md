# Introduction

Robust File Transfer (RFT) is a file-transfer protocol on top of UDP [@?RFC0768]. This document defines version 0.2 of RFT. In spirit it is very similar to QUIC [@?RFC9000], albeit arguably a bit easier.

RFT is connection-oriented and stateful. A RFT _stream_ is a unidirectional ordered sequences of byte between a _client_ and a _server_. Streams support IP address migration, flow control and congestion control. The protocol guarantees in-order delivery for all messages belonging to a stream. There is no such guarantee for messages belonging to different streams.

RFT _messages_ are either _chunk messages_, containing file data, or _control messages_. Control messages are used to request and signal state changes on either the server or the client. There's a 1:1 correspondence between a RFT message and a UDP datagram.

RFT employs flow and congestion control on a per-stream basis.