# Binary protocol

## Notes

- How to gracefully handle schema changes?
- How to handle versioning of schemas?

## Use-cases

- REST API
- RPC API
- JSON data storage
- Database data storage

## Features

- Bit level precision
- Custom IDL
- Compiled into many languages
- High performance
- Small payload size
- Small library size
- Data validation
- Zero-copy
- (optional) Streaming
- (optional) Lazy decoding
- (optional) Delta encoding
- (optional) Schema
- (optional) Compression

## Research

### Papers

- https://arxiv.org/html/2407.13494v2

### Cap'n Proto

```
struct Date @0x5c5a558ef006c0c1 {
  year @0 :Int16; # @n marks order values were added to the schema
  month @1 :UInt8;
  day @2 :UInt8;
}
```

- Outer objects appear entirely before inner objects
- Cap’n Proto objects are always allocated in an “arena” or “region” style, which is faster and promotes cache locality
- Uses little-endian
- Cap'n Proto schemas are designed to be flexible as possible and pushes data validation to the application level, allowing arbitrary renaming of fields, adding new fields, and making concrete types generic

#### Videos

- https://www.youtube.com/watch?v=6V_lVZzV6AE&pp=ygULQ2FwJ24gUHJvdG8%3D
- https://www.youtube.com/watch?v=2H-Azm8tM24&pp=ygULQ2FwJ24gUHJvdG8%3D

### FlatBuffers

- It supports “zero-copy” deserialization
- The serialized format allows random access to specific data elements without parsing all data
- Uses little-endian
- FlatBuffers enables the schema to evolve over time while still maintaining forwards and backwards compatibility with old flatbuffers
    - New fields MUST be added to the end of the table definition
    - You MUST not remove a field from the schema, even if you don't use it anymore
    - Its generally OK to change the name of tables and fields, as these are not serialized to the buffer. It may break code that would have to be refactored with the updated name
- Fields in a table can have any order, and objects to some extent can be stored in many orders
    - Instead, the format is defined in terms of offsets and adjacency only
    - This may mean two different implementations may produce different binaries given the same input values, and this is perfectly valid
- The format also doesn't contain information for format identification and versioning

### Protocol Buffers

#### Videos

- https://www.youtube.com/watch?v=9IxE2UQqJCw&pp=ygULQ2FwJ24gUHJvdG8%3D

### Apache Avro

### Apache Thrift

### External Data Representation

### Structured Data eXchange Format

### Abstract Syntax Notation One
