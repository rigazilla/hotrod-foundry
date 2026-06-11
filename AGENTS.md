# Porting the Hot Rod Java Client to Another Language

**Audience**: AI assistants (Claude, GPT-4, etc.) tasked with porting the Infinispan Hot Rod Java client in any programming language.

**Purpose**: This document explains how to use the `hotrod-foundry` documentation repository to generate a complete, production-ready Hot Rod client from scratch.

This repository documents how to port the Java Hot Rod client into another language. Each target project should be a language-specific port of the Java reference client that preserves protocol semantics and wire-format behavior (byte-for-byte where required).

---

## ⚠️ CRITICAL: Before You Start

### 1. DO NOT Write Code in hotrod-foundry/

This repository contains **documentation only**. Your port must be in a **separate directory**.

**First action**: Create a new directory OUTSIDE of hotrod-foundry:
```bash
cd ..  # Exit hotrod-foundry
mkdir [language]-client
cd [language]-client
# NOW you can start coding
```

If you've already written code in hotrod-foundry/, **STOP** and move it to a separate directory before continuing.

### 2. DO NOT Search for Existing Ports

**DO NOT look for existing Hot Rod client ports in your target language unless explicitly requested by the user.**

**What to do instead**:
1. ✅ Study the **Java** implementation (reference)
2. ✅ Read hotrod-foundry documentation
3. ✅ Use test vectors for validation
4. ✅ Port step-by-step from ROADMAP
5. ✅ Write code yourself based on Java logic

---

## Quick Start

When a user asks you to "port the Hot Rod client to [language]", follow this sequence:

1. **Read this file completely** (you're doing it now ✓)
2. **Clone Java reference implementation** - `git clone https://github.com/infinispan/infinispan.git` if it's not already available locally
3. **Read `ROADMAP.md`** to understand the porting steps
4. **Read `docs/development-guidelines.md`** for process requirements
5. **Present your porting plan to the user** - WAIT for approval before coding
7. **Follow the roadmap step-by-step** (don't skip ahead)
8. **Use Java code as reference** for each step (MANDATORY)
9. **Use test vectors for validation** (not live server initially)
10. **Track progress in PROGRESS.md** (update after each step)

### ⚠️ Before Writing ANY Code

**MANDATORY**: Present your porting plan and get user approval first.

**What to include in your porting plan**:
```markdown
## Porting Plan: Hot Rod Client for [Language]

### Project Setup
- Repository name: [language]-client
- Location: /path/to/[language]-client (NOT in hotrod-foundry/)
- Build system: [CMake/Gradle/npm/etc.]
- Test framework: [Google Test/xUnit/pytest/etc.]
- **Platform support**: Linux (Fedora/RHEL) + Windows (MANDATORY)

### Porting Approach
- Following ROADMAP.md steps 0-8
- Test-driven development with test vectors

### Steps I'll Port
1. Step 0: Foundation Setup
2. Step 1: Wire Format Primitives
3. Step 2: Protocol Headers
4. Step 3: Authentication (SCRAM-SHA-256)
5. Step 4: Topology Awareness
6. Step 5: Consistent Hashing
7. Step 6: PING Operation
8. [Optional] Step 7+: GET, PUT operations

### Deliverables
- Source code in [language]-client/
- Cross-platform support: Linux (Fedora/RHEL) + Windows
- Unit tests with 80%+ coverage
- Integration tests against Infinispan 16.0
- PROGRESS.md tracking
- CI/CD pipeline (test on both Linux and Windows)
- README with usage examples

**Ready to proceed? [WAIT FOR USER APPROVAL]**
```

**DO NOT start coding until the user approves your porting plan.**

### If User Modifies or Rejects Your Plan

**User says "Only port Steps 0-2 for now":**
- ✅ Update your plan accordingly
- ✅ Present revised plan for approval
- ✅ Proceed with reduced scope

**User says "Use pytest instead of unittest":**
- ✅ Update test framework choice
- ✅ Confirm change with user
- ✅ Proceed

**User says "Start with Step 1, skip Step 0":**
- ⚠️ Explain that Step 0 (separate directory, CI/CD setup) is mandatory
- ⚠️ Suggest minimal Step 0 if they want to move fast
- ✅ Wait for confirmation

---

## Core Principles

### 1. Cross-Platform Support (MANDATORY)

**REQUIRED**: All ports must support both Linux and Windows.

**Linux**:
- Primary targets: Fedora, RHEL (Red Hat Enterprise Linux)
- Should also work on Ubuntu/Debian

**Windows**:
- Windows 10/11
- Visual Studio compatibility (for C/C++)

**Requirements**:
- ✅ No platform-specific code without fallbacks
- ✅ Use cross-platform libraries (e.g., OpenSSL available on both)
- ✅ Build system works on both (CMake, Gradle, npm, etc.)
- ✅ Documentation includes setup for both platforms

**Common pitfalls**:
- ❌ Using POSIX-only APIs without Windows alternatives
- ❌ Hard-coded path separators (`/` vs `\`)
- ❌ Linux-only dependencies
- ❌ Testing only on one platform

### 2. Infrastructure Before Operations

**CRITICAL**: Steps 1-5 build the foundation. Steps 6+ add operations.

```
Foundation (Steps 1-5):
  Step 1: Wire Format Primitives  → Encoding/decoding basics
  Step 2: Protocol Headers         → Request/response structure  
  Step 3: Authentication           → SASL SCRAM-SHA-256
  Step 4: Topology Awareness       → Cluster tracking
  Step 5: Consistent Hashing       → Smart routing

Operations (Steps 6+):
  Step 6: PING                     → Simplest operation
  Step 7: GET                      → Read data
  Step 8: PUT                      → Write data
  Step 9+: Other operations        → REMOVE, CLEAR, STATS, etc.
```

### 3. Use Java Reference Implementation (MANDATORY)

**CRITICAL**: The Java Hot Rod client is your reference implementation.

**Every ROADMAP step includes "Java Reference" section** with exact class names and methods to study.

**Workflow for each step**:
```
1. Read ROADMAP step documentation
2. Find "Java Reference" section → class names listed
3. Clone Infinispan source: https://github.com/infinispan/infinispan
4. Navigate to Java class: infinispan/client/hotrod-client/src/main/java/...
5. Study the implementation (encoding, wire format, logic)
6. Read test vectors from hotrod-foundry
7. Port into your language based on Java logic
8. Validate bytes match test vectors
```

**Example (Step 1 - Primitives)**:
```
ROADMAP says:
  Java Reference:
  - org.infinispan.client.hotrod.impl.transport.netty.ByteBufUtil
    - writeVInt() / readVInt() (lines 70-112)

You should:
  1. Open: infinispan/client/hotrod-client/.../ByteBufUtil.java
  2. Read writeVInt() implementation (see exact algorithm)
  3. Read readVInt() implementation (see decoding)
  4. Replicate the SAME logic in your language
  5. Test against test vectors
```

**Why Java is authoritative**:
- ✅ Most complete, production-tested implementation
- ✅ Handles all edge cases
- ✅ Shows exact wire format expectations
- ✅ Infinispan server is also Java (same codebase)

**When in doubt**: Check what Java does. If your bytes don't match test vectors, compare with Java output.

**DO NOT**:
- ❌ Guess the wire format
- ❌ Port from docs alone without checking Java
- ❌ Skip reading the Java code

### 4. Test-Driven Porting (MANDATORY)

**For EVERY step**, follow this workflow:

```
1. Read ROADMAP step docs       → Understand what to port
2. Read Java reference code     → See how Java does it (MANDATORY)
3. Read test vectors            → test-vectors/step-XX-*/
4. Write failing unit tests     → Load test vectors, assert expected bytes
5. Port based on Java           → Replicate Java logic in your language
6. Validate bytes match         → 100% match with test vectors
7. Compare with Java if stuck   → Run Java client, compare bytes
8. Write integration tests      → Use Testcontainers library (MANDATORY)
9. Update PROGRESS.md           → Mark step complete
```

**Integration Testing Requirements**:
- ✅ Use **Testcontainers** library if available
- ✅ Ask the user if unsure

See `docs/development-guidelines.md` for complete Testcontainers examples in all languages.

## How to Use Each Documentation Type

### 📘 ROADMAP.md

**What**: Step-by-step porting plan  
**When**: Read first, reference constantly  
**How to use**:
1. Start at Step 0 (Foundation Setup)
2. Complete each step fully before moving to next
3. Use "Success Criteria" as checklist
4. Use "Java Reference" to find reference implementation

### ☕ Java Reference Implementation

**What**: The authoritative Hot Rod client implementation  
**When**: ALWAYS - for every step before writing code  
**Where**: https://github.com/infinispan/infinispan  
**Path**: `client/hotrod-client/src/main/java/org/infinispan/client/hotrod/`

**How to use Java code**:

**Step 1: Ask the user for the path**

**Step 2: Find the class (from ROADMAP "Java Reference" section)**

Example for Step 1 (Primitives):
```
ROADMAP says: org.infinispan.client.hotrod.impl.transport.netty.ByteBufUtil

Navigate to:
  infinispan/client/hotrod-client/src/main/java/
    org/infinispan/client/hotrod/impl/transport/netty/ByteBufUtil.java
```

**Step 3: Study the implementation**
```java
// Example: writeVInt() method shows exact encoding algorithm
public static void writeVInt(ByteBuf buf, int i) {
    while ((i & ~0x7F) != 0) {
        buf.writeByte((byte) ((i & 0x7F) | 0x80));
        i >>>= 7;
    }
    buf.writeByte((byte) i);
}
```

**Step 4: Replicate in your language**
```csharp
// C# equivalent based on Java logic
public void WriteVInt(List<byte> buffer, int value) {
    while ((value & ~0x7F) != 0) {
        buffer.Add((byte)((value & 0x7F) | 0x80));
        value >>>= 7;
    }
    buffer.Add((byte)value);
}
```

**Step 5: Validate against test vectors**

**Common Java classes to reference**:
- **Primitives**: `ByteBufUtil` (vInt, vLong, strings)
- **Headers**: `Codec40` (writeHeader, readHeader)
- **Operations**: `PingOperation`, `GetOperation`, `PutOperation`
- **Auth**: `AuthMechListOperation`, `AuthOperation`
- **Topology**: `TopologyInfo`, `Codec30.readNewTopology()`
- **Hashing**: `org.infinispan.client.hotrod.impl.consistenthash.*`

**Why this matters**:
- ✅ Java client is battle-tested in production
- ✅ Handles all edge cases
- ✅ Shows exact wire format (what server expects)
- ✅ Infinispan server is Java (same encoding/decoding)
- ✅ When bytes don't match, Java shows you why

**If your port doesn't work**:
1. Check Java code for that specific encoding
2. Fix your port to match Java logic

**DO NOT** skip reading Java code. It's the Java reference implementation.

### 🧪 test-vectors/*.json Files

**What**: Expected input/output bytes for validation  
**When**: Before writing ANY porting code  
**How to use**:

**Structure of a test vector**:
```json
{
  "description": "Human-readable description",
  "input": {"field1": "value1", "field2": 42},
  "expected_bytes": "A0 01 28 21 00 00",
  "byte_breakdown": [
    {"field": "magic", "bytes": "A0", "decimal": 160},
    {"field": "message_id", "bytes": "01", "vlong": 1}
  ]
}
```

**Workflow**:
```python
# 1. Load test vector
test = json.load("test-vectors/step-01-primitives/vint-test-cases.json")

# 2. Write failing test
def test_vint_encoding():
    for case in test["test_cases"]:
        result = encode_vint(case["input"])
        assert result == bytes.fromhex(case["expected_bytes"])
        # Initially fails because encode_vint() doesn't exist

# 3. Port encode_vint() to make test pass

# 4. Verify 100% of test vectors pass before moving on
```

**Key Directories**:
- `step-01-primitives/` - vInt, vLong encoding
- `step-03-authentication/` - Auth request/response examples
- `protocol-40-complete/` - Complete Protocol 4.0 headers with ALL fields

### Step-by-Step Workflow

2. **Initialize project** (package.json, csproj, setup.py, CMakeLists.txt, etc.)
3. **Copy progress template**:
   ```bash
   cp ../hotrod-foundry/templates/PROGRESS.md.template ./PROGRESS.md
   ```
4. **Create `.github/workflows/build.yml`** (see `development-guidelines.md`)
5. **Create basic README.md**
6. **Set up test framework**
7. **Commit initial structure**

**Deliverables**:
- [ ] New directory/repo created OUTSIDE of hotrod-foundry
- [ ] Confirmed you are NOT in hotrod-foundry/ directory
- [ ] CI/CD pipeline running (even if tests don't exist yet)
- [ ] PROGRESS.md tracking started
- [ ] Test framework configured

**Verification**:
```bash
pwd  # Should show: /path/to/[language]-client
     # Should NOT show: /path/to/hotrod-foundry
```

### Step 1: Wire Format Primitives

**Goal**: Port vInt, vLong, string, array encoding/decoding

**Actions**:
1. **Read**: `docs/01-wire-format-primitives.md`
2. **Load test vectors**: `test-vectors/step-01-primitives/*.json`
3. **Write tests**:
   ```csharp
   // Load test vectors
   var tests = LoadTestVectors("test-vectors/step-01-primitives/vint-test-cases.json");
   
   [Fact]
   public void TestVIntEncoding()
   {
       foreach (var test in tests.TestCases)
       {
           var buffer = new List<byte>();
           Codec.WriteVInt(buffer, test.Input);
           Assert.Equal(test.ExpectedBytes, buffer.ToArray());
       }
   }
   ```
4. **Port**:
   ```csharp
   public static void WriteVInt(List<byte> buffer, int value)
   {
       // See docs/01-wire-format-primitives.md for algorithm
       // Port based on spec / Java reference
   }
   ```
5. **Validate**: All test vectors pass
6. **Update**: PROGRESS.md

**Success Criteria**:
- [ ] vInt encoding matches test vectors
- [ ] vLong encoding matches test vectors
- [ ] String encoding matches test vectors
- [ ] All unit tests green
- [ ] PROGRESS.md updated

### Step 2: Protocol Headers

**Goal**: Encode/decode complete Protocol 4.0 request/response headers

**Actions**:
1. **Read**: `docs/02-protocol-headers.md` (pay attention to warnings!)
2. **Read Kaitai schema**: `reference/hotrod-dissector/schemas/hotrod40.ksy` lines 19-60
3. **Load test vectors**: `test-vectors/protocol-40-complete/ping-request.json`
4. **Critical**: Note ALL conditional fields:
   ```
   - key_media_type (if version >= 0x28)  
   - value_media_type (if version >= 0x28)
   - other_param_count (if version >= 40)   ← Missing this = timeout!
   ```
5. **Write test**:
   ```csharp
   [Fact]
   public void TestPingRequestHeader()
   {
       var test = LoadTestVector("protocol-40-complete/ping-request.json");
       var header = new RequestHeader
       {
           MessageId = test.Fields.MessageId,
           Opcode = Opcodes.PING_REQUEST,
           // ... other fields
       };
       var bytes = EncodeRequestHeader(header);
       Assert.Equal(test.ExpectedBytes, bytes);
   }
   ```
6. **Port**: Complete header encoder/decoder
7. **Validate byte-by-byte**: Compare with test vector's `byte_breakdown`
8. **Common mistake**: Forgetting `other_param_count` field for Protocol 4.0

**Success Criteria**:
- [ ] Can encode complete Protocol 4.0 request header
- [ ] Can decode Protocol 4.0 response header
- [ ] Bytes match test vectors exactly (use hex dump comparison)
- [ ] All conditional fields implemented
- [ ] PROGRESS.md updated

---

## Common Pitfalls (CRITICAL - READ THIS)

### ❌ Pitfall 1: Missing Conditional Fields

**Symptom**: Server timeout (no response)

**Cause**: Kaitai schema has fields with `if: version >= X`. You skipped them.

**Example**:
```yaml
# In hotrod40.ksy
- id: other_param_count
  type: vint
  if: version >= 40    # ← Easy to miss!
```

**Fix**: 
1. Read COMPLETE Kaitai schema
2. Port ALL fields, including those with `if:` conditions
3. Validate bytes match test vectors

**Prevention**: Use test vectors from `protocol-40-complete/` which include ALL fields.

### ❌ Pitfall 2: Testing Against Server Too Early

**Symptom**: Wasted hours debugging, trying different server versions, checking network, etc.

**Cause**: Skipped byte-level validation, went straight to live server testing.

**Fix**:
1. ALWAYS validate against test vectors first
2. Bytes must match exactly before testing server
3. Use hex dump comparison to find discrepancies

**Correct workflow**:
```
✓ 1. Load test vector
✓ 2. Write unit test
✓ 3. Port feature
✓ 4. Validate bytes match test vector 100%
✓ 5. THEN test against live server
```

### ❌ Pitfall 3: Searching for Existing Implementations

**Symptom**: Spending time searching GitHub, npm, NuGet, PyPI for existing clients

**Cause**: Thinking "maybe someone already did this in [language]"

**Why this is wrong**:
- ❌ Existing implementations may be incomplete (missing Steps 3-5)
- ❌ May be for old protocol versions (3.x instead of 4.0)
- ❌ May have bugs or wrong wire formats
- ❌ You won't learn the protocol if you just copy code
- ❌ User asked YOU to port, not to find someone else's work

**Fix**: 
1. **STOP searching** for existing code
2. Your references are: (1) Java, (2) hotrod-foundry, (3) test vectors
3. Port from scratch following ROADMAP
4. Study Java implementation, not random GitHub code

**Correct mindset**: "I'm porting FROM SCRATCH using Java as reference"

---

## Validation Checklist

Before claiming a step is "complete", verify:

**Unit Tests**:
- [ ] All test vectors pass
- [ ] Error cases handled
- [ ] Edge cases tested

**Integration Tests**:
- [ ] Works against live Infinispan server (Docker)
- [ ] Handles server errors gracefully
- [ ] Connection management works

**Documentation**:
- [ ] PROGRESS.md updated

---

## Final Checklist

Before telling the user "porting complete":

**Functionality**:
- [ ] All roadmap steps 1-5 complete
- [ ] At least PING, GET, PUT operations work
- [ ] Authentication works (SCRAM-SHA-256)
- [ ] Topology awareness works
- [ ] Consistent hashing works

**Testing**:
- [ ] All test vectors pass
- [ ] Integration tests pass
- [ ] Works with 3-node Infinispan cluster

---

## Questions to Ask the User

At the **start** of porting:

1. "Which language should I use?" (C#, Python, Go, Rust, etc.)
2. "What should I name the package/library?" (e.g., `hotrod-client-python`)
3. "Should I create a new repository or work in an existing one?"
4. "Do you have a preferred test framework?" (xUnit, pytest, etc.)
5. "Do you want CI/CD setup?" (GitHub Actions, GitLab CI, etc.)

At **Step 5 completion**:

1. "Foundation complete (auth, topology, hashing). Should I release v0.1.0 with PING only, or continue to GET/PUT first?"

At **feature-complete**:

1. "All core operations ported. Ready to publish to [package registry]?"

---

## Summary

**Your role as an AI agent**:
1. Follow ROADMAP.md step-by-step (don't skip)
2. Use test vectors for validation (not live server initially)
3. Read Kaitai schema when docs are unclear
4. Track progress in PROGRESS.md
5. Set up CI/CD early
6. Release after Step 5 (foundation complete)

**Most important**: Validate bytes against test vectors BEFORE testing against live server. This saves hours of debugging.

Good luck! 🚀
