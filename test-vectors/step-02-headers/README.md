# Step 2 Test Vectors: Protocol Headers

This directory contains test vectors for validating Hot Rod protocol header ports.

## ⚠️ CRITICAL WARNING: Protocol 4.0 Requires 11 Fields!

**DO NOT use `request-header-test-cases.json` for Protocol 4.0 implementations!**

The original test vectors show **incomplete** headers with only **8 fields**. Protocol 4.0 requires **11 fields**. Missing the 3 conditional fields causes **server timeout**!

**Use these files instead:**
- ✅ `request-header-protocol40-complete.json` - COMPLETE Protocol 4.0 (11 fields)
- ✅ `ping-protocol40-complete.json` - COMPLETE PING operation headers

## Test Files

| File | Status | Description |
|------|--------|-------------|
| `request-header-protocol40-complete.json` | ✅ **USE THIS** | COMPLETE Protocol 4.0 request headers (11 fields) |
| `request-header-test-cases.json` | ⚠️ **INCOMPLETE** | OLD - Missing 3 fields for Protocol 4.0 |
| `response-header-test-cases.json` | ✅ OK | Response header decoding test cases |

## Request Header Format (Protocol 4.0 - COMPLETE)

**All 11 fields required:**

```
1.  Magic (0xA0):          1 byte
2.  Message ID:            vLong
3.  Version:               1 byte (0x28 = 40 = Protocol 4.0)
4.  Opcode:                1 byte
5.  Cache Name:            string (vInt length + UTF-8)
6.  Flags:                 vInt
7.  Client Intelligence:   1 byte (0x01, 0x02, or 0x03)
8.  Topology ID:           vInt
9.  Key Media Type:        1 byte (REQUIRED if version >= 0x28) ⚠️
10. Value Media Type:      1 byte (REQUIRED if version >= 0x28) ⚠️
11. Other Param Count:     vInt (REQUIRED if version >= 40) ⚠️
```

**Fields 9-11 are conditional but ALWAYS present for Protocol 4.0!**

### Why These Fields Are Required

From `hotrod40.ksy` lines 339-349:
```yaml
- id: key_media_type
  if: version >= 28    # 0x28 = 40 decimal, so REQUIRED for Protocol 4.0
  
- id: value_media_type
  if: version >= 28    # REQUIRED for Protocol 4.0
  
- id: other_param_count
  if: version >= 40    # REQUIRED for Protocol 4.0 (40 decimal)
```

Since Protocol 4.0 has version byte `0x28` (40 decimal):
- `version >= 28` (0x1C) is **TRUE** → media types required
- `version >= 40` is **TRUE** → other_param_count required

## Response Header Format

```
Magic (0xA1):           1 byte
Message ID:             vLong (matching request)
Opcode:                 1 byte
Status:                 1 byte
Topology Change Marker: 1 byte (0 = no change, 1 = changed)
```

## Usage

### Request Header Tests

For each test case:
1. Create `RequestHeader` with specified fields
2. Encode to bytes
3. Compare with `expected_bytes`
4. All bytes must match exactly

### Response Header Tests

For each test case:
1. Use `expected_bytes` as input
2. Decode to `ResponseHeader`
3. Verify all fields match `header` object
4. Validate magic byte is 0xA1

### Validation

```cpp
// Request encoding test
RequestHeader req;
req.messageId = 1;
req.version = 0x28;
req.opcode = 0x17;  // PING
req.cacheName = "";
req.flags = 0;
req.clientIntelligence = 0x01;
req.topologyId = 0;

std::vector<uint8_t> encoded;
encodeRequestHeader(encoded, req);

// Expected: [0xA0, 0x01, 0x28, 0x17, 0x00, 0x00, 0x01, 0x00]
```

```cpp
// Response decoding test
std::vector<uint8_t> bytes = {0xA1, 0x01, 0x18, 0x00, 0x00};
const uint8_t* ptr = bytes.data();

ResponseHeader resp = decodeResponseHeader(ptr);

// Verify:
// resp.magic == 0xA1
// resp.messageId == 1
// resp.opcode == 0x18 (PING_RESPONSE)
// resp.status == 0x00 (NO_ERROR)
// resp.topologyChangeMarker == 0
```

## Test Coverage

### Request Headers

- **Minimal header** (PING with defaults)
- **Named cache** (GET with "myCache")
- **Flags set** (PUT with FORCE_RETURN_VALUE)
- **Large message IDs** (test vLong encoding)
- **Various opcodes** (PUT, GET, REMOVE, STATS, etc.)
- **Topology ID** (non-zero topology tracking)
- **Client intelligence** (BASIC, HASH_DISTRIBUTION_AWARE)
- **Protocol versions** (4.0 and 4.1)

### Response Headers

- **Success responses** (NO_ERROR status)
- **Not found** (KEY_DOES_NOT_EXIST)
- **Success with previous** (returning old value)
- **Errors** (UNKNOWN_COMMAND, SERVER_ERROR, etc.)
- **Topology changes** (marker = 1)
- **Large message IDs** (vLong decoding)
- **Various opcodes** (matching request types)

## Common Operations

### Operation Code Mapping

| Request Opcode | Request Name | Response Opcode | Response Name |
|----------------|--------------|-----------------|---------------|
| 0x01 | PUT | 0x02 | PUT_RESPONSE |
| 0x03 | GET | 0x04 | GET_RESPONSE |
| 0x0B | REMOVE | 0x0C | REMOVE_RESPONSE |
| 0x17 | PING | 0x18 | PING_RESPONSE |
| 0x1B | GET_WITH_METADATA | 0x1C | GET_WITH_METADATA_RESPONSE |

**Pattern**: Response opcode = Request opcode + 1 (except ERROR = 0x50)

### Status Code Interpretation

**Success** (0x00-0x7F):
- 0x00: NO_ERROR
- 0x01: NOT_PUT_REMOVED_REPLACED (operation didn't apply)
- 0x02: KEY_DOES_NOT_EXIST
- 0x03: SUCCESS_WITH_PREVIOUS (old value returned)

**Errors** (0x80+):
- 0x81: INVALID_MAGIC_OR_MESSAGE_ID
- 0x82: UNKNOWN_COMMAND
- 0x83: UNKNOWN_VERSION
- 0x85: SERVER_ERROR
- 0x86: COMMAND_TIMEOUT

## Error Cases to Test

1. **Invalid magic byte** in response → should throw exception
2. **Mismatched message ID** → caller should detect
3. **Error status codes** → should be handled appropriately
4. **Topology change marker** → acknowledge but skip topology data (Step 11)

## Cross-Language Validation

These test vectors work for any language:

**C++**:
```cpp
EXPECT_EQ(encoded, expected_bytes);
EXPECT_EQ(header.status, expected_status);
```

**C#**:
```csharp
CollectionAssert.AreEqual(encoded, expectedBytes);
Assert.AreEqual(header.Status, expectedStatus);
```

**Python**:
```python
assert encoded == expected_bytes
assert header.status == expected_status
```

## Next Steps

After Step 2 validation:
- Headers can encode/decode correctly
- Ready for **Step 3: PING Operation** (first network I/O!)
- PING will use these headers over TCP connection

## References

- **Documentation**: `../../docs/02-protocol-headers.md`
- **Java Implementation**: `Codec30.java`, `Codec40.java`
- **Kaitai Schema**: `hotrod40.ksy` (request_header, response_header types)
