# Hot Rod Client Implementation Roadmap

**Version**: 2.0 (Infrastructure-First)  
**Target Protocol**: Hot Rod 4.0 (extensible to 4.1)  
**Last Updated**: 2026-06-05

## Overview

This roadmap defines **incremental, testable steps** for implementing a Hot Rod protocol client in any programming language. Each step:

- Is **small and focused** (1-3 hours of work)
- Produces **working, testable code**
- Has **language-agnostic documentation**
- Mirrors **Java client tests** for validation
- Can be **replicated in any language**

## Implementation Philosophy: Infrastructure First

**Critical insight**: Build the foundation (Steps 0-5) before operations (Steps 6+).

**Why this order?**
- **Infinispan 15+ requires authentication** → Can't test without Step 3
- **Production deployments need clustering** → Topology (Step 4) and hashing (Step 5) are essential
- **Release early**: After Step 5, you have a production-ready client (can release v0.1)
- **Operations benefit**: Every GET/PUT/REMOVE inherits auth, topology, and smart routing

**Old approach** (operations-first):
```
Steps 1-10: Operations → Step 11-12: Clustering → Step 13+: Auth
Problem: Can't release until Step 12, even for simple use cases
```

**New approach** (infrastructure-first):
```
Steps 1-5: Foundation → Steps 6+: Operations
Benefit: Release after Step 5 with PING only, add operations incrementally
```

---

## Development Principles

**IMPORTANT**: Read `docs/development-guidelines.md` for complete process requirements.

### Key Rules

1. **Cross-Platform**: Linux (Fedora/RHEL) + Windows (MANDATORY)
2. **Standalone Project**: Each language = separate repository with own CI/CD
3. **Test-Driven**: Test vectors → Unit tests → Implementation → Integration tests
4. **Track Progress**: Update PROGRESS.md after each step
5. **Validate Bytes**: Compare with test vectors BEFORE testing against server
6. **CI Required**: Automated builds and tests on every commit, both platforms
7. **One Step at a Time**: Complete step N fully before starting step N+1

### Per-Step Checklist

For a step to be "complete":
- [ ] All test vectors implemented and passing
- [ ] Unit tests written and green (on both Linux and Windows)
- [ ] Integration tests written (auto-skip if no server)
- [ ] CI pipeline green (both platforms)
- [ ] PROGRESS.md updated
- [ ] Code documented
- [ ] PR reviewed (if applicable)

**Do NOT** skip ahead. Each step builds on previous ones.

---

## Reference Sources

| Source | Purpose |
|--------|---------|
| `hotrod-foundry/docs/` | Protocol specifications |
| `hotrod-foundry/test-vectors/` | Validation data |
| `org.infinispan.client.hotrod.*` | Java reference implementation |
| `hotrod-dissector/schemas/*.ksy` | Formal Kaitai specification |
| Infinispan Protocol Docs | Official documentation |

---

## Step 0: Foundation Setup ⚙️

**Goal**: Project structure, build system, test infrastructure

**⚠️ CRITICAL**: Create this in a SEPARATE directory from hotrod-foundry

### Deliverables
- [ ] New directory/repository: `hotrod-client-[language]` (NOT in hotrod-foundry/)
- [ ] Project structure (src/, include/, tests/, docs/)
- [ ] Build system (CMake, Gradle, npm, etc.) - works on Linux + Windows
- [ ] Test framework integration (Google Test, xUnit, pytest, etc.)
- [ ] CI/CD pipeline (GitHub Actions with Linux + Windows jobs)
- [ ] PROGRESS.md (copy from `hotrod-foundry/templates/PROGRESS.md.template`)
- [ ] Can connect to test server via TCP on both platforms

### Platform Requirements
- ✅ Build works on Linux (Fedora/RHEL/Ubuntu)
- ✅ Build works on Windows 10/11
- ✅ CI tests both platforms

### Documentation
- README.md with setup instructions for both platforms
- Reference to `hotrod-foundry/docs/development-guidelines.md`

### Tests
- Build system compiles successfully (Linux + Windows)
- Test framework runs
- Can establish TCP connection to Infinispan server (port 11222)

### Java Reference
- Maven build structure
- Test infrastructure in `src/test/java/`

### Success Criteria
✅ CI pipeline runs and passes on both Linux and Windows  
✅ Can connect to Infinispan test server  
✅ Test framework executes  
✅ PROGRESS.md tracking started
✅ Confirmed working in separate directory (not hotrod-foundry/)

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
- [ ] Test vectors validated against expected output

### Documentation
Reference: `hotrod-foundry/docs/01-wire-format-primitives.md`
- vInt/vLong encoding algorithm
- ZigZag encoding explanation
- String encoding (vInt length + UTF-8 bytes)
- Byte array encoding (vInt length + raw bytes)
- Wire format examples with hex dumps

### Test Vectors
Location: `hotrod-foundry/test-vectors/step-01-primitives/*.json`

**vInt encoding examples**:
- `0` → `0x00`
- `127` → `0x7F`
- `128` → `0x80 0x01`
- `16384` → `0x80 0x80 0x01`

**Strings**:
- `"hello"` → `0x05 68 65 6C 6C 6F`
- `"café"` (UTF-8) → `0x05 63 61 66 C3 A9`

### Java Reference
- `org.infinispan.client.hotrod.impl.transport.netty.ByteBufUtil`
- `org.infinispan.commons.io.SignedNumeric` (ZigZag)

### Success Criteria
✅ All test vectors pass (100% match)  
✅ Encoded bytes match expected output byte-for-byte  
✅ Round-trip encode/decode works  
✅ Edge cases handled (0, max values, empty strings/arrays)
✅ Works on both Linux and Windows

---

## Step 2: Protocol Headers 📋

**Goal**: Build and parse Hot Rod 4.0 request/response headers (COMPLETE spec)

**⚠️ CRITICAL**: Protocol 4.0 has conditional fields. Missing ANY field causes server timeouts.

### Deliverables
- [ ] Request header encoding (ALL fields including conditionals)
- [ ] Response header decoding
- [ ] Magic byte validation (0xA0 request, 0xA1 response)
- [ ] Unit tests for complete headers
- [ ] Test vectors validated

### Documentation
Reference: `hotrod-foundry/docs/02-protocol-headers.md`
- Complete Protocol 4.0 header specification
- Conditional fields (media types for version >= 0x28, other_param_count for version >= 40)
- Protocol version matrix
- Kaitai schema cross-references

### Request Header Structure (Protocol 4.0 - version 0x28 / 40)

**ALL fields in exact order**:
1. Magic Byte: `0xA0` (1 byte)
2. Message ID: vLong (unique per request)
3. Version: `0x28` (1 byte, for Protocol 4.0)
4. Opcode: 1 byte
5. Cache Name: string (vInt length + UTF-8)
6. Flags: vInt
7. Client Intelligence: 1 byte (0x01=BASIC, 0x02=TOPOLOGY_AWARE, 0x03=HASH_AWARE)
8. Topology ID: vInt
9. **Key Media Type**: media_type (if version >= 0x28) ← **Required for 4.0**
10. **Value Media Type**: media_type (if version >= 0x28) ← **Required for 4.0**
11. **Other Param Count**: vInt (if version >= 40) ← **Required for 4.0, usually 0**
12. Other Params: key-value pairs (if count > 0)

**⚠️ Common mistake**: Forgetting field #11 (other_param_count) causes server to wait indefinitely = timeout.

For operations without keys/values (PING, AUTH_MECH_LIST):
- Key Media Type: `0x00` (none)
- Value Media Type: `0x00` (none)
- Other Param Count: `0x00` (usually 0)

### Response Header Structure
1. Magic Byte: `0xA1`
2. Message ID: vLong (matches request)
3. Opcode: 1 byte (matches request)
4. Status: 1 byte (0x00=success, 0x01=not found, 0x85=server error, etc.)
5. Topology Change Flag: 1 byte (0x00=no change, 0x01=changed)

### Test Vectors
Location: `hotrod-foundry/test-vectors/protocol-40-complete/*.json`
- Complete PING request header
- Complete AUTH_MECH_LIST request header
- Response header examples

### Server Version Compatibility

| Infinispan Version | Protocol Version | Auth Required? |
|-------------------|------------------|----------------|
| 14.x              | 4.0 (0x28)       | No (optional)  |
| 15.x - 16.x       | 4.0/4.1 (0x28+)  | Yes (required) |

**Important**: For Infinispan 15+, you MUST implement authentication (Step 3) before testing operations.

### Java Reference
- `org.infinispan.client.hotrod.impl.protocol.Codec40`
  - `writeHeader()` method
  - `readHeader()` method

### Success Criteria
✅ Headers encode/decode correctly  
✅ Bytes match test vectors exactly (hex dump comparison)
✅ ALL conditional fields implemented (media types, other_param_count)  
✅ Can distinguish request vs response by magic byte
✅ Works on both platforms

**Validation workflow**:
1. Implement header encoder
2. Test against test vectors (bytes must match 100%)
3. ONLY THEN test against live server

---

## Step 3: Authentication (SASL) 🔐

**Goal**: Implement SASL authentication required for Infinispan 15+

**Why This Early?**
Modern Infinispan servers (15.0+) require authentication by default. You cannot test PING or any other operation without it. While older versions (14.x) support unauthenticated access, implementing authentication now enables testing against current server versions.

### Deliverables
- [ ] AUTH_MECH_LIST operation (opcode 0x21 request / 0x22 response)
- [ ] AUTH operation (opcode 0x23 request / 0x24 response)
- [ ] SCRAM-SHA-256 authentication mechanism (RFC 5802)
  - [ ] PBKDF2-SHA256 key derivation
  - [ ] HMAC-SHA-256 for signatures
  - [ ] Base64 encoding/decoding
  - [ ] Nonce generation (cryptographically secure random)
  - [ ] Client-first-message, server-first-message, client-final-message flow
  - [ ] Server signature verification
- [ ] Optional: DIGEST-MD5 as fallback
- [ ] Optional: PLAIN for development (rarely supported by servers)
- [ ] Persistent authenticated connection

### Documentation
Reference: `hotrod-foundry/docs/03-authentication.md` (to be created)

Should include:
- SASL mechanism overview
- AUTH_MECH_LIST wire format
- AUTH wire format
- SCRAM-SHA-256 implementation guide (RFC 5802)
- Authentication flow diagram
- Challenge/response examples

### Test Vectors
Location: `hotrod-foundry/test-vectors/step-03-authentication/` (to be created)
- auth-mech-list-request.json
- auth-mech-list-response.json
- auth-scram-challenge.json
- auth-scram-response.json

### Implementation Notes

**SCRAM-SHA-256 flow**:
1. Client → Server: AUTH_MECH_LIST (query supported mechanisms)
2. Server → Client: List of mechanisms (expect "SCRAM-SHA-256")
3. Client → Server: AUTH with mechanism="SCRAM-SHA-256", initial response
4. Server → Client: Challenge (salt, iteration count)
5. Client → Server: Proof (HMAC-based)
6. Server → Client: Server signature
7. Connection now authenticated for subsequent operations

**Cross-platform dependencies**:
- OpenSSL (available on both Linux and Windows)
- Or language-specific crypto (System.Security.Cryptography in C#, hashlib in Python)

### Java Reference
- `org.infinispan.client.hotrod.impl.operations.AuthMechListOperation`
- `org.infinispan.client.hotrod.impl.operations.AuthOperation`
- `org.infinispan.client.hotrod.impl.transport.netty.AuthHandler`
- `javax.security.sasl` package

### Testing

**Test server setup**:
```bash
docker run -d -p 11222:11222 \
  -e USER=admin -e PASS=password \
  infinispan/server:16.0
```

**Test cases**:
- Query mechanisms with AUTH_MECH_LIST
- Authenticate with SCRAM-SHA-256 (admin/password)
- Verify connection remains authenticated
- Test PING after authentication
- Test multiple operations on same authenticated connection

### Success Criteria
✅ AUTH_MECH_LIST returns server mechanisms  
✅ SCRAM-SHA-256 authentication succeeds with admin/password  
✅ Connection remains authenticated for subsequent operations  
✅ Works with Infinispan 15.0+ and 16.0+
✅ Works on both Linux and Windows  
✅ Test vectors pass

---

## Step 4: Topology Awareness 🗺️

**Goal**: Track cluster topology for failover and load balancing

**Prerequisite**: Authentication (Step 3) implemented

**Why Before Operations?**
Topology awareness is infrastructure, not an operation. A client that can only PING but knows the cluster topology is more production-ready than one with GET/PUT but no topology support. This enables:
- Automatic failover when nodes go down
- Load balancing across cluster nodes
- Awareness of cluster changes (nodes added/removed)

### Deliverables
- [ ] Parse topology change responses (topology change marker = 0x01)
- [ ] Track server list (host:port pairs + hash IDs)
- [ ] Topology version tracking (increment on changes)
- [ ] Update client intelligence to 0x02 (TOPOLOGY_AWARE) in requests
- [ ] Connection failover to other servers
- [ ] Server health tracking

### Response Header Enhancement

When `topology_change_marker = 0x01`, response includes:
- Topology ID (vInt)
- Number of servers (vInt)
- For each server:
  - Hostname (length-prefixed string)
  - Port (uint16, 2 bytes big-endian)
  - Hash ID (int32, 4 bytes) - used in Step 5 for consistent hashing

### Documentation
Reference: `hotrod-foundry/docs/04-topology-awareness.md` (to be created)

Should include:
- Topology update wire format
- Server list structure
- Failover strategy
- Client intelligence levels (0x01, 0x02, 0x03)

### Test Vectors
Location: `hotrod-foundry/test-vectors/step-04-topology/` (to be created)
- topology-update-response.json
- multi-server-topology.json

### Implementation

**Client Intelligence levels**:
- `0x01` (BASIC): No topology awareness, connects to initial server only
- `0x02` (TOPOLOGY_AWARE): Knows cluster topology, can failover ← **This step**
- `0x03` (HASH_DISTRIBUTION_AWARE): Smart routing with consistent hashing ← **Step 5**

**Topology tracking**:
```cpp
struct Server {
    std::string host;
    uint16_t port;
    int32_t hashId;  // For Step 5
};

class TopologyInfo {
    int topologyId;
    std::vector<Server> servers;
    
    void updateFrom(ResponseHeader& resp);
    Server& selectServer();  // Round-robin or random
};
```

### Testing

**Test with multi-node cluster**:
```bash
# Start 3-node cluster
docker run -d --name node1 -p 11222:11222 \
  -e USER=admin -e PASS=password \
  infinispan/server:16.0

docker run -d --name node2 -p 11322:11222 \
  -e USER=admin -e PASS=password \
  infinispan/server:16.0

docker run -d --name node3 -p 11422:11222 \
  -e USER=admin -e PASS=password \
  infinispan/server:16.0
```

**Test cases**:
- Connect to node1, receive topology update with all 3 nodes
- Stop node1, client fails over to node2 or node3
- Add node4, client receives topology update
- Verify topology ID increments on changes

### Java Reference
- `org.infinispan.client.hotrod.impl.topology.TopologyInfo`
- `org.infinispan.client.hotrod.impl.protocol.Codec30.readNewTopology()`

### Success Criteria
✅ Parses topology updates from server responses  
✅ Maintains list of all cluster nodes  
✅ Can failover to different node if primary fails  
✅ Topology version increments correctly  
✅ Client intelligence = 0x02 in request headers
✅ Works on both platforms  
✅ Integration tests pass with 3-node cluster

---

## Step 5: Consistent Hashing 🎯

**Goal**: Smart routing - send requests directly to key owner

**Prerequisite**: Topology Awareness (Step 4) implemented

**Why Before Operations?**
With consistent hashing, every GET/PUT goes directly to the right server (primary owner). Without it, requests bounce through multiple hops (remote GET). This is infrastructure that benefits ALL operations - implement it once, every operation inherits smart routing.

### Deliverables
- [ ] Hash function implementation (MurmurHash3)
- [ ] Consistent hash ring
- [ ] Segment ownership mapping
- [ ] Key-to-server routing
- [ ] Update client intelligence to 0x03 (HASH_DISTRIBUTION_AWARE)
- [ ] Topology updates with segment information

### Hash Topology Response

With intelligence = 0x03, server sends enhanced topology:
- Number of segments (vInt) - typically 256
- Number of owners per segment (uint8)
- For each segment (256 iterations):
  - List of owner server hash IDs (reference to servers from Step 4)

### Documentation
Reference: `hotrod-foundry/docs/05-consistent-hashing.md` (to be created)

Should include:
- MurmurHash3 implementation details
- Segment-based routing algorithm
- Hash topology wire format
- Key routing examples

### Test Vectors
Location: `hotrod-foundry/test-vectors/step-05-hashing/` (to be created)
- murmurhash3-test-cases.json (input → hash value)
- segment-ownership.json (segment mappings)
- key-routing-examples.json (key → server)

### Implementation

**MurmurHash3**:
```cpp
int32_t MurmurHash3::hash(const std::vector<uint8_t>& key) {
    // Implement MurmurHash3_x86_32
    // Seed = 0x9747b28c (Infinispan default)
    // Must match Java implementation exactly
}
```

**Segment routing**:
```cpp
class ConsistentHash {
    int numSegments;  // Usually 256
    std::vector<std::vector<Server*>> segmentOwners;  // [segment][owners]
    
    int getSegment(const std::vector<uint8_t>& key) {
        int hash = MurmurHash3::hash(key);
        return (hash & 0x7FFFFFFF) % numSegments;
    }
    
    Server* getPrimaryOwner(const std::vector<uint8_t>& key) {
        int segment = getSegment(key);
        return segmentOwners[segment][0];  // First owner = primary
    }
};
```

**Client Intelligence 0x03**:
- Request headers now set `client_intelligence = 0x03`
- Server responds with segment-aware topology
- Client routes each key to its primary owner

### Testing

**MurmurHash3 validation**:
Must match Java output exactly. Test with known inputs from test vectors.

**Routing validation**:
```bash
# 3-node cluster, 256 segments
# PUT key="foo" → hash → segment → route to owner
# Verify server receives request (not forwarded)
```

**Test cases**:
- Hash test vectors match expected values
- Segment calculation correct
- Routes to correct server for various keys
- Handles topology changes (rebalancing)
- Works with 1-node cluster (all segments on one server)
- Works with 3-node cluster (segments distributed)

### Java Reference
- `org.infinispan.client.hotrod.impl.consistenthash.*`
- `org.infinispan.commons.hash.MurmurHash3`

### Success Criteria
✅ MurmurHash3 matches Java implementation (test vectors pass)  
✅ Routes keys to correct server based on segments  
✅ Handles topology changes (rebalancing)  
✅ Client intelligence = 0x03 in requests  
✅ Works with clustered Infinispan (3+ nodes)
✅ Works on both platforms  
✅ Verified requests go to primary owner (not forwarded)

---

## ✨ Milestone: Foundation Complete

**After Step 5, you have a production-ready Hot Rod client foundation.**

**Can release**: v0.1.0  
**Includes**:
- Cross-platform support (Linux + Windows)
- Protocol 4.0 headers (complete spec)
- Authentication (SCRAM-SHA-256)
- Topology awareness (cluster failover)
- Consistent hashing (smart routing)

**Missing**: Only operations (GET, PUT, etc.)

**Benefit**: You can now add operations incrementally. Each operation automatically inherits:
- Authentication
- Topology-aware failover
- Smart routing to primary owner

**Next**: Add operations (Steps 6+) one at a time, release as v0.2, v0.3, etc.

---

## Step 6: PING Operation 🏓

**Goal**: First complete operation with full infrastructure

**Prerequisites**: Steps 0-5 complete (foundation ready)

**Why After Infrastructure?**
Now PING benefits from auth, topology, and routing. This validates the complete foundation before adding data operations.

### Deliverables
- [ ] TCP connection management
- [ ] PING request (opcode 0x17)
- [ ] PING response parsing (opcode 0x18)
- [ ] Connection error handling
- [ ] Works with authenticated connection (Step 3)
- [ ] Works with any server in cluster (Step 4)
- [ ] Integration tests with 3-node cluster

### Documentation
Reference: `hotrod-foundry/docs/06-ping-operation.md`
- PING purpose and use cases
- Request format (header only, no payload)
- Response format (header + status)
- Testing with authenticated connection

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

### Request Format
- Complete Protocol 4.0 header (from Step 2)
- Opcode: `0x17` (PING_REQUEST)
- No payload (header only)
- Media types: `0x00, 0x00` (none)
- Other param count: `0x00`

### Response Format
- Magic: `0xA1`
- Opcode: `0x18` (PING_RESPONSE)
- Status: `0x00` (success) or error code
- No payload

### Test Cases
- PING with authentication
- PING against each node in cluster
- Connection failover (stop node, PING fails over)
- Timeout handling
- Cross-validation: Wireshark capture matches Java PING

### Java Reference
- `org.infinispan.client.hotrod.impl.operations.PingOperation`

### Success Criteria
✅ PING works with authentication  
✅ PING works against any cluster node  
✅ Can failover if one node is down  
✅ Wire format matches Java (Wireshark verified)  
✅ Error handling works correctly
✅ All infrastructure features validated  
✅ Works on both platforms

**At this point**: Foundation complete, ready for production. Can release **v0.1.0** with just PING support.

---

## Step 7: GET Operation 🔍

**Goal**: First read operation with key serialization

**Benefits from infrastructure**:
- ✅ Authenticated connection (Step 3)
- ✅ Routes to primary owner (Step 5)
- ✅ Fails over if node down (Step 4)

### Deliverables
- [ ] GET request encoding (opcode 0x03)
- [ ] Key serialization (byte array)
- [ ] Response value parsing
- [ ] Handle "key not found" (status 0x01)
- [ ] Cross-client validation (Java PUT → Your GET)

### Request Format
- Protocol 4.0 header
- Opcode: `0x03` (GET_REQUEST)
- Key: length-prefixed byte array (vInt length + bytes)

### Response Format
- Status: `0x00` (found) or `0x01` (not found)
- Value: length-prefixed byte array (if found)

### Test Cases
- GET non-existent key → returns null
- Java client PUTs key, your client GETs → verify value
- GET with large key (>127 bytes, multi-byte vInt)
- GET routed to correct server (verify with Wireshark)

### Java Reference
- `org.infinispan.client.hotrod.impl.operations.GetOperation`

### Success Criteria
✅ Can retrieve values PUT by Java client  
✅ Correctly handles key not found  
✅ Routes to correct server (primary owner)
✅ Works on both platforms

---

## Step 8: PUT Operation ✍️

**Goal**: First write operation

### Deliverables
- [ ] PUT request (opcode 0x01)
- [ ] Key and value serialization
- [ ] Lifespan/maxIdle parameters (optional)
- [ ] Previous value in response (if exists)
- [ ] Cross-client validation (Your PUT → Java GET)

### Request Format
- Opcode: `0x01` (PUT_REQUEST)
- Lifespan (seconds): vInt (0 = immortal)
- MaxIdle (seconds): vInt (0 = no max idle)
- Key: byte array
- Value: byte array

### Response Format
- Status: `0x00` (success)
- Previous value: byte array (if existed)

### Test Cases
- PUT and verify with GET
- PUT with lifespan, verify expiration
- Cross-client: Your PUT → Java GET
- PUT updates existing key

### Java Reference
- `org.infinispan.client.hotrod.impl.operations.PutOperation`

### Success Criteria
✅ Can store values retrievable by Java client  
✅ Lifespan/maxIdle work correctly  
✅ Routes to correct server
✅ Works on both platforms

---

## Step 9: REMOVE Operation 🗑️

**Goal**: Delete operation

### Deliverables
- [ ] REMOVE request (opcode 0x0B)
- [ ] Return previous value (if existed)
- [ ] Handle "key not found"

### Java Reference
- `org.infinispan.client.hotrod.impl.operations.RemoveOperation`

---

## Step 10: Metadata Operations 📊

**Goal**: Version-based operations (Compare-And-Swap)

### Deliverables
- [ ] GET_WITH_METADATA (opcode 0x1B)
- [ ] REPLACE_IF_UNMODIFIED (opcode 0x09)
- [ ] Metadata structure (version, created, lifespan, etc.)

### Java Reference
- `org.infinispan.client.hotrod.impl.operations.GetWithMetadataOperation`
- `org.infinispan.client.hotrod.impl.operations.ReplaceIfUnmodifiedOperation`

---

## Step 11: Error Handling ⚠️

**Goal**: Proper error response handling

### Deliverables
- [ ] ERROR response parsing (opcode 0x50)
- [ ] Error message extraction (length-prefixed string)
- [ ] Exception hierarchy
- [ ] Retry logic for transient errors

### Java Reference
- `org.infinispan.client.hotrod.exceptions.*`

---

## Step 12: Bulk Operations 📦

**Goal**: Multi-key operations for efficiency

### Deliverables
- [ ] GET_ALL (opcode 0x2F)
- [ ] PUT_ALL (opcode 0x2D)
- [ ] BULK_GET (opcode 0x1F) - iterator-style

### Java Reference
- Bulk operation classes

---

## Step 13: Connection Pooling 🔄

**Goal**: Efficient connection management

### Deliverables
- [ ] Connection pool (reuse connections)
- [ ] Thread safety (concurrent requests)
- [ ] Connection lifecycle (health checks, timeouts)

### Java Reference
- `org.infinispan.client.hotrod.impl.transport.netty.ChannelPool`

---

## Step 14+: Advanced Features

**After core operations (Steps 6-13), consider**:

- Transactions (XA)
- Client listeners and events
- Counters
- Queries (Ickle)
- Streaming operations (Protocol 4.1+)
- Multimap operations
- Near caching

---

## Completion Criteria Per Step

For each step to be considered **complete**:

1. ✅ **Tests Pass**: All unit tests green on both platforms
2. ✅ **CI Passes**: GitHub Actions successful (Linux + Windows)
3. ✅ **Cross-Validation**: Wire format matches Java (Wireshark check)
4. ✅ **Test Vectors**: Byte-level validation against expected output
5. ✅ **Documentation**: Another language could implement using only the docs
6. ✅ **Progress Updated**: PROGRESS.md updated with step status

---

## Version Milestones

| Version | Steps Complete | Features | Release Readiness |
|---------|---------------|----------|-------------------|
| v0.1.0  | 0-6 (Foundation + PING) | Auth, topology, hashing, PING | ✅ Production-ready for health checks |
| v0.2.0  | + GET (Step 7) | Read operations | ✅ Production-ready for read-only caches |
| v0.3.0  | + PUT (Step 8) | Write operations | ✅ Production-ready for read/write |
| v0.4.0  | + REMOVE (Step 9) | Delete operations | ✅ Full CRUD support |
| v1.0.0  | 0-13 (All core ops) | Complete client | ✅ Feature-complete |

**Key insight**: Release v0.1.0 after Step 6. Foundation makes it production-ready even with only PING.

---

## Notes

**Infrastructure-first benefits**:
- Steps 0-5 build foundation (encoding, headers, auth, topology, hashing)
- Steps 6+ add operations (each inherits full infrastructure)
- Can release early (v0.1.0 after Step 6)
- Operations are simpler (no auth/routing logic needed)

**Start with Step 0 and proceed sequentially.** Each step assumes previous steps are complete.

**Cross-platform requirement**: Every step must work on both Linux and Windows. CI must test both.
