# Development Guidelines

## Project Structure Requirements

Each language implementation MUST be a **standalone, isolated project** in its own repository:

### Repository Structure
```
your-language-client/
├── README.md                    # Project-specific README
├── PROGRESS.md                  # Implementation progress tracker
├── LICENSE                      # License file
├── .gitignore                   # Language-specific ignores
├── src/                         # Source code
├── tests/                       # Test suite
├── .github/workflows/           # CI/CD pipeline
│   └── build.yml               # Automated builds + tests
└── docs/                        # Implementation-specific docs
    └── (can link to hotrod-foundry docs)
```

**Separation of Concerns:**
- **hotrod-foundry/** = Language-agnostic documentation + test vectors
- **your-client/** = Language-specific implementation + progress tracking

### Progress Tracking (PROGRESS.md)

**REQUIRED**: Every implementation must maintain `PROGRESS.md`

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
   - Test against live Infinispan server
   - Auto-skip if server unavailable (for CI)
   - Document required Docker setup

### Test Coverage Requirements

**Minimum per step:**
- ✅ Unit tests for all test vectors
- ✅ Unit tests for error cases
- ✅ Integration tests (can auto-skip)
- ✅ All tests passing before marking step complete

**Example (C#):**
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

## CI/CD Requirements

**REQUIRED**: Every implementation must have automated CI/CD

### GitHub Actions (Recommended)

**Minimum workflow (`.github/workflows/build.yml`):**

```yaml
name: Build and Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
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
```

**CI Must:**
- ✅ Run on every push and PR
- ✅ Build the project
- ✅ Run all unit tests
- ✅ Run integration tests (with Dockerized Infinispan)
- ✅ Report test coverage
- ✅ Fail if any test fails

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

## Code Quality Standards

**Must-Have:**
- ✅ No compiler warnings
- ✅ Linter passes (language-specific)
- ✅ Test coverage ≥ 80%
- ✅ All public APIs documented
- ✅ No hardcoded credentials/secrets

**Nice-to-Have:**
- Static analysis tools (SonarQube, etc.)
- Security scanning
- Dependency vulnerability checks
- Performance benchmarks
