## Notation
We define `U8`, `U16`, `U32`, `U64` and `U128`, `U256` as unsigned 8-, 16-, 32-, 64-, 128-, or 256-bit integers. `I8`, `I16`, `I32`, `I64`, `I128`, and `I256` are signed 8-, 16-, 32-, 64-, 128-, or 256-bit integers. A `string` is a UTF-8 [@!RFC3629] encoded zero-terminated string. The syntax `DataType[]` defines a variable-length array of DataType; `DataType[N]` a fixed size array with `N` elements of type `DataType`.

Messages are represented in a C-style way (see (#exampleMessage)). They may be annotated by C-style comments. All members are laid out continuously on wire, any padding will be made explicit. If a field has a constant value we write it in the form of an assignment. We use the prefix `0x` to denote hexadecimal values.

{#exampleMessage}
```
ExampleMessage {
    U8 firstMember = 42,
    I8 secondMember = 0x01, // A hexadecimal value
    U16 padding0,
    string thirdMember,
    I32[16] array
}
```
Figure: Example Message