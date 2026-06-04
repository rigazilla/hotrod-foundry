# Hot Rod Client Implementation Roadmap

**Version**: 1.0  
**Target Protocol**: Hot Rod 4.0 (extensible to 4.1)  
**Last Updated**: 2026-06-04

## Overview

This roadmap defines **incremental, testable steps** for implementing a Hot Rod protocol client in any programming language. Each step:

- Is **small and focused** (1-3 hours of work)
- Produces **working, testable code**
- Has **language-agnostic documentation**
- Mirrors **Java client tests** for validation
- Can be **replicated in any language**

## Development Principles

1. **Test-Driven**: Tests written before/during implementation
2. **Incremental**: Each step builds on previous ones
3. **Cross-Validated**: Compare wire format with Java client using Wireshark
4. **Language-Agnostic**: Documentation doesn't assume any specific language

## Reference Sources

| Source | Purpose |
|--------|---------|
| `../infinispan/client/hotrod-client/` | Java reference implementation |
| `../hotrod-dissector/schemas/*.ksy` | Formal protocol specification (Kaitai) |
| Infinispan Protocol Docs | Official documentation |

---

## Step 0: Foundation Setup ⚙️

**Goal**: Project structure, build system, test infrastructure

### Deliverables
- [ ] Project structure (src/, include/, tests/, docs/)
- [ ] Build system (CMake for C++17, MSBuild for C#, etc.)
- [ ] Test framework integration (Google Test, NUnit, etc.)
- [ ] CI/CD pipeline (GitHub Actions)
- [ ] Test Infinispan server setup guide
- [ ] Can connect to test server via TCP

### Documentation
- `docs/00-getting-started.md` - Setup instructions
- Language-specific build/test setup

### Tests
- Build system compiles successfully
- Test framework runs
- Can establish TCP connection to Infinispan server (port 11222)

### Java Reference
- Maven build structure
- Test infrastructure in `src/test/java/`

### Success Criteria
✅ CI pipeline runs and passes  
✅ Can connect to Infinispan test server  
✅ Test framework executes  

---

## Step 1: Wire Format Primitives 🔤

**Goal**: Implement basic protocol encoding/decoding (no network I/O yet)

### Deliverables
- [ ] Variable-length integer encoding/decoding (vInt - 32-bit)
- [ ] Variable-length long encoding/decoding (vLong - 64-bit)
- [ ] ZigZag encoding for signed integers
- [ ] Length-prefixed string encoding/decoding (UTF-8)
- [ ] Length-prefixed byte array encoding/decoding
- [ ] Unit tests for all primitives
- [ ] Test vectors validated against Java output

### Documentation
- `docs/01-wire-format-primitives.md`
  - vInt/vLong encoding algorithm
  - ZigZag encoding explanation
  - String encoding (vInt length + UTF-8 bytes)
  - Byte array encoding (vInt length + raw bytes)
  - Wire format examples with hex dumps

### Implementation (C++17 example)
```cpp
class Codec {
public:
    // Variable-length integers (1-5 bytes for vInt, 1-9 for vLong)
    void writeVInt(uint32_t value);
    uint32_t readVInt();
    
    void writeVLong(uint64_t value);
    uint64_t readVLong();
    
    // Signed integers with ZigZag encoding
    void writeSignedVInt(int32_t value);
    int32_t readSignedVInt();
    
    // Strings (UTF-8)
    void writeString(const std::string& str);
    std::string readString();
    
    // Byte arrays
    void writeBytes(const std::vector<uint8_t>& bytes);
    std::vector<uint8_t> readBytes();
};
```

### Test Cases
Test vectors in `test-vectors/step-01-primitives/`:

**vInt encoding**:
- `0` → `0x00`
- `127` → `0x7F`
- `128` → `0x80 0x01`
- `255` → `0xFF 0x01`
- `256` → `0x80 0x02`
- `16383` → `0xFF 0x7F`
- `16384` → `0x80 0x80 0x01`

**vLong encoding**: Similar pattern, up to 9 bytes

**Signed vInt (ZigZag)**:
- `0` → `0x00`
- `-1` → `0x01`
- `1` → `0x02`
- `-2` → `0x03`
- `2` → `0x04`

**Strings**:
- `""` (empty) → `0x00`
- `"hello"` → `0x05 68 65 6C 6C 6F`
- `"café"` (UTF-8) → `0x05 63 61 66 C3 A9`

**Byte arrays**:
- `[]` → `0x00`
- `[0x01, 0x02, 0x03]` → `0x03 01 02 03`

### Java Reference
- `org.infinispan.client.hotrod.impl.transport.netty.ByteBufUtil`
  - `writeVInt()` / `readVInt()` (lines 70-112)
  - `writeVLong()` / `readVLong()` (lines 82-102)
  - `writeString()` / `readString()` (lines 26-36)
  - `writeArray()` / `readArray()` (lines 19-54)
- `org.infinispan.commons.io.SignedNumeric`
  - `encode()` / `decode()` for ZigZag (lines 41-47)

### Validation
✅ All test vectors pass  
✅ Encoded bytes match Java output byte-for-byte  
✅ Round-trip encode/decode works  
✅ Edge cases handled (0, max values, empty strings/arrays)

### Success Criteria
✅ Can encode/decode all primitive types  
✅ Output matches Java client exactly  
✅ Documentation sufficient to implement in another language without seeing code

---

## Step 2: Protocol Headers 📋

**Goal**: Build and parse Hot Rod request/response headers

### Deliverables
- [ ] Request header structure
- [ ] Response header structure
- [ ] Header encoding/decoding
- [ ] Magic byte validation
- [ ] Unit tests for headers
- [ ] Test vectors for various header combinations

### Documentation
- `docs/02-protocol-headers.md`
  - Request header format (magic, message ID, version, opcode, etc.)
  - Response header format
  - Field descriptions and constraints
  - Wire format examples

### Request Header Structure
```
Magic Byte:          0xA0 (1 byte)
Message ID:          vLong (unique per request)
Version:             1 byte (0x28 for 4.0, 0x29 for 4.1)
Opcode:              1 byte (operation code)
Cache Name:          string (vInt length + UTF-8)
Flags:               vInt (bit field)
Client Intelligence: 1 byte (0x01 = basic, 0x03 = hash-aware)
Topology ID:         vInt (current topology version)
```

### Response Header Structure
```
Magic Byte:           0xA1 (1 byte)
Message ID:           vLong (matching request)
Opcode:               1 byte (matching request opcode)
Status:               1 byte (0x00 = success, 0x01 = not found, etc.)
Topology Change Flag: 1 byte (0x00 = no change, 0x01 = topology changed)
```

### Test Cases
- Encode/decode request header
- Encode/decode response header
- Round-trip test
- Invalid magic byte detection
- Message ID matching between request/response

### Java Reference
- `org.infinispan.client.hotrod.impl.protocol.Codec40`
  - `writeHeader()` method
  - `readHeader()` method
- `org.infinispan.client.hotrod.impl.protocol.HotRodConstants`
  - Magic bytes, opcodes, status codes

### Success Criteria
✅ Headers encode/decode correctly  
✅ Wire format matches Java  
✅ Can distinguish request vs response by magic byte

---

## Step 3: PING Operation 🏓

**Goal**: First complete network operation - simplest request/response

### Deliverables
- [ ] TCP connection management
- [ ] PING request (opcode 0x17)
- [ ] PING response parsing
- [ ] Connection error handling
- [ ] Integration test against real server

### Documentation
- `docs/03-ping-operation.md`
  - PING purpose and use cases
  - Request format (header only, no payload)
  - Response format (header + status)
  - Connection handling

### Implementation
```cpp
class Connection {
    void connect(const std::string& host, int port);
    void send(const std::vector<uint8_t>& data);
    std::vector<uint8_t> receive();
    void close();
};

class RemoteCache {
    bool ping();  // Returns true if server responds successfully
};
```

### Test Cases
- Connect to server and PING successfully
- Verify response status is 0x00 (success)
- Test connection error handling (server not running)
- Test timeout handling
- **Cross-validation**: Capture with Wireshark and compare to Java PING

### Java Reference
- `org.infinispan.client.hotrod.impl.operations.PingOperation`
- Connection handling in transport layer

### Success Criteria
✅ Can PING server and receive success response  
✅ Wire format matches Java (verified with Wireshark)  
✅ Error handling works correctly

---

## Step 4: GET Operation 🔍

**Goal**: First read operation with key serialization

### Deliverables
- [ ] GET request encoding (opcode 0x03)
- [ ] Key serialization
- [ ] Response value parsing
- [ ] Handle "key not found" (status 0x01)
- [ ] Cross-client validation (Java PUT → Your GET)

### Documentation
- `docs/04-get-operation.md`

### Test Cases
- GET non-existent key → returns null
- Java client PUTs key, your client GETs → verify value
- GET with empty key
- GET with large key (>127 bytes)

### Java Reference
- `org.infinispan.client.hotrod.impl.operations.GetOperation`

### Success Criteria
✅ Can retrieve values PUT by Java client  
✅ Correctly handles key not found

---

## Step 5: PUT Operation ✍️

**Goal**: First write operation

### Deliverables
- [ ] PUT request (opcode 0x01)
- [ ] Lifespan/maxIdle parameters
- [ ] Previous value in response
- [ ] Cross-client validation (Your PUT → Java GET)

### Documentation
- `docs/05-put-operation.md`

### Test Cases
- PUT and verify with GET
- PUT with lifespan, verify expiration
- Cross-client: Your PUT → Java GET

### Java Reference
- `org.infinispan.client.hotrod.impl.operations.PutOperation`

---

## Step 6: REMOVE Operation 🗑️

**Goal**: Delete operation

### Deliverables
- [ ] REMOVE request (opcode 0x0B)
- [ ] Return previous value

### Java Reference
- `org.infinispan.client.hotrod.impl.operations.RemoveOperation`

---

## Step 7: Metadata Operations 📊

**Goal**: Version-based operations (CAS)

### Deliverables
- [ ] GET_WITH_METADATA (opcode 0x1B)
- [ ] REPLACE_IF_UNMODIFIED (opcode 0x09)
- [ ] Metadata structure (version, created, lifespan, etc.)

### Java Reference
- `org.infinispan.client.hotrod.impl.operations.GetWithMetadataOperation`
- `org.infinispan.client.hotrod.impl.operations.ReplaceIfUnmodifiedOperation`

---

## Step 8: Error Handling ⚠️

**Goal**: Proper error response handling

### Deliverables
- [ ] ERROR response parsing (opcode 0x50)
- [ ] Error message extraction
- [ ] Exception hierarchy

### Java Reference
- `org.infinispan.client.hotrod.exceptions.*`

---

## Step 9: Bulk Operations 📦

**Goal**: Multi-key operations

### Deliverables
- [ ] GET_ALL (opcode 0x2F)
- [ ] PUT_ALL (opcode 0x2D)
- [ ] BULK_GET (opcode 0x1F)

### Java Reference
- Bulk operation classes

---

## Step 10: Connection Pooling 🔄

**Goal**: Efficient connection management

### Deliverables
- [ ] Connection pool
- [ ] Thread safety
- [ ] Connection lifecycle

### Java Reference
- `org.infinispan.client.hotrod.impl.transport.netty.ChannelPool`

---

## Step 11: Topology Awareness 🗺️

**Goal**: Track server topology

### Deliverables
- [ ] Parse topology update responses
- [ ] Track server nodes
- [ ] Topology version tracking

### Java Reference
- `org.infinispan.client.hotrod.impl.topology.TopologyInfo`

---

## Step 12: Consistent Hashing 🎯

**Goal**: Smart routing to servers

### Deliverables
- [ ] Consistent hash implementation
- [ ] Key-to-server mapping
- [ ] Segment-based routing

### Java Reference
- `org.infinispan.client.hotrod.impl.consistenthash.*`

---

## Step 13+: Advanced Features

- Authentication (SASL)
- Transactions (XA)
- Client listeners and events
- Counters
- Queries (Ickle)
- Streaming operations (4.1+)
- Multimap operations

---

## Completion Criteria Per Step

For each step to be considered **complete**:

1. ✅ **Tests Pass**: All unit tests green
2. ✅ **CI Passes**: GitHub Actions successful
3. ✅ **Cross-Validation**: Wire format matches Java (Wireshark check)
4. ✅ **Test Vectors**: Golden test data validated
5. ✅ **Documentation**: Another language could implement using only the docs
6. ✅ **Progress Updated**: Implementation PROGRESS.md updated

---

## Notes

- Steps 1-3 establish foundation (encoding + first network operation)
- Steps 4-6 cover basic cache operations
- Steps 7-9 add robustness
- Steps 10-12 add performance and clustering support
- Step 13+ enables advanced use cases

**Start with Step 1 and proceed sequentially.** Each step assumes previous steps are complete.
