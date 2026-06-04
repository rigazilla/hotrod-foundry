# Step 1: Wire Format Primitives

**Status**: Foundation  
**Prerequisite**: None (first implementation step)  
**Estimated Time**: 2-3 hours  
**Test Vectors**: `../test-vectors/step-01-primitives/`

## Overview

Hot Rod protocol uses **variable-length encoding** for integers and **length-prefixed encoding** for strings and byte arrays. This step implements the basic building blocks for all protocol messages.

**No network I/O in this step** - pure encoding/decoding functions with byte buffers.

## Why Variable-Length Encoding?

Most values in Hot Rod are small (message IDs, lengths, flags). Variable-length encoding saves space:
- `0-127` → 1 byte (vs 4 bytes for fixed int32)
- `128-16383` → 2 bytes
- Larger values use more bytes as needed

## Primitives to Implement

1. **vInt** - Variable-length 32-bit unsigned integer (1-5 bytes)
2. **vLong** - Variable-length 64-bit unsigned integer (1-9 bytes)
3. **Signed vInt** - Signed 32-bit integer with ZigZag encoding
4. **String** - Length-prefixed UTF-8 string
5. **Byte Array** - Length-prefixed raw bytes

---

## 1. Variable-Length Integer (vInt)

### Encoding Algorithm

Each byte stores 7 bits of data + 1 continuation bit:
- **Bit 7** (0x80): Continuation flag (1 = more bytes follow, 0 = last byte)
- **Bits 0-6** (0x7F): Data payload

**Algorithm**:
```
while (value has bits beyond 7 remaining) {
    write: (value & 0x7F) | 0x80   // 7 data bits + continuation bit
    value >>= 7                     // shift right 7 bits
}
write: value & 0x7F                 // final byte, no continuation bit
```

### Encoding Examples

| Value | Binary Breakdown | Hex Bytes | Explanation |
|-------|-----------------|-----------|-------------|
| 0 | `0000000` | `00` | Fits in 7 bits, no continuation |
| 127 | `1111111` | `7F` | Max value in 1 byte |
| 128 | `10000000` | `80 01` | Byte 1: `00000000\|1`, Byte 2: `00000001` |
| 255 | `11111111` | `FF 01` | Byte 1: `01111111\|1`, Byte 2: `00000001` |
| 256 | `100000000` | `80 02` | Byte 1: `00000000\|1`, Byte 2: `00000010` |
| 300 | `100101100` | `AC 02` | Byte 1: `00101100\|1`, Byte 2: `00000010` |
| 16383 | `11111111111111` | `FF 7F` | Byte 1: `01111111\|1`, Byte 2: `01111111` |
| 16384 | `100000000000000` | `80 80 01` | 3 bytes needed |

### Decoding Algorithm

```
result = 0
shift = 0
for each byte {
    result |= (byte & 0x7F) << shift
    if ((byte & 0x80) == 0) break   // no continuation bit, done
    shift += 7
}
return result
```

### C++17 Reference Implementation

```cpp
void Codec::writeVInt(std::vector<uint8_t>& buffer, uint32_t value) {
    while ((value & ~0x7F) != 0) {
        buffer.push_back((uint8_t)((value & 0x7F) | 0x80));
        value >>= 7;
    }
    buffer.push_back((uint8_t)value);
}

uint32_t Codec::readVInt(const uint8_t*& ptr) {
    uint8_t b = *ptr++;
    uint32_t result = b & 0x7F;
    
    int shift = 7;
    while ((b & 0x80) != 0) {
        b = *ptr++;
        result |= (b & 0x7F) << shift;
        shift += 7;
    }
    
    return result;
}
```

### Java Reference

**File**: `org.infinispan.client.hotrod.impl.transport.netty.ByteBufUtil`

```java
// Write vInt (lines 70-76)
public static void writeVInt(ByteBuf buf, int i) {
    while ((i & ~0x7F) != 0) {
        buf.writeByte((byte) ((i & 0x7f) | 0x80));
        i >>>= 7;
    }
    buf.writeByte((byte) i);
}

// Read vInt (lines 104-112)
public static int readVInt(ByteBuf buf) {
    byte b = buf.readByte();
    int i = b & 0x7F;
    for (int shift = 7; (b & 0x80) != 0; shift += 7) {
        b = buf.readByte();
        i |= (b & 0x7FL) << shift;
    }
    return i;
}
```

---

## 2. Variable-Length Long (vLong)

**Same algorithm as vInt, but for 64-bit values** (up to 9 bytes).

### Encoding Examples

| Value | Hex Bytes |
|-------|-----------|
| 0 | `00` |
| 255 | `FF 01` |
| 4294967295 (max uint32) | `FF FF FF FF 0F` |
| 4294967296 (uint32 + 1) | `80 80 80 80 10` |

### C++17 Reference Implementation

```cpp
void Codec::writeVLong(std::vector<uint8_t>& buffer, uint64_t value) {
    while ((value & ~0x7F) != 0) {
        buffer.push_back((uint8_t)((value & 0x7F) | 0x80));
        value >>= 7;
    }
    buffer.push_back((uint8_t)value);
}

uint64_t Codec::readVLong(const uint8_t*& ptr) {
    uint8_t b = *ptr++;
    uint64_t result = b & 0x7F;
    
    int shift = 7;
    while ((b & 0x80) != 0) {
        b = *ptr++;
        result |= (uint64_t)(b & 0x7F) << shift;
        shift += 7;
    }
    
    return result;
}
```

### Java Reference

**File**: `org.infinispan.client.hotrod.impl.transport.netty.ByteBufUtil` (lines 82-102)

---

## 3. Signed Variable-Length Integer (ZigZag Encoding)

Hot Rod uses **ZigZag encoding** for signed integers (same as Protocol Buffers).

### Why ZigZag?

Standard variable-length encoding is inefficient for negative numbers:
- `-1` as int32 = `0xFFFFFFFF` → would encode as 5 bytes!

ZigZag maps signed integers to unsigned so small negatives encode efficiently:

| Signed Value | ZigZag Encoded | Result |
|--------------|----------------|--------|
| 0 | 0 | `00` |
| -1 | 1 | `01` |
| 1 | 2 | `02` |
| -2 | 3 | `03` |
| 2 | 4 | `04` |
| -64 | 127 | `7F` |
| 64 | 128 | `80 01` |

### ZigZag Encoding Formula

```
encoded = (value << 1) ^ (value >> 31)
```

**How it works**:
- Positive numbers: `value << 1` (multiply by 2)
- Negative numbers: `(value << 1) ^ -1` (multiply by 2, then flip all bits)

### ZigZag Decoding Formula

```
decoded = (encoded >>> 1) ^ -(encoded & 1)
```

Or equivalently:
```
decoded = (encoded & 1) == 0 ? encoded >>> 1 : ~(encoded >>> 1)
```

### C++17 Reference Implementation

```cpp
uint32_t Codec::zigzagEncode(int32_t value) {
    return (value << 1) ^ (value >> 31);
}

int32_t Codec::zigzagDecode(uint32_t value) {
    return (value >> 1) ^ -(value & 1);
}

void Codec::writeSignedVInt(std::vector<uint8_t>& buffer, int32_t value) {
    writeVInt(buffer, zigzagEncode(value));
}

int32_t Codec::readSignedVInt(const uint8_t*& ptr) {
    return zigzagDecode(readVInt(ptr));
}
```

### Java Reference

**File**: `org.infinispan.commons.io.SignedNumeric` (lines 41-47)

```java
public static int encode(int vint) {
    return (vint << 1) ^ (vint >> 31);
}

public static int decode(int vint) {
    return (vint & 1) == 0 ? vint >>> 1 : ~(vint >>> 1);
}
```

---

## 4. Length-Prefixed String

Strings in Hot Rod are UTF-8 encoded with a vInt length prefix.

### Encoding Format

```
[vInt: length in bytes] [UTF-8 bytes]
```

### Encoding Algorithm

```
1. Convert string to UTF-8 bytes
2. Write byte count as vInt
3. Write UTF-8 bytes
```

### Encoding Examples

| String | UTF-8 Bytes | Encoded Hex |
|--------|-------------|-------------|
| `""` (empty) | - | `00` |
| `"a"` | `61` | `01 61` |
| `"hello"` | `68 65 6C 6C 6F` | `05 68 65 6C 6C 6F` |
| `"café"` | `63 61 66 C3 A9` | `05 63 61 66 C3 A9` |
| `"🔥"` | `F0 9F 94 A5` | `04 F0 9F 94 A5` |

**Important**: Length is **byte count**, not character count!  
`"café"` = 4 characters, 5 bytes (é = 2 bytes in UTF-8)

### C++17 Reference Implementation

```cpp
void Codec::writeString(std::vector<uint8_t>& buffer, const std::string& str) {
    // std::string in C++ is already bytes (not necessarily UTF-8, ensure UTF-8!)
    writeVInt(buffer, str.size());
    buffer.insert(buffer.end(), str.begin(), str.end());
}

std::string Codec::readString(const uint8_t*& ptr) {
    uint32_t length = readVInt(ptr);
    std::string result(reinterpret_cast<const char*>(ptr), length);
    ptr += length;
    return result;
}
```

### Java Reference

**File**: `org.infinispan.client.hotrod.impl.transport.netty.ByteBufUtil` (lines 26-36)

```java
public static String readString(ByteBuf buf) {
    byte[] strContent = readArray(buf);
    return new String(strContent, HotRodConstants.HOTROD_STRING_CHARSET);  // UTF-8
}

public static void writeString(ByteBuf buf, String string) {
    if (string != null && !string.isEmpty()) {
        writeArray(buf, string.getBytes(HotRodConstants.HOTROD_STRING_CHARSET));
    } else {
        writeVInt(buf, 0);
    }
}
```

---

## 5. Length-Prefixed Byte Array

Identical to strings, but raw bytes (no character encoding).

### Encoding Format

```
[vInt: length] [raw bytes]
```

### Encoding Examples

| Byte Array | Encoded Hex |
|------------|-------------|
| `[]` (empty) | `00` |
| `[0x01]` | `01 01` |
| `[0x01, 0x02, 0x03]` | `03 01 02 03` |
| `[0xFF, 0xFE, 0xFD]` | `03 FF FE FD` |

### C++17 Reference Implementation

```cpp
void Codec::writeBytes(std::vector<uint8_t>& buffer, const std::vector<uint8_t>& bytes) {
    writeVInt(buffer, bytes.size());
    buffer.insert(buffer.end(), bytes.begin(), bytes.end());
}

std::vector<uint8_t> Codec::readBytes(const uint8_t*& ptr) {
    uint32_t length = readVInt(ptr);
    std::vector<uint8_t> result(ptr, ptr + length);
    ptr += length;
    return result;
}
```

### Java Reference

**File**: `org.infinispan.client.hotrod.impl.transport.netty.ByteBufUtil` (lines 19-54)

```java
public static byte[] readArray(ByteBuf buf) {
    int length = readVInt(buf);
    byte[] bytes = new byte[length];
    buf.readBytes(bytes, 0, length);
    return bytes;
}

public static void writeArray(ByteBuf buf, byte[] toAppend) {
    writeVInt(buf, toAppend.length);
    buf.writeBytes(toAppend);
}
```

---

## Testing Strategy

### Unit Tests

For each primitive, test:

1. **Zero values**: `0`, empty string, empty array
2. **Boundary values**: `127`, `128`, `255`, `256`, `16383`, `16384`
3. **Large values**: Max int32, max uint32, max int64
4. **Negative values** (for signed vInt): `-1`, `-127`, `-128`, min int32
5. **UTF-8 edge cases**: Empty, ASCII, multi-byte chars, emoji
6. **Round-trip**: encode then decode, should equal original

### Test Vectors

Use the provided test vectors in `../test-vectors/step-01-primitives/`:

- `vint-test-cases.json` - Input values and expected hex output
- `vlong-test-cases.json`
- `signed-vint-test-cases.json`
- `string-test-cases.json`
- `bytes-test-cases.json`

### Cross-Validation with Java

Create a small Java program to:
1. Encode test values using Infinispan's codec
2. Print hex output
3. Compare byte-for-byte with your implementation

**Example validation**:
```bash
# Java outputs: 0x80 0x01 for value 128
# Your implementation must output: 0x80 0x01
```

---

## Common Pitfalls

### 1. Shift Operator Edge Cases

**C/C++**: Use unsigned right shift (`>>`) for unsigned values  
**Java**: Use unsigned right shift (`>>>`) vs signed shift (`>>`)

```cpp
// C++: OK for unsigned types
uint32_t value = 0x80;
value >>= 7;  // Result: 0x01

// Java: Must use >>> for unsigned shift
int value = 0x80;
value >>>= 7;  // Result: 0x01 (unsigned)
value >>= 7;   // Result: 0x01 (in this case, but use >>> for clarity)
```

### 2. String Encoding

**Always use UTF-8**, not platform default encoding!

```cpp
// C++: std::string is just bytes, ensure UTF-8 input
// C#: Use Encoding.UTF8.GetBytes(str)
// Python: str.encode('utf-8')
// Java: string.getBytes(StandardCharsets.UTF_8)
```

### 3. Empty Strings/Arrays

Empty string encodes as `0x00` (length = 0), not as nothing!

### 4. Signed vs Unsigned

vInt/vLong are **unsigned**. Use ZigZag encoding for signed values.

---

## Implementation Checklist

- [ ] vInt encode/decode
- [ ] vLong encode/decode
- [ ] Signed vInt (ZigZag) encode/decode
- [ ] String encode/decode (UTF-8)
- [ ] Byte array encode/decode
- [ ] Unit tests for all primitives
- [ ] Test vectors validated
- [ ] Round-trip tests pass
- [ ] Cross-validation with Java encoder output

---

## Next Step

Once all tests pass and you've validated against Java output:
- ✅ Mark Step 1 complete in `PROGRESS.md`
- ➡️ Proceed to **Step 2: Protocol Headers**

## References

- **Java Implementation**: `ByteBufUtil.java`, `SignedNumeric.java`
- **Kaitai Schema**: `hotrod40.ksy` lines 186-265
- **Protocol Buffers Encoding**: https://developers.google.com/protocol-buffers/docs/encoding
- **ZigZag Encoding**: https://en.wikipedia.org/wiki/Variable-length_quantity#Zigzag_encoding
