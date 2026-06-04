# Step 3: PING Operation

**Status**: First Network Operation  
**Prerequisite**: Steps 1-2 (Wire Primitives, Protocol Headers)  
**Estimated Time**: 3-4 hours  
**Test Vectors**: `../test-vectors/step-03-ping/`

## Overview

PING is the simplest Hot Rod operation - a health check that verifies server connectivity. This step implements:

1. **TCP connection** to Infinispan server
2. **Sending** a PING request
3. **Receiving** and parsing the response
4. **Connection management** (connect, send, receive, close)

**First real network I/O!** Previous steps were pure encoding/decoding.

## PING Operation

### Purpose

- Verify server is reachable and responsive
- Check protocol version compatibility
- Warmup connection before data operations
- Health monitoring

### Wire Format

**Request** (8 bytes for default cache):
```
0xA0              Magic byte (request)
0x01              Message ID (example: 1)
0x28              Version (Protocol 4.0)
0x17              Opcode (PING_REQUEST = 0x17)
0x00              Cache name (empty = default cache)
0x00              Flags (none)
0x01              Client intelligence (BASIC)
0x00              Topology ID (0)
```

**Response** (5 bytes minimum):
```
0xA1              Magic byte (response)
0x01              Message ID (matching request)
0x18              Opcode (PING_RESPONSE = 0x18)
0x00              Status (NO_ERROR)
0x00              Topology change marker (no change)
```

### No Payload

PING has **no request payload** and **no response payload** (beyond the header). It's just:
- Client: "Are you there?"
- Server: "Yes!" (or error)

---

## Connection Management

### TCP Socket Basics

Hot Rod uses **plain TCP sockets** (or TLS, but we'll start with plain).

**Connection Parameters**:
- **Host**: Server hostname/IP (default: `localhost`)
- **Port**: Hot Rod port (default: `11222`)
- **Timeout**: Connect/read timeout (recommended: 10-30 seconds)

### Connection Lifecycle

```
1. Connect    → Establish TCP connection
2. Send       → Write request bytes to socket
3. Receive    → Read response bytes from socket
4. Close      → Clean up connection
```

For PING, we can close after each operation. Later steps will implement connection pooling (Step 10).

---

## Implementation Guide

### Step 3A: Connection Class

Create a simple TCP connection wrapper.

#### C++17 Example

```cpp
class Connection {
public:
    /**
     * @brief Connect to Infinispan server
     * @param host Server hostname or IP
     * @param port Server port (default 11222)
     * @param timeoutMs Connection timeout in milliseconds
     */
    void connect(const std::string& host, int port, int timeoutMs = 10000);

    /**
     * @brief Send bytes to server
     * @param data Bytes to send
     */
    void send(const std::vector<uint8_t>& data);

    /**
     * @brief Receive bytes from server
     * @param maxBytes Maximum bytes to read
     * @return Received bytes
     */
    std::vector<uint8_t> receive(size_t maxBytes = 4096);

    /**
     * @brief Close connection
     */
    void close();

    /**
     * @brief Check if connected
     */
    bool isConnected() const;

private:
    int socketFd = -1;  // Socket file descriptor (POSIX)
    // Or SOCKET on Windows
};
```

**Platform Notes**:
- **Linux/macOS**: Use POSIX sockets (`<sys/socket.h>`, `<netinet/in.h>`)
- **Windows**: Use Winsock2 (`<winsock2.h>`, `<ws2tcpip.h>`)
- **Cross-platform**: Consider using a library (Boost.Asio, etc.) or platform-specific code

### Step 3B: PING Operation

Combine connection + headers to implement PING.

#### C++17 Example

```cpp
class RemoteCache {
public:
    /**
     * @brief Ping server to check connectivity
     * @param host Server hostname
     * @param port Server port
     * @return true if server responds successfully
     */
    bool ping(const std::string& host = "localhost", int port = 11222);

private:
    uint64_t nextMessageId = 1;
};

bool RemoteCache::ping(const std::string& host, int port) {
    // 1. Build PING request
    RequestHeader req(nextMessageId++, HotRodConstants::PING_REQUEST);
    
    std::vector<uint8_t> requestBytes;
    encodeRequestHeader(requestBytes, req);

    // 2. Connect to server
    Connection conn;
    conn.connect(host, port);

    // 3. Send request
    conn.send(requestBytes);

    // 4. Receive response header (at least 5 bytes)
    std::vector<uint8_t> responseBytes = conn.receive(1024);

    // 5. Decode response
    const uint8_t* ptr = responseBytes.data();
    ResponseHeader resp = decodeResponseHeader(ptr);

    // 6. Validate response
    if (resp.messageId != req.messageId) {
        throw std::runtime_error("Message ID mismatch");
    }

    if (resp.opcode != HotRodConstants::PING_RESPONSE) {
        throw std::runtime_error("Unexpected opcode");
    }

    // 7. Clean up
    conn.close();

    // 8. Return success if status is NO_ERROR
    return resp.status == HotRodConstants::NO_ERROR_STATUS;
}
```

---

## Platform-Specific Socket Code

### POSIX Sockets (Linux/macOS)

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <netdb.h>

void Connection::connect(const std::string& host, int port, int timeoutMs) {
    // Create socket
    socketFd = socket(AF_INET, SOCK_STREAM, 0);
    if (socketFd < 0) {
        throw std::runtime_error("Failed to create socket");
    }

    // Set timeout
    struct timeval tv;
    tv.tv_sec = timeoutMs / 1000;
    tv.tv_usec = (timeoutMs % 1000) * 1000;
    setsockopt(socketFd, SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(tv));
    setsockopt(socketFd, SOL_SOCKET, SO_SNDTIMEO, &tv, sizeof(tv));

    // Resolve hostname
    struct hostent* server = gethostbyname(host.c_str());
    if (server == nullptr) {
        ::close(socketFd);
        throw std::runtime_error("Failed to resolve host");
    }

    // Setup address
    struct sockaddr_in serverAddr;
    std::memset(&serverAddr, 0, sizeof(serverAddr));
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_port = htons(port);
    std::memcpy(&serverAddr.sin_addr.s_addr, server->h_addr, server->h_length);

    // Connect
    if (::connect(socketFd, (struct sockaddr*)&serverAddr, sizeof(serverAddr)) < 0) {
        ::close(socketFd);
        throw std::runtime_error("Failed to connect to " + host + ":" + std::to_string(port));
    }
}

void Connection::send(const std::vector<uint8_t>& data) {
    if (socketFd < 0) {
        throw std::runtime_error("Not connected");
    }

    size_t totalSent = 0;
    while (totalSent < data.size()) {
        ssize_t sent = ::send(socketFd, data.data() + totalSent, 
                             data.size() - totalSent, 0);
        if (sent < 0) {
            throw std::runtime_error("Send failed");
        }
        totalSent += sent;
    }
}

std::vector<uint8_t> Connection::receive(size_t maxBytes) {
    if (socketFd < 0) {
        throw std::runtime_error("Not connected");
    }

    std::vector<uint8_t> buffer(maxBytes);
    ssize_t received = ::recv(socketFd, buffer.data(), maxBytes, 0);
    
    if (received < 0) {
        throw std::runtime_error("Receive failed");
    }
    
    if (received == 0) {
        throw std::runtime_error("Connection closed by server");
    }

    buffer.resize(received);
    return buffer;
}

void Connection::close() {
    if (socketFd >= 0) {
        ::close(socketFd);
        socketFd = -1;
    }
}
```

### Windows Sockets (Winsock2)

```cpp
#include <winsock2.h>
#include <ws2tcpip.h>
#pragma comment(lib, "ws2_32.lib")

// Similar structure, using SOCKET instead of int
// Call WSAStartup() before first use, WSACleanup() on shutdown
// Use closesocket() instead of close()
```

---

## Testing Strategy

### Unit Tests (Mock Socket)

For unit tests, mock the socket I/O to avoid requiring a real server.

**Test cases**:
- Build correct PING request bytes
- Parse PING response bytes
- Handle connection errors
- Handle invalid responses
- Message ID correlation

### Integration Tests (Real Server)

For integration tests, connect to actual Infinispan server.

**Prerequisites**:
1. Infinispan server running on `localhost:11222`
2. Server accessible (no firewall blocking)
3. Server healthy and responsive

**Test cases**:
- PING default cache (empty cache name)
- PING named cache
- PING with wrong host (connection failure expected)
- PING with wrong port (connection failure expected)
- Multiple PINGs in sequence

### Running Infinispan Server

#### Using Docker

```bash
docker run -d -p 11222:11222 infinispan/server:latest
```

Wait for server to start (~5-10 seconds):
```bash
# Check if server is ready
curl http://localhost:11222
```

#### Manual Download

```bash
wget https://downloads.jboss.org/infinispan/15.0.0.Final/infinispan-server-15.0.0.Final.zip
unzip infinispan-server-15.0.0.Final.zip
cd infinispan-server-15.0.0.Final
./bin/server.sh
```

---

## Wire Format Validation

### Using Wireshark

1. **Install hotrod-dissector**:
   ```bash
   cp /home/rigazilla/git/hotrod-dissector/src/hotrod.lua \
      ~/.local/lib/wireshark/plugins/
   ```

2. **Start Wireshark** capturing on loopback (`lo`) interface

3. **Filter**: `hotrod`

4. **Run your client PING**

5. **Compare**:
   - Request bytes should match expected format
   - Response should parse correctly
   - Verify with Java client side-by-side

### Expected Wireshark Output

```
Request (→):
  Magic: 0xa0 (Request)
  Message ID: 1
  Version: 40 (Protocol 4.0)
  Opcode: 0x17 (PING)
  Cache Name: (empty)
  Flags: 0
  Client Intelligence: 1 (BASIC)
  Topology ID: 0

Response (←):
  Magic: 0xa1 (Response)
  Message ID: 1
  Opcode: 0x18 (PING_RESPONSE)
  Status: 0x00 (NO_ERROR)
  Topology Change: 0 (no change)
```

---

## Java Reference

**File**: `org.infinispan.client.hotrod.impl.operations.NoCachePingOperation`

```java
@Override
public short requestOpCode() {
    return HotRodConstants.PING_REQUEST;  // 0x17
}

@Override
public short responseOpCode() {
    return HotRodConstants.PING_RESPONSE;  // 0x18
}
```

**No request body** - just the header  
**No response body** - just the header (status indicates success)

---

## Common Pitfalls

### 1. Not Reading Full Response

❌ **Don't**: Assume response is exactly 5 bytes  
✅ **Do**: Read response header, check for topology change data

If `topologyChangeMarker == 1`, more data follows (handle in Step 11).

### 2. Forgetting to Close Connections

❌ **Don't**: Leave sockets open  
✅ **Do**: Always close in finally block or use RAII (C++)

### 3. Message ID Mismatch

❌ **Don't**: Reuse same message ID  
✅ **Do**: Increment message ID for each request

### 4. Blocking Indefinitely

❌ **Don't**: Use infinite timeout  
✅ **Do**: Set reasonable timeout (10-30 seconds)

### 5. Assuming localhost

❌ **Don't**: Hardcode `localhost`  
✅ **Do**: Accept host/port as parameters

---

## Error Handling

### Connection Errors

```cpp
try {
    conn.connect(host, port);
} catch (const std::exception& e) {
    // Server not reachable
    // - Wrong host/port
    // - Server not running
    // - Firewall blocking
    // - Network issue
}
```

### Protocol Errors

```cpp
try {
    ResponseHeader resp = decodeResponseHeader(ptr);
    
    if (resp.status != NO_ERROR_STATUS) {
        // Server returned error
        // - Server internal error
        // - Unknown command
        // - Protocol version mismatch
    }
} catch (const HotRodException& e) {
    // Invalid magic byte
    // - Corrupt data
    // - Wrong protocol
    // - Not a Hot Rod server
}
```

---

## Implementation Checklist

- [ ] Connection class
  - [ ] connect() method
  - [ ] send() method
  - [ ] receive() method
  - [ ] close() method
  - [ ] Error handling
- [ ] RemoteCache class
  - [ ] ping() method
  - [ ] Message ID generation
- [ ] Unit tests
  - [ ] PING request encoding
  - [ ] PING response decoding
  - [ ] Error cases
- [ ] Integration tests
  - [ ] PING against real server
  - [ ] Connection failure handling
- [ ] Wireshark validation
  - [ ] Compare with Java client
  - [ ] Verify wire format

---

## Next Step

Once PING works:
- ✅ Mark Step 3 complete in `PROGRESS.md`
- ➡️ Proceed to **Step 4: GET Operation** (first data operation!)

## References

- **Java Implementation**: `NoCachePingOperation.java`
- **POSIX Sockets**: `man 2 socket`, `man 2 connect`
- **Winsock**: Microsoft Winsock2 documentation
- **Wireshark**: Use hotrod-dissector for protocol analysis
