## Chunk Messages {#Chunk}
A `CHUNK` message contains the first 8 bytes of the SHA-3 ([@SHA3]) checksum of the chunk. The rest are 1001 bytes (i.e. one _chunk_ of the file). We chose 997 bytes so that the size of the entire UDP datagram is equal to 997 bytes. 

{#chunkMessage}
```
ChunkMessage {
    U16 StreamId,
    U8 MessageType = 0x00,
    U64 SequenceNumber = 0x00,
    U8[8] Checksum
    U8[997] Payload
}
```
Figure: The structure of a chunk message