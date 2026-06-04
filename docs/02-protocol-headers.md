# Step 2: Protocol Headers

**Status**: Foundation  
**Prerequisite**: Step 1 (Wire Format Primitives)  
**Estimated Time**: 2-3 hours  
**Test Vectors**: `../test-vectors/step-02-headers/`

## Overview

Hot Rod protocol messages consist of a header followed by operation-specific payload. This step implements request and response header encoding/decoding.

**No network I/O yet** - pure header serialization with byte buffers.

## Header Structure

### Request Header

Every request message starts with magic byte `0xA0` followed by:

```
Magic Byte (0xA0):       1 byte (constant)
Message ID:              vLong (unique per request)
Version:                 1 byte (0x28 = Protocol 4.0, 0x29 = Protocol 4.1)
Opcode:                  1 byte (operation code)
Cache Name:              string (vInt length + UTF-8 bytes)
Flags:                   vInt (bitfield for options)
Client Intelligence:     1 byte (topology awareness level)
Topology ID:             vInt (current topology version)
[Protocol 4.0+] Other Params Count: vInt
[Protocol 4.0+] Other Params: key-value pairs (if count > 0)
```

**Simplified for Step 2**: We'll skip MediaTypes and Other Params for now (added in later protocol versions). Focus on core header fields.

### Response Header

Every response message starts with magic byte `0xA1` followed by:

```
Magic Byte (0xA1):       1 byte (constant)
Message ID:              vLong (matching request)
Opcode:                  1 byte (response operation code)
Status:                  1 byte (result status)
Topology Change Marker:  1 byte (0x00 = no change, 0x01 = topology changed)
[If topology changed]    Topology data (handled in Step 11)
```

---

## Request Header Fields

### Magic Byte (0xA0)

Fixed value identifying this as a request message.

**Value**: `0xA0` (160 decimal)

### Message ID

Unique identifier for request/response correlation. Client generates this (typically incrementing counter).

**Type**: vLong (variable-length 64-bit unsigned integer)  
**Range**: 0 to 2^64-1  
**Example**: Message ID 1 → `0x01`

### Version

Protocol version byte.

**Type**: uint8  
**Values**:
- `0x28` (40 decimal) = Protocol 4.0
- `0x29` (41 decimal) = Protocol 4.1
- `0x1E` (30 decimal) = Protocol 3.0 (legacy)
- `0x1F` (31 decimal) = Protocol 3.1 (legacy)

**For Step 2**: Use `0x28` (Protocol 4.0)

### Opcode

Operation code identifying the request type.

**Type**: uint8

**Common Request Opcodes**:
| Opcode | Value | Operation |
|--------|-------|-----------|
| PUT | 0x01 | Store key-value |
| GET | 0x03 | Retrieve value |
| CONTAINS_KEY | 0x0F | Check if key exists |
| REMOVE | 0x0B | Delete entry |
| PING | 0x17 | Server health check |
| STATS | 0x15 | Get server statistics |
| CLEAR | 0x13 | Clear cache |
| SIZE | 0x29 | Get cache size |
| GET_WITH_METADATA | 0x1B | Get with version/metadata |
| REPLACE_IF_UNMODIFIED | 0x09 | CAS operation |

**See**: `reference/operation-codes.md` for complete list

### Cache Name

Name of the cache to operate on.

**Type**: string (length-prefixed UTF-8)  
**Encoding**: vInt length + UTF-8 bytes (from Step 1)

**Examples**:
- `""` (empty, default cache) → `0x00`
- `"myCache"` → `0x07 6D 79 43 61 63 68 65`

### Flags

Bitfield for request options.

**Type**: vInt (variable-length unsigned integer)

**Common Flags** (bitwise OR):
| Flag | Value | Meaning |
|------|-------|---------|
| FORCE_RETURN_VALUE | 0x0001 | Return previous value |
| DEFAULT_LIFESPAN | 0x0002 | Use server default lifespan |
| DEFAULT_MAXIDLE | 0x0004 | Use server default max idle |
| SKIP_CACHE_LOAD | 0x0008 | Don't load from cache loader |
| SKIP_INDEXING | 0x0010 | Don't index this entry |

**For Step 2**: Use `0x00` (no flags)

### Client Intelligence

Indicates client's topology awareness level.

**Type**: uint8

**Values**:
| Value | Name | Description |
|-------|------|-------------|
| 0x01 | BASIC | No topology awareness |
| 0x02 | TOPOLOGY_AWARE | Knows server list |
| 0x03 | HASH_DISTRIBUTION_AWARE | Smart routing with consistent hashing |

**For Step 2**: Use `0x01` (BASIC) - we'll implement smart routing in Step 12

### Topology ID

Current topology version known by client.

**Type**: vInt  
**Value**: Starts at 0, incremented when topology changes

**For Step 2**: Use `0x00` (no topology knowledge yet)

---

## Response Header Fields

### Magic Byte (0xA1)

Fixed value identifying this as a response message.

**Value**: `0xA1` (161 decimal)

### Message ID

Matches the request's message ID for correlation.

**Type**: vLong  
**Must Match**: Request message ID

### Opcode

Response operation code (usually request opcode + 1).

**Type**: uint8

**Common Response Opcodes**:
| Opcode | Value | Corresponding Request |
|--------|-------|----------------------|
| PUT_RESPONSE | 0x02 | PUT (0x01) |
| GET_RESPONSE | 0x04 | GET (0x03) |
| CONTAINS_KEY_RESPONSE | 0x10 | CONTAINS_KEY (0x0F) |
| REMOVE_RESPONSE | 0x0C | REMOVE (0x0B) |
| PING_RESPONSE | 0x18 | PING (0x17) |
| ERROR_RESPONSE | 0x50 | Any (error) |

**Pattern**: Response opcode = Request opcode + 1 (except ERROR)

### Status

Result status code.

**Type**: uint8

**Status Codes**:
| Code | Name | Meaning |
|------|------|---------|
| 0x00 | NO_ERROR | Success |
| 0x01 | NOT_PUT_REMOVED_REPLACED | Operation not applied (e.g., PUT_IF_ABSENT when key exists) |
| 0x02 | KEY_DOES_NOT_EXIST | Key not found |
| 0x03 | SUCCESS_WITH_PREVIOUS | Success, previous value returned |
| 0x04 | NOT_EXECUTED_WITH_PREVIOUS | Not executed, previous value returned |
| 0x81 | INVALID_MAGIC_OR_MESSAGE_ID | Protocol error |
| 0x82 | UNKNOWN_COMMAND | Unknown opcode |
| 0x83 | UNKNOWN_VERSION | Unsupported protocol version |
| 0x84 | REQUEST_PARSING_ERROR | Malformed request |
| 0x85 | SERVER_ERROR | Internal server error |
| 0x86 | COMMAND_TIMEOUT | Operation timed out |
| 0x87 | NODE_SUSPECTED | Server node is suspected failed |
| 0x88 | ILLEGAL_LIFECYCLE_STATE | Server not ready |

**Success Range**: 0x00-0x7F  
**Error Range**: 0x80-0xFF

### Topology Change Marker

Indicates if server's topology has changed.

**Type**: uint8  
**Values**:
- `0x00` = No topology change
- `0x01` = Topology changed (followed by topology data)

**For Step 2**: Just parse this byte, ignore topology data if present (Step 11)

---

## Wire Format Examples

### Example 1: Minimal PING Request

```
Request Header:
0xA0              Magic byte (request)
0x01              Message ID (1)
0x28              Version (Protocol 4.0)
0x17              Opcode (PING = 0x17)
0x00              Cache name (empty string, default cache)
0x00              Flags (none)
0x01              Client intelligence (BASIC)
0x00              Topology ID (0)

Total: 8 bytes
Hex: A0 01 28 17 00 00 01 00
```

### Example 2: PING Response

```
Response Header:
0xA1              Magic byte (response)
0x01              Message ID (1, matching request)
0x18              Opcode (PING_RESPONSE = 0x18)
0x00              Status (NO_ERROR)
0x00              Topology change marker (no change)

Total: 5 bytes
Hex: A1 01 18 00 00
```

### Example 3: GET Request for "myCache"

```
Request Header:
0xA0              Magic byte
0x02              Message ID (2)
0x28              Version (4.0)
0x03              Opcode (GET)
0x07              Cache name length (7 bytes)
6D 79 43 61 63 68 65  "myCache" in UTF-8
0x00              Flags
0x01              Client intelligence (BASIC)
0x00              Topology ID

Total: 15 bytes
Hex: A0 02 28 03 07 6D 79 43 61 63 68 65 00 01 00
```

---

## C++17 Implementation Guide

### Request Header Structure

```cpp
struct RequestHeader {
    uint8_t magic = 0xA0;           // Always 0xA0
    uint64_t messageId;              // Unique request ID
    uint8_t version = 0x28;          // Protocol 4.0
    uint8_t opcode;                  // Operation code
    std::string cacheName;           // Cache name (can be empty)
    uint32_t flags = 0;              // Request flags
    uint8_t clientIntelligence = 0x01; // BASIC intelligence
    uint32_t topologyId = 0;         // Known topology version
};
```

### Response Header Structure

```cpp
struct ResponseHeader {
    uint8_t magic;                   // Should be 0xA1
    uint64_t messageId;              // Matching request
    uint8_t opcode;                  // Response operation
    uint8_t status;                  // Status code
    uint8_t topologyChangeMarker;    // 0 or 1
};
```

### Encoding Request Header

```cpp
void encodeRequestHeader(std::vector<uint8_t>& buffer, const RequestHeader& header) {
    // Magic byte
    buffer.push_back(header.magic);
    
    // Message ID (vLong)
    Codec::writeVLong(buffer, header.messageId);
    
    // Version
    buffer.push_back(header.version);
    
    // Opcode
    buffer.push_back(header.opcode);
    
    // Cache name (string)
    Codec::writeString(buffer, header.cacheName);
    
    // Flags (vInt)
    Codec::writeVInt(buffer, header.flags);
    
    // Client intelligence
    buffer.push_back(header.clientIntelligence);
    
    // Topology ID (vInt)
    Codec::writeVInt(buffer, header.topologyId);
}
```

### Decoding Response Header

```cpp
ResponseHeader decodeResponseHeader(const uint8_t*& ptr) {
    ResponseHeader header;
    
    // Magic byte
    header.magic = *ptr++;
    
    // Validate magic
    if (header.magic != 0xA1) {
        throw std::runtime_error("Invalid response magic byte");
    }
    
    // Message ID (vLong)
    header.messageId = Codec::readVLong(ptr);
    
    // Opcode
    header.opcode = *ptr++;
    
    // Status
    header.status = *ptr++;
    
    // Topology change marker
    header.topologyChangeMarker = *ptr++;
    
    // TODO (Step 11): Parse topology data if marker == 1
    
    return header;
}
```

---

## Java Reference

**File**: `org.infinispan.client.hotrod.impl.protocol.Codec30` (lines 156-173)

```java
protected void writeHeader(ByteBuf buf, long messageId, ClientTopology clientTopology, 
                          HotRodOperation<?> operation, byte version) {
    buf.writeByte(HotRodConstants.REQUEST_MAGIC);      // 0xA0
    ByteBufUtil.writeVLong(buf, messageId);
    buf.writeByte(version);
    buf.writeByte(operation.requestOpCode());
    ByteBufUtil.writeArray(buf, operation.getCacheNameBytes());
    ByteBufUtil.writeVInt(buf, operation.flags());
    buf.writeByte(clientTopology.getClientIntelligence().getValue());
    ByteBufUtil.writeVInt(buf, clientTopology.getTopologyId());
    writeDataTypes(buf, operation.getDataFormat());    // Skip for now
}
```

**Constants**: `org.infinispan.client.hotrod.impl.protocol.HotRodConstants`

---

## Testing Strategy

### Unit Tests

1. **Request Header Encoding**:
   - Encode with various opcodes
   - Encode with empty vs named cache
   - Encode with different message IDs
   - Verify byte output matches expected

2. **Response Header Decoding**:
   - Decode success response
   - Decode error response
   - Decode with topology change marker
   - Validate magic byte check
   - Verify message ID extracted correctly

3. **Round-Trip**:
   - Not applicable (client sends requests, receives responses)
   - But test request encoding → manual response → decode response

4. **Edge Cases**:
   - Large message IDs (test vLong encoding)
   - Long cache names (test string encoding)
   - All status codes
   - Invalid magic bytes (should throw)

### Test Vectors

See `test-vectors/step-02-headers/`:
- `request-header-test-cases.json` - Various request headers
- `response-header-test-cases.json` - Various response headers

### Validation

- Compare encoded requests byte-for-byte with test vectors
- Decode test vector responses and verify fields
- Use Wireshark (Step 3+) to compare with real traffic

---

## Common Pitfalls

### 1. Magic Byte Validation

❌ **Don't**: Silently accept wrong magic byte  
✅ **Do**: Throw exception if magic != 0xA1 for responses

### 2. Message ID Correlation

❌ **Don't**: Ignore message ID in responses  
✅ **Do**: Verify response message ID matches request

### 3. String Encoding for Cache Name

❌ **Don't**: Write cache name as raw bytes  
✅ **Do**: Use length-prefixed string encoding (vInt + UTF-8)

### 4. Opcode Pairing

❌ **Don't**: Expect exact opcode match in response  
✅ **Do**: Response opcode usually = request opcode + 1

### 5. Status Code Interpretation

❌ **Don't**: Treat all non-zero status as error  
✅ **Do**: Codes 0x00-0x7F = success variants, 0x80+ = errors

---

## Implementation Checklist

- [ ] RequestHeader struct/class
- [ ] ResponseHeader struct/class
- [ ] Request header encoding
- [ ] Response header decoding
- [ ] Magic byte validation
- [ ] Unit tests for request encoding
- [ ] Unit tests for response decoding
- [ ] Test vectors validated
- [ ] Error handling for invalid magic/status

---

## Next Step

Once all tests pass:
- ✅ Mark Step 2 complete in `PROGRESS.md`
- ➡️ Proceed to **Step 3: PING Operation** (first network operation!)

## References

- **Java Implementation**: `Codec30.java`, `Codec40.java`
- **Kaitai Schema**: `hotrod40.ksy` lines 315-394
- **Operation Codes**: `HotRodConstants.java`
- **Protocol Spec**: https://infinispan.org/docs/stable/titles/hotrod_protocol/
