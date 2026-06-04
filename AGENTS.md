# Hot Rod Client Implementation Guide for AI Agents

**Audience**: AI assistants (Claude, GPT-4, etc.) tasked with implementing an Infinispan Hot Rod client in any programming language.

**Purpose**: This document explains how to use the `hotrod-foundry` documentation repository to generate a complete, production-ready Hot Rod client from scratch.

---

## Quick Start

When a user asks you to "implement a Hot Rod client in [language]", follow this sequence:

1. **Read this file completely** (you're doing it now ✓)
2. **Read `ROADMAP.md`** to understand the implementation steps
3. **Read `docs/development-guidelines.md`** for process requirements
4. **Follow the roadmap step-by-step** (don't skip ahead)
5. **Use test vectors for validation** (not live server initially)
6. **Track progress in PROGRESS.md** (update after each step)

---

## Core Principles

### 1. Infrastructure Before Operations

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

**Why this order?**
- After Step 5, the client is production-ready (can release v0.1)
- Every operation benefits from auth, topology, and routing
- User can deploy with just PING/GET/PUT and still have proper cluster support

### 2. Test-Driven Development (MANDATORY)

**For EVERY step**, follow this workflow:

```
1. Read test vectors         → test-vectors/step-XX-*/
2. Write failing tests       → Load test vectors, assert expected bytes
3. Implement minimal code    → Make tests pass
4. Validate bytes match      → 100% match with test vectors
5. Write integration tests   → Test against live server (optional)
6. Update PROGRESS.md        → Mark step complete
```

**NEVER skip to live server testing without byte-level validation first.**

### 3. Separate Repository

**DO NOT** put implementation code in `hotrod-foundry/`.

This repository = language-agnostic docs + test vectors only.

Your implementation = separate repository:
```
hotrod-client-[language]/       ← NEW REPO
├── src/                        ← Your code
├── tests/                      ← Your tests
├── PROGRESS.md                 ← Your progress tracker
└── .github/workflows/          ← Your CI/CD
```

---

## How to Use Each Documentation Type

### 📘 ROADMAP.md

**What**: Step-by-step implementation plan  
**When**: Read first, reference constantly  
**How to use**:
1. Start at Step 0 (Foundation Setup)
2. Complete each step fully before moving to next
3. Use "Success Criteria" as checklist
4. Use "Java Reference" to find reference implementation

**Example**:
```markdown
User: "Implement a C# Hot Rod client"
You: 
  1. Read ROADMAP.md
  2. Create new repo: hotrod-client-csharp
  3. Start Step 0: Set up project structure, CI/CD
  4. Move to Step 1: Implement vInt, vLong, strings
  5. ... (continue step by step)
```

### 📗 docs/*.md Files

**What**: Detailed specifications for each concept  
**When**: Read when starting each step  
**How to use**:

| File | Purpose | When to Read |
|------|---------|--------------|
| `00-introduction.md` | Protocol overview | Before Step 1 |
| `01-wire-format-primitives.md` | vInt, vLong, strings | Step 1 |
| `02-protocol-headers.md` | Request/response headers | Step 2 |
| `03-authentication.md` | SASL mechanisms | Step 3 |
| `04-topology-awareness.md` | Cluster tracking | Step 4 |
| `05-consistent-hashing.md` | Smart routing | Step 5 |
| `06-ping-operation.md` | PING operation | Step 6 |
| `development-guidelines.md` | **CRITICAL** - TDD, CI/CD, process | Before Step 0 |
| `troubleshooting.md` | Debugging guide | When stuck |

**Reading Strategy**:
1. Skim the doc for overall structure
2. Focus on "Wire Format" sections (exact byte layout)
3. Note all conditional fields (if version >= X)
4. Check "Common Mistakes" sections
5. Follow links to Kaitai schema for authoritative spec

### 🧪 test-vectors/*.json Files

**What**: Expected input/output bytes for validation  
**When**: Before writing ANY implementation code  
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

# 3. Implement encode_vint() to make test pass

# 4. Verify 100% of test vectors pass before moving on
```

**Key Directories**:
- `step-01-primitives/` - vInt, vLong encoding
- `step-03-authentication/` - Auth request/response examples
- `protocol-40-complete/` - Complete Protocol 4.0 headers with ALL fields

### 🔬 reference/hotrod-dissector/schemas/*.ksy

**What**: Kaitai Struct schema - **SOURCE OF TRUTH**  
**When**: When doc is unclear or you suspect missing fields  
**How to use**:

**The Kaitai schema is authoritative.** If docs say one thing and Kaitai says another, trust Kaitai.

**Reading Kaitai schema**:
```yaml
seq:
  - id: magic
    contents: [0xa0]           # Fixed byte: 0xA0
  
  - id: message_id
    type: vlong                # Variable-length long
  
  - id: key_media_type
    type: media_type
    if: version >= 0x28        # ← CONDITIONAL FIELD! Required for Protocol 4.0+
  
  - id: other_param_count
    type: vint
    if: version >= 40          # ← CONDITIONAL! Easy to miss!
```

**Critical**: Always implement fields with `if:` conditions. Missing conditionals cause silent server timeouts.

**When to read**:
1. Starting Step 2 (headers) - understand complete structure
2. Debugging timeouts - check for missing conditional fields
3. Validating wire format - ensure byte order matches `seq:`

### 📋 templates/PROGRESS.md.template

**What**: Template for tracking implementation progress  
**When**: Copy to your repo at Step 0  
**How to use**:

```bash
# 1. Copy template to your repo
cp templates/PROGRESS.md.template ../hotrod-client-csharp/PROGRESS.md

# 2. Update after EVERY step
# 3. Include in every commit
# 4. Reference in README.md
```

**Update frequency**: After completing each step, or at least weekly.

---

## Step-by-Step Workflow

### Step 0: Foundation Setup

**Goal**: Create project structure, CI/CD, and tracking

**Actions**:
1. Create new repository: `hotrod-client-[language]`
2. Initialize project (package.json, csproj, setup.py, etc.)
3. Copy `templates/PROGRESS.md.template` to `PROGRESS.md`
4. Create `.github/workflows/build.yml` (see `development-guidelines.md`)
5. Create basic README.md
6. Set up test framework
7. Commit initial structure

**Deliverables**:
- [ ] New repo created and initialized
- [ ] CI/CD pipeline running (even if tests don't exist yet)
- [ ] PROGRESS.md tracking started
- [ ] Test framework configured

### Step 1: Wire Format Primitives

**Goal**: Implement vInt, vLong, string, array encoding/decoding

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
4. **Implement**:
   ```csharp
   public static void WriteVInt(List<byte> buffer, int value)
   {
       // See docs/01-wire-format-primitives.md for algorithm
       // Implement based on spec
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
6. **Implement**: Complete header encoder/decoder
7. **Validate byte-by-byte**: Compare with test vector's `byte_breakdown`
8. **Common mistake**: Forgetting `other_param_count` field for Protocol 4.0

**Success Criteria**:
- [ ] Can encode complete Protocol 4.0 request header
- [ ] Can decode Protocol 4.0 response header
- [ ] Bytes match test vectors exactly (use hex dump comparison)
- [ ] All conditional fields implemented
- [ ] PROGRESS.md updated

**Debugging**:
If bytes don't match:
1. Print hex dump of your output vs expected
2. Check `byte_breakdown` field in test vector
3. Find first byte that differs
4. Check Kaitai schema for that field's encoding rules

### Step 3: Authentication

**Goal**: Implement SASL SCRAM-SHA-256 authentication

**Actions**:
1. **Read**: `docs/03-authentication.md`
2. **Read RFCs**: `reference/rfc5802-scram.txt` (at least section 3)
3. **Load test vectors**: `test-vectors/step-03-authentication/*.json`
4. **Implement operations**:
   - AUTH_MECH_LIST (opcode 0x21/0x22) - query supported mechanisms
   - AUTH (opcode 0x23/0x24) - perform authentication
5. **Implement SCRAM-SHA-256**:
   - PBKDF2-SHA256 key derivation
   - HMAC-SHA256 for signatures
   - Base64 encoding/decoding
   - Nonce generation (cryptographically secure random)
   - Challenge/response flow
6. **Test against vectors first**, then against server
7. **Integration test**:
   ```bash
   docker run -d -p 11222:11222 \
     -e USER=admin -e PASS=password \
     infinispan/server:16.0
   
   # Test: authenticate as admin/password
   ```

**Success Criteria**:
- [ ] AUTH_MECH_LIST returns server mechanisms
- [ ] SCRAM-SHA-256 authentication succeeds
- [ ] Connection remains authenticated
- [ ] Subsequent operations work on authenticated connection
- [ ] Test vectors pass
- [ ] Integration test with Infinispan 16.0 passes
- [ ] PROGRESS.md updated

**Dependencies**: OpenSSL, BouncyCastle, or equivalent crypto library

### Step 4: Topology Awareness

**Goal**: Track cluster topology for failover

**Actions**:
1. **Read**: `docs/04-topology-awareness.md`
2. **Load test vectors**: `test-vectors/step-04-topology/*.json`
3. **Implement**:
   - Parse topology change marker in response headers
   - Track server list (host:port pairs)
   - Topology version tracking
   - Update client intelligence to 0x02 (TOPOLOGY_AWARE)
4. **Test with multi-node cluster**:
   ```bash
   # Start 3-node cluster
   docker run -d --name node1 -p 11222:11222 infinispan/server:16.0
   docker run -d --name node2 -p 11322:11222 infinispan/server:16.0
   docker run -d --name node3 -p 11422:11222 infinispan/server:16.0
   ```

**Success Criteria**:
- [ ] Parses topology updates from responses
- [ ] Tracks all cluster nodes
- [ ] Can failover to different node
- [ ] Client intelligence = 0x02 in requests
- [ ] PROGRESS.md updated

### Step 5: Consistent Hashing

**Goal**: Smart routing - send requests to key owner

**Actions**:
1. **Read**: `docs/05-consistent-hashing.md`
2. **Load test vectors**: `test-vectors/step-05-hashing/*.json`
3. **Implement**:
   - MurmurHash3 algorithm
   - Consistent hash ring
   - Segment ownership mapping
   - Key-to-server routing
   - Update client intelligence to 0x03 (HASH_DISTRIBUTION_AWARE)
4. **Validate MurmurHash3**: Must match Java implementation exactly
   ```csharp
   [Fact]
   public void TestMurmurHash3()
   {
       var tests = LoadTestVectors("step-05-hashing/murmurhash3-test-cases.json");
       foreach (var test in tests.TestCases)
       {
           var hash = MurmurHash3.Hash(test.Input);
           Assert.Equal(test.ExpectedHash, hash);
       }
   }
   ```

**Success Criteria**:
- [ ] MurmurHash3 matches test vectors
- [ ] Routes keys to correct server
- [ ] Client intelligence = 0x03
- [ ] Works with 3+ node cluster
- [ ] PROGRESS.md updated

**Milestone**: After Step 5, foundation is complete. You can release **v0.1.0**!

### Step 6+: Operations

**Goal**: Implement PING, GET, PUT, etc.

**Actions**:
1. **Read**: `docs/06-ping-operation.md`, `docs/07-get-operation.md`, etc.
2. **Load test vectors**: `test-vectors/step-06-ping/*.json`
3. **Implement operation**:
   - Request encoding
   - Response decoding
   - Error handling
4. **Test**: Unit tests with vectors, then integration tests

**Success Criteria** (per operation):
- [ ] Request encoding matches test vectors
- [ ] Response decoding works
- [ ] Error handling works (status codes)
- [ ] Integration test passes
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
2. Implement ALL fields, including those with `if:` conditions
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
✓ 3. Implement feature
✓ 4. Validate bytes match test vector 100%
✓ 5. THEN test against live server
```

### ❌ Pitfall 3: Skipping Steps

**Symptom**: GET/PUT work but client can't handle cluster failover, poor performance

**Cause**: Skipped Steps 4-5 (topology, hashing), went straight to operations

**Fix**: Follow roadmap order. Steps 1-5 are NOT optional.

**Why order matters**:
- Step 3 (Auth): Required for Infinispan 15+
- Step 4 (Topology): Required for failover in production
- Step 5 (Hashing): Required for performance at scale

### ❌ Pitfall 4: Not Tracking Progress

**Symptom**: Incomplete implementation, hard to resume work, no clear status

**Cause**: Didn't maintain PROGRESS.md

**Fix**: Update PROGRESS.md after every step. Include in commits.

### ❌ Pitfall 5: No CI/CD

**Symptom**: Regressions, broken builds, manual testing burden

**Cause**: Skipped CI/CD setup (Step 0)

**Fix**: Set up GitHub Actions (or equivalent) at Step 0. See `development-guidelines.md`.

### ❌ Pitfall 6: Wrong Protocol Version

**Symptom**: Authentication failures, unexpected errors

**Cause**: Using Protocol 3.x when server expects 4.0+

**Fix**: 
- Infinispan 14+ requires Protocol 4.0 (version byte = 0x28 = 40)
- Always use Protocol 4.0+ for new implementations
- Check server version compatibility table in docs

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

**Code Quality**:
- [ ] No compiler warnings
- [ ] Linter passes
- [ ] Code documented (public API)
- [ ] No hardcoded credentials

**Documentation**:
- [ ] PROGRESS.md updated
- [ ] README.md reflects current features
- [ ] Examples provided

**CI/CD**:
- [ ] Pipeline green
- [ ] Tests run automatically
- [ ] Coverage report generated

---

## Debugging Workflow

When something doesn't work:

### 1. Timeout (No Response from Server)

**Most likely**: Missing required field in request header

**Debug**:
```bash
# 1. Print hex dump of your request
your_bytes: A0 01 28 21 00 00 01 00 00 00

# 2. Compare with test vector
expected:   A0 01 28 21 00 00 01 00 00 00 00
                                          ^^-- Missing!

# 3. Check byte_breakdown in test vector
# 4. Find missing field in Kaitai schema
# 5. Implement missing field
```

**Prevention**: Always validate bytes against test vectors before testing server.

### 2. Server Error (Status != 0x00)

**Check**: `docs/troubleshooting.md` for status code meanings

**Common status codes**:
- `0x85` (SERVER_ERROR) - Wrong media type encoding
- `0x86` (COMMAND_TIMEOUT) - Operation took too long
- `0x81` (INVALID_MAGIC) - Magic byte should be 0xA0

### 3. Authentication Failure

**Check**:
1. Server version (15+ requires auth)
2. Credentials correct (default: admin/password)
3. Mechanism supported (query with AUTH_MECH_LIST first)
4. SCRAM-SHA-256 implementation (test against vectors)

### 4. Bytes Don't Match Test Vector

**Debug**:
```python
# 1. Generate byte-by-byte diff
your_bytes = [0xA0, 0x01, 0x28, 0x21, 0x00, 0x00, 0x01, 0x00]
expected   = [0xA0, 0x01, 0x28, 0x21, 0x00, 0x00, 0x01, 0x00, 0x00, 0x00]

# 2. Find first difference
for i, (a, b) in enumerate(zip(your_bytes, expected)):
    if a != b:
        print(f"Byte {i}: got {a:02X}, expected {b:02X}")
        break

# 3. Check test vector's byte_breakdown for that position
# 4. Check Kaitai schema for encoding rules
# 5. Fix implementation
```

---

## Release Strategy

**Version Milestones**:

| Version | Steps Complete | Features | Release Readiness |
|---------|---------------|----------|-------------------|
| v0.1.0  | 0-5 + PING    | Foundation + PING | ✅ Production-ready for health checks |
| v0.2.0  | + GET         | Read operations | ✅ Production-ready for read-only caches |
| v0.3.0  | + PUT         | Write operations | ✅ Production-ready for read/write |
| v0.4.0  | + REMOVE      | Delete operations | ✅ Full CRUD support |
| v1.0.0  | All core ops  | Complete client | ✅ Feature-complete |

**Key insight**: You can release after Step 5 + PING. The foundation (auth, topology, hashing) makes it production-ready even with limited operations.

---

## Final Checklist

Before telling the user "implementation complete":

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
- [ ] CI/CD pipeline green

**Documentation**:
- [ ] README.md with examples
- [ ] PROGRESS.md shows all steps complete
- [ ] Public API documented
- [ ] Installation instructions provided

**Code Quality**:
- [ ] No compiler warnings
- [ ] Linter passes
- [ ] Test coverage ≥ 80%
- [ ] No TODOs or FIXMEs in main branch

**Release**:
- [ ] Versioned (e.g., v0.1.0)
- [ ] Tagged in git
- [ ] Published to package registry (NuGet, npm, PyPI, etc.)
- [ ] CHANGELOG.md created

---

## Questions to Ask the User

At the **start** of implementation:

1. "Which language should I use?" (C#, Python, Go, Rust, etc.)
2. "What should I name the package/library?" (e.g., `hotrod-client-python`)
3. "Should I create a new repository or work in an existing one?"
4. "Do you have a preferred test framework?" (xUnit, pytest, etc.)
5. "Do you want CI/CD setup?" (GitHub Actions, GitLab CI, etc.)

At **Step 5 completion**:

1. "Foundation complete (auth, topology, hashing). Should I release v0.1.0 with PING only, or continue to GET/PUT first?"

At **feature-complete**:

1. "All core operations implemented. Ready to publish to [package registry]?"

---

## Summary

**Your role as an AI agent**:
1. Follow ROADMAP.md step-by-step (don't skip)
2. Use test vectors for validation (not live server initially)
3. Read Kaitai schema when docs are unclear
4. Track progress in PROGRESS.md
5. Set up CI/CD early
6. Release after Step 5 (foundation complete)

**Success = Production-ready client with**:
- Authentication (SCRAM-SHA-256)
- Topology awareness (cluster failover)
- Consistent hashing (smart routing)
- At least GET/PUT operations
- 80%+ test coverage
- CI/CD pipeline
- Published to package registry

**Most important**: Validate bytes against test vectors BEFORE testing against live server. This saves hours of debugging.

---

## Additional Resources

- **Java Reference**: `org.infinispan.client.hotrod.*` (GitHub: infinispan/infinispan)
- **Protocol Spec**: Kaitai schema is authoritative (`reference/hotrod-dissector/schemas/hotrod40.ksy`)
- **Test Server**: `docker run -p 11222:11222 -e USER=admin -e PASS=password infinispan/server:16.0`
- **Troubleshooting**: See `docs/troubleshooting.md`

Good luck! 🚀
