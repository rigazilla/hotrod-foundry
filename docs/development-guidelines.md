# Development Guidelines

## Project Structure Requirements

Each language port MUST be a **standalone, isolated project** in its own repository:

### Repository Structure
```
your-language-client/
├── README.md                    # Project-specific README
├── PROGRESS.md                  # Porting progress tracker
├── LICENSE                      # License file
├── .gitignore                   # Language-specific ignores
├── src/                         # Source code
├── tests/                       # Test suite
├── .github/workflows/           # CI/CD pipeline
│   └── build.yml               # Automated builds + tests
└── docs/                        # Port-specific docs
    └── (can link to hotrod-foundry docs)
```

**Separation of Concerns:**
- **hotrod-foundry/** = Language-agnostic documentation + test vectors
- **your-client/** = Language-specific port + progress tracking

### Progress Tracking (PROGRESS.md)

**REQUIRED**: Every port must maintain `PROGRESS.md`

See `templates/PROGRESS.md.template` for the standard format.

**Update Frequency**: After completing each step or at least weekly during active development.

## Test-Driven Development (MANDATORY)

**Rule**: For EACH step, tests come first:

### TDD Workflow

1. **Read Test Vectors**
   ```
   test-vectors/step-XX-feature/*.json
   ```

2. **Write Unit Tests**
   - Load test vectors from JSON
   - Test each primitive/feature in isolation
   - All tests RED (failing) initially

3. **Implement Feature**
   - Write minimal code to pass tests
   - Validate byte-by-byte against test vectors

4. **Validate**
   - All unit tests GREEN
   - Test vectors pass 100%
   - ONLY THEN proceed to integration tests

5. **Integration Tests**
   - Use **Testcontainers** library to start Infinispan server (MANDATORY)
   - Test against live server instance
   - Automatic cleanup after tests

### Test Coverage Requirements

**Minimum per step:**
- ✅ Unit tests for all test vectors
- ✅ Unit tests for error cases
- ✅ Integration tests using Testcontainers
- ✅ All tests passing before marking step complete

**Unit Test Example (C#):**
```csharp
[Fact]
public void TestVInt_FromTestVectors()
{
    var vectors = TestVectorLoader.Load("step-01-primitives/vint-test-cases.json");
    foreach (var test in vectors.TestCases)
    {
        var buffer = new List<byte>();
        Codec.WriteVInt(buffer, test.Input);
        Assert.Equal(test.ExpectedBytes, buffer.ToArray());
    }
}
```

**Integration Test Example with Testcontainers (C#):**
```csharp
using Testcontainers.Infinispan;

public class IntegrationTests : IAsyncLifetime
{
    private InfinispanContainer _container;
    
    public async Task InitializeAsync()
    {
        _container = new InfinispanBuilder()
            .WithImage("infinispan/server:16.0")
            .WithUsername("admin")
            .WithPassword("password")
            .Build();
        
        await _container.StartAsync();
    }
    
    [Fact]
    public void TestPingAgainstRealServer()
    {
        var client = new RemoteCache(
            _container.GetHost(),
            _container.GetMappedPublicPort(11222),
            "admin",
            "password"
        );
        
        Assert.True(client.Ping());
    }
    
    public async Task DisposeAsync()
    {
        await _container.DisposeAsync();
    }
}
```

**Integration Test Example with Testcontainers (Java):**
```java
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.utility.DockerImageName;

@Testcontainers
class IntegrationTest {
    
    @Container
    static GenericContainer<?> infinispan = new GenericContainer<>(
        DockerImageName.parse("infinispan/server:16.0"))
        .withEnv("USER", "admin")
        .withEnv("PASS", "password")
        .withExposedPorts(11222);
    
    @Test
    void testPing() {
        String host = infinispan.getHost();
        int port = infinispan.getMappedPort(11222);
        
        RemoteCache cache = new RemoteCache(host, port, "admin", "password");
        assertTrue(cache.ping());
    }
}
```

**Integration Test Example with Testcontainers (Python):**
```python
from testcontainers.core.container import DockerContainer

class TestIntegration:
    
    @pytest.fixture(scope="class")
    def infinispan_container(self):
        with DockerContainer("infinispan/server:16.0") \
            .with_env("USER", "admin") \
            .with_env("PASS", "password") \
            .with_exposed_ports(11222) as container:
            yield container
    
    def test_ping(self, infinispan_container):
        host = infinispan_container.get_container_host_ip()
        port = infinispan_container.get_exposed_port(11222)
        
        cache = RemoteCache(host, port, "admin", "password")
        assert cache.ping()
```

**Why Testcontainers?**
- ✅ Automatic server lifecycle (start/stop)
- ✅ Isolated test environment
- ✅ Works in CI without manual setup
- ✅ Consistent across all languages
- ✅ No manual Docker commands needed

**Testcontainers libraries**:
- C#: `Testcontainers` NuGet package
- Java: `org.testcontainers:testcontainers`
- Python: `testcontainers` PyPI package
- Go: `github.com/testcontainers/testcontainers-go`
- Node.js: `testcontainers` npm package

## CI/CD Requirements

**REQUIRED**: Every port must have automated CI/CD

**MANDATORY**: CI must test on BOTH Linux and Windows

### Platform Support

**Required platforms:**
- ✅ **Linux**: Fedora/RHEL (primary), Ubuntu/Debian (secondary)
- ✅ **Windows**: Windows 10/11

**Why both?**
- Infinispan is used in enterprise environments (RHEL)
- Developers use Windows workstations
- Cross-platform validation catches platform-specific bugs

### GitHub Actions (Recommended)

**Minimum workflow (`.github/workflows/build.yml`):**

```yaml
name: Build and Test

on: [push, pull_request]

jobs:
  test-linux:
    runs-on: ubuntu-latest  # Represents Fedora/RHEL/Debian family
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup [Language]
        # Language-specific setup
      
      - name: Build
        run: # Build command
      
      - name: Unit Tests
        run: # Run unit tests (no server needed)
      
      - name: Start Infinispan
        run: |
          docker run -d -p 11222:11222 \
            -e USER=admin -e PASS=password \
            infinispan/server:16.0
          sleep 20
      
      - name: Integration Tests
        run: # Run integration tests
      
      - name: Test Coverage
        run: # Generate coverage report

  test-windows:
    runs-on: windows-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup [Language]
        # Language-specific setup (Windows paths, etc.)
      
      - name: Build
        run: # Build command (Windows-compatible)
      
      - name: Unit Tests
        run: # Run unit tests
      
      - name: Start Infinispan (Windows)
        run: |
          docker run -d -p 11222:11222 `
            -e USER=admin -e PASS=password `
            infinispan/server:16.0
          Start-Sleep -Seconds 20
      
      - name: Integration Tests
        run: # Run integration tests
```

**CI Must:**
- ✅ Run on every push and PR
- ✅ Test on BOTH Linux and Windows (separate jobs)
- ✅ Build the project on both platforms
- ✅ Run all unit tests on both platforms
- ✅ Run integration tests (with Dockerized Infinispan) on both platforms
- ✅ Report test coverage
- ✅ Fail if any test fails on either platform

## Documentation Requirements

**For each step you complete, ensure:**

1. **Code Comments**
   - Why, not what (code shows what)
   - Non-obvious decisions
   - Links to spec: "See RFC 5802 section 3.1"

2. **README Updates**
   - Installation instructions
   - Basic usage examples
   - Current feature set

3. **PROGRESS.md Updates**
   - Mark step complete
   - Note any deviations from reference
   - Update test results

4. **API Documentation**
   - Public API documented (Javadoc/XML docs/docstrings)
   - Examples for each major feature

## Git Workflow

**Branch Strategy:**
- `main` branch = stable, released code
- Feature branches: `step-XX-feature-name`
- One PR per step (for reviewability)

**Commit Messages:**
```
Step 3: Implement SCRAM-SHA-256 authentication

- Add AuthMechListOperation (opcode 0x21/0x22)
- Implement SCRAM-SHA-256 mechanism per RFC 5802
- Add PBKDF2 and HMAC-SHA256 helpers
- All auth test vectors passing (15/15)

Closes #12
```

## Release Process

**Versioning**: Semantic Versioning (semver.org)

**Version Milestones:**
- **v0.1.0**: Steps 0-5 complete (production-ready foundation, PING only)
- **v0.2.0**: Add GET operation
- **v0.3.0**: Add PUT operation
- **v1.0.0**: Steps 0-9 complete (core operations stable)

**Release Checklist:**
- [ ] All tests passing
- [ ] CI green
- [ ] PROGRESS.md updated
- [ ] README reflects current features
- [ ] CHANGELOG.md updated
- [ ] Tagged in git
- [ ] Published to package registry (NuGet/Maven/PyPI/etc.)

## Cross-Platform Coding Standards

**MANDATORY**: Code must work on both Linux and Windows

### Platform-Specific Code

**Avoid when possible**:
```cpp
// ❌ BAD: POSIX-only
#include <unistd.h>
sleep(1);

// ✅ GOOD: Cross-platform
#include <thread>
#include <chrono>
std::this_thread::sleep_for(std::chrono::seconds(1));
```

**File paths**:
```python
# ❌ BAD: Hard-coded separator
path = "data/config/server.conf"

# ✅ GOOD: Platform-independent
import os
path = os.path.join("data", "config", "server.conf")
```

**When platform-specific code is unavoidable**:
```cpp
#ifdef _WIN32
    // Windows port
#else
    // Linux/Unix port
#endif
```

### Cross-Platform Dependencies

**Preferred libraries** (available on both platforms):
- **Networking**: Standard library sockets, Boost.Asio
- **Crypto**: OpenSSL (available on both)
- **Testing**: Google Test, xUnit, pytest (all cross-platform)
- **Build**: CMake, Gradle, npm (all cross-platform)

**Avoid**:
- Linux-only: `epoll`, `inotify`
- Windows-only: Windows-specific APIs without POSIX fallback

### Testing on Both Platforms

**Local testing**:
```bash
# Linux (Fedora/RHEL)
sudo dnf install cmake gcc g++ openssl-devel
cmake . && make && ctest

# Windows (PowerShell)
choco install cmake visualstudio2022-workload-vctools openssl
cmake . && cmake --build . && ctest
```

**CI/CD**: See CI/CD Requirements section - both platforms MUST pass.

## Code Quality Standards

**Must-Have:**
- ✅ No compiler warnings (on both Linux and Windows)
- ✅ Linter passes (language-specific)
- ✅ Test coverage ≥ 80% (on both platforms)
- ✅ All public APIs documented
- ✅ No hardcoded credentials/secrets
- ✅ Works on Linux (Fedora/RHEL) and Windows

**Nice-to-Have:**
- Static analysis tools (SonarQube, etc.)
- Security scanning
- Dependency vulnerability checks
- Performance benchmarks
