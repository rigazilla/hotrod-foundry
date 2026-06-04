# Hot Rod Operation Codes Reference

## Request Operation Codes

| Opcode | Hex | Name | Description |
|--------|-----|------|-------------|
| 0x01 | 0x01 | PUT | Store key-value pair |
| 0x03 | 0x03 | GET | Retrieve value by key |
| 0x05 | 0x05 | PUT_IF_ABSENT | Store only if key doesn't exist |
| 0x07 | 0x07 | REPLACE | Replace existing value |
| 0x09 | 0x09 | REPLACE_IF_UNMODIFIED | CAS operation (compare-and-swap) |
| 0x0B | 0x0B | REMOVE | Delete entry |
| 0x0D | 0x0D | REMOVE_IF_UNMODIFIED | Conditional remove |
| 0x0F | 0x0F | CONTAINS_KEY | Check if key exists |
| 0x13 | 0x13 | CLEAR | Clear entire cache |
| 0x15 | 0x15 | STATS | Get server statistics |
| 0x17 | 0x17 | PING | Health check |
| 0x1B | 0x1B | GET_WITH_METADATA | Get value with version/metadata |
| 0x29 | 0x29 | SIZE | Get cache size |
| 0x2D | 0x2D | PUT_ALL | Bulk put operation |
| 0x2F | 0x2F | GET_ALL | Bulk get operation |

## Response Operation Codes

**Pattern**: Response opcode = Request opcode + 1 (except ERROR)

| Opcode | Hex | Name | Corresponding Request |
|--------|-----|------|----------------------|
| 0x02 | 0x02 | PUT_RESPONSE | PUT |
| 0x04 | 0x04 | GET_RESPONSE | GET |
| 0x06 | 0x06 | PUT_IF_ABSENT_RESPONSE | PUT_IF_ABSENT |
| 0x08 | 0x08 | REPLACE_RESPONSE | REPLACE |
| 0x0A | 0x0A | REPLACE_IF_UNMODIFIED_RESPONSE | REPLACE_IF_UNMODIFIED |
| 0x0C | 0x0C | REMOVE_RESPONSE | REMOVE |
| 0x0E | 0x0E | REMOVE_IF_UNMODIFIED_RESPONSE | REMOVE_IF_UNMODIFIED |
| 0x10 | 0x10 | CONTAINS_KEY_RESPONSE | CONTAINS_KEY |
| 0x14 | 0x14 | CLEAR_RESPONSE | CLEAR |
| 0x16 | 0x16 | STATS_RESPONSE | STATS |
| 0x18 | 0x18 | PING_RESPONSE | PING |
| 0x1C | 0x1C | GET_WITH_METADATA_RESPONSE | GET_WITH_METADATA |
| 0x2A | 0x2A | SIZE_RESPONSE | SIZE |
| 0x2E | 0x2E | PUT_ALL_RESPONSE | PUT_ALL |
| 0x30 | 0x30 | GET_ALL_RESPONSE | GET_ALL |
| 0x50 | 0x50 | ERROR_RESPONSE | Any (error occurred) |

## Status Codes

### Success Codes (0x00-0x7F)

| Code | Hex | Name | Meaning |
|------|-----|------|---------|
| 0x00 | 0x00 | NO_ERROR | Operation succeeded |
| 0x01 | 0x01 | NOT_PUT_REMOVED_REPLACED | Operation didn't apply (e.g., PUT_IF_ABSENT when key exists) |
| 0x02 | 0x02 | KEY_DOES_NOT_EXIST | Key not found |
| 0x03 | 0x03 | SUCCESS_WITH_PREVIOUS | Success, previous value returned in response |
| 0x04 | 0x04 | NOT_EXECUTED_WITH_PREVIOUS | Not executed, previous value returned |

### Error Codes (0x80+)

| Code | Hex | Name | Meaning |
|------|-----|------|---------|
| 0x81 | 0x81 | INVALID_MAGIC_OR_MESSAGE_ID | Protocol error |
| 0x82 | 0x82 | UNKNOWN_COMMAND | Unrecognized opcode |
| 0x83 | 0x83 | UNKNOWN_VERSION | Unsupported protocol version |
| 0x84 | 0x84 | REQUEST_PARSING_ERROR | Malformed request |
| 0x85 | 0x85 | SERVER_ERROR | Internal server error |
| 0x86 | 0x86 | COMMAND_TIMEOUT | Operation timed out |
| 0x87 | 0x87 | NODE_SUSPECTED | Server node suspected failed |
| 0x88 | 0x88 | ILLEGAL_LIFECYCLE_STATE | Server not ready |

## Client Intelligence Levels

| Value | Hex | Name | Description |
|-------|-----|------|-------------|
| 0x01 | 0x01 | BASIC | No topology awareness, random server |
| 0x02 | 0x02 | TOPOLOGY_AWARE | Knows server list, failover support |
| 0x03 | 0x03 | HASH_DISTRIBUTION_AWARE | Smart routing with consistent hashing |

## Flags (Bitwise OR)

| Flag | Hex | Name | Description |
|------|-----|------|-------------|
| 0x0001 | 0x0001 | FORCE_RETURN_VALUE | Return previous value |
| 0x0002 | 0x0002 | DEFAULT_LIFESPAN | Use server default lifespan |
| 0x0004 | 0x0004 | DEFAULT_MAXIDLE | Use server default max idle time |
| 0x0008 | 0x0008 | SKIP_CACHE_LOAD | Don't load from cache loader |
| 0x0010 | 0x0010 | SKIP_INDEXING | Don't index this entry |

## References

- **Java Source**: `org.infinispan.client.hotrod.impl.protocol.HotRodConstants`
- **Protocol Spec**: https://infinispan.org/docs/stable/titles/hotrod_protocol/
