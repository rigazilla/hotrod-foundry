# Infinispan Hot Rod Client Implementation Guide

This repository provides comprehensive, language-agnostic documentation for implementing Infinispan Hot Rod protocol clients in any programming language.

## Purpose

Enable developers (with AI coding assistants like Claude Code) to implement production-ready Hot Rod clients through:
- Step-by-step incremental development
- Language-agnostic documentation
- Test-driven approach mirroring Java reference implementation
- Validated test vectors for each step

## Quick Start

**Implementing a new client?** Start here:

1. Read [ROADMAP.md](ROADMAP.md) - Understand the incremental steps
2. Check [PROGRESS.md](PROGRESS.md) - See current status
3. Follow documentation in order: `docs/01-wire-format-primitives.md` → `docs/02-protocol-headers.md` → etc.
4. Use test vectors in `test-vectors/` to validate your implementation
5. Reference Java implementation: `../infinispan/client/hotrod-client/`

**Starting an AI session?** Read [SESSION_START.md](SESSION_START.md) first.

## Repository Structure

```
├── ROADMAP.md              # Step-by-step implementation plan
├── PROGRESS.md             # Current progress across all implementations
├── SESSION_START.md        # AI assistant session startup guide
├── docs/                   # Language-agnostic documentation
│   ├── 01-wire-format-primitives.md
│   ├── 02-protocol-headers.md
│   ├── ... (one per step)
│   └── language-guides/    # Language-specific recommendations
├── reference/              # Reference materials
│   ├── java-client-mapping.md
│   ├── operation-codes.md
│   └── kaitai-schemas/     # Links to hotrod-dissector schemas
├── test-vectors/           # Test data for validation
│   ├── step-01-primitives/
│   ├── step-02-headers/
│   └── ...
└── implementations/        # Links to separate implementation repos
    ├── cpp.md              # → infinispan-hotrod-cpp
    └── csharp.md           # → infinispan-hotrod-csharp
```

## Reference Sources

This documentation is derived from:

1. **Java Reference Implementation**: `/home/rigazilla/git/infinispan/client/hotrod-client/`
   - Canonical, production-tested implementation
   - Package: `org.infinispan.client.hotrod.*`

2. **Kaitai Struct Schemas**: `/home/rigazilla/git/hotrod-dissector/schemas/`
   - Formal protocol specification
   - Files: `hotrod30.ksy`, `hotrod40.ksy`

3. **Protocol Documentation**: [Infinispan Hot Rod Protocol Spec](https://infinispan.org/docs/stable/titles/hotrod_protocol/hotrod_protocol.html)

## Implementation Repositories

Each implementation is a **separate, standalone repository** with its own:
- Build system and CI/CD
- Documentation (linking back to this repo)
- Test suite (mirroring Java tests)
- Examples and getting started guides
- PROGRESS.md tracking implementation status

Current implementations:

- [**C++17 Client**](implementations/cpp17.md) → `../cpp17-client/`
- [**C# Client**](implementations/csharp.md) → `../csharp-client/` *(planned)*

## Development Approach

### Incremental Steps

Each step:
1. **Small & focused** - Completable in 1-3 hours
2. **Testable** - Has concrete validation criteria
3. **Language-agnostic** - Documentation works for any language
4. **Java-validated** - Test vectors derived from Java client behavior

### Workflow

```
For each step:
  1. Read language-agnostic documentation
  2. Identify Java reference classes
  3. Implement in target language
  4. Port Java tests
  5. Validate against test vectors
  6. Update PROGRESS.md
  7. Move to next step
```

### Cross-Language Validation

Documentation is validated when:
- C++ implementation works using only the docs
- C# implementation can replicate without looking at C++ code
- Wire format matches Java byte-for-byte (verified with Wireshark)

## Progress Tracking

See [PROGRESS.md](PROGRESS.md) for:
- Overall completion status
- Per-implementation progress
- Current focus and next steps
- Session handoff notes

## Contributing

To add a new language implementation:

1. Create separate repository (e.g., `infinispan-hotrod-go`)
2. Follow the ROADMAP steps in order
3. Validate against test vectors
4. Add link in `implementations/`
5. Update PROGRESS.md

## Resources

- **Infinispan Project**: https://infinispan.org/
- **Hot Rod Protocol Spec**: https://infinispan.org/docs/stable/titles/hotrod_protocol/hotrod_protocol.html
- **Java Client Source**: https://github.com/infinispan/infinispan
- **Wireshark Dissector**: https://github.com/rigazilla/hotrod-dissector
- **Kaitai Struct**: https://kaitai.io/

## License

[To be determined - likely Apache 2.0 to match Infinispan]

---

**Maintained by**: Infinispan community
**Status**: Active development
**Current Step**: 1 (Wire Format Primitives)
