# Step 1 Test Vectors: Wire Format Primitives

This directory contains **language-agnostic test vectors** for validating wire format primitive ports.

## Test Files

| File | Description |
|------|-------------|
| `vint-test-cases.json` | Variable-length 32-bit unsigned integer (vInt) |
| `vlong-test-cases.json` | Variable-length 64-bit unsigned integer (vLong) |
| `signed-vint-test-cases.json` | Signed vInt with ZigZag encoding |
| `string-test-cases.json` | Length-prefixed UTF-8 strings |
| `bytes-test-cases.json` | Length-prefixed byte arrays |

## Usage

### Reading Test Cases

Each JSON file contains an array of test cases with:
- `description` - What the test case covers
- `value` - Input value to encode
- `expected_hex` - Expected output as hex string (human-readable)
- `expected_bytes` - Expected output as byte array (for programmatic testing)

### Validation Approach

**Option 1: Unit Tests**
```
For each test case:
  1. Encode the input value
  2. Compare output bytes to expected_bytes
  3. Decode the output bytes
  4. Compare decoded value to original input value (round-trip)
```

**Option 2: Cross-Validation with Java**

Create a Java program using Infinispan's codec:
```java
import org.infinispan.client.hotrod.impl.transport.netty.ByteBufUtil;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;

ByteBuf buf = Unpooled.buffer();
ByteBufUtil.writeVInt(buf, 128);
// Output: [0x80, 0x01]
```

Compare your port's output byte-for-byte.

## Test Coverage

### vInt Test Cases
- Zero and small values (0-127, single byte)
- Two-byte values (128-16383)
- Three-byte values (16384-2097151)
- Four-byte values (2097152-268435455)
- Five-byte values (268435456-4294967295)
- Edge cases: 127→128, 16383→16384, etc.

### vLong Test Cases
- Same as vInt for values fitting in uint32
- Values beyond uint32 (5-9 bytes)
- Large values (trillions, max int64)

### Signed vInt (ZigZag) Test Cases
- Zero
- Small positive and negative values (-10 to +10)
- Boundary cases (±64, ±128, ±1000)
- Extreme values (max/min int32)

### String Test Cases
- Empty string
- ASCII strings (single-byte chars)
- Multi-byte UTF-8 (2-byte: café, 3-byte: 世界, 4-byte: 🔥)
- Long strings (length > 127 bytes)

### Byte Array Test Cases
- Empty array
- Small arrays (1-16 bytes)
- Arrays with various byte values (0x00, 0xFF, etc.)
- Large arrays (length > 127)

## Validation Criteria

✅ **Pass Criteria**:
- All test cases encode to expected bytes
- All test cases decode correctly (round-trip)
- Output matches Java implementation byte-for-byte

❌ **Common Failures**:
- Off-by-one in shift operations
- Wrong string encoding (not UTF-8)
- Missing continuation bit in vInt/vLong
- Incorrect ZigZag encoding

## Cross-Language Validation

These test vectors are designed to validate ports in **any language**:

```
C++:      std::vector<uint8_t> matches expected_bytes
C#:       byte[] matches expected_bytes
Python:   bytes() matches expected_bytes
Go:       []byte matches expected_bytes
Rust:     Vec<u8> matches expected_bytes
```

## Reference Implementations

### Java
- `org.infinispan.client.hotrod.impl.transport.netty.ByteBufUtil`
- `org.infinispan.commons.io.SignedNumeric`

### Kaitai Struct
- `../../../hotrod-dissector/schemas/hotrod40.ksy` (lines 186-265)

## Running Tests

### Example: C++17 with Google Test

```cpp
#include <gtest/gtest.h>
#include "Codec.h"
#include <nlohmann/json.hpp>  // for reading JSON test vectors

TEST(CodecTest, VIntEncoding) {
    auto testVectors = loadJsonFile("test-vectors/step-01-primitives/vint-test-cases.json");
    
    for (const auto& testCase : testVectors["test_cases"]) {
        uint32_t value = testCase["value"];
        std::vector<uint8_t> expected = testCase["expected_bytes"];
        
        std::vector<uint8_t> encoded;
        Codec::writeVInt(encoded, value);
        
        EXPECT_EQ(encoded, expected) << "Failed for value: " << value;
    }
}
```

### Example: C# with NUnit

```csharp
[Test]
public void TestVIntEncoding()
{
    var testVectors = LoadJsonFile("test-vectors/step-01-primitives/vint-test-cases.json");
    
    foreach (var testCase in testVectors["test_cases"])
    {
        uint value = testCase["value"];
        byte[] expected = testCase["expected_bytes"].ToObject<byte[]>();
        
        byte[] encoded = Codec.WriteVInt(value);
        
        CollectionAssert.AreEqual(expected, encoded, $"Failed for value: {value}");
    }
}
```

## Next Steps

Once all tests pass:
- ✅ Mark Step 1 complete in PROGRESS.md
- ➡️ Move to Step 2: Protocol Headers

## Questions?

Refer to:
- `../../docs/01-wire-format-primitives.md` for detailed explanations
- Java source code for reference implementation
- Kaitai schema for formal protocol specification
