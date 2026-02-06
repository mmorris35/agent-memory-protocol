# PROJECT_BRIEF.md

## Basic Information

- **Project Name**: amp-storage
- **Project Type**: cli (HTTP server)
- **Primary Goal**: Minimal AMP-compliant storage node with NO local search capability. Stores and serves records; relies on MESH network for discovery. Designed for extreme resource constraints.
- **Target Users**: Edge nodes, IoT, MESH network participants who want to contribute content without running search infrastructure
- **Timeline**: 3 days
- **Team Size**: 1

## Functional Requirements

### Key Features (MVP)

- AMP protocol compliance for storage operations only
- Store records with full AMP schema
- Serve records by ID
- Delete records
- List records with basic filters (type, limit, offset)
- Status endpoint with health and record counts
- Search endpoint returns 501 Not Implemented (protocol compliant, just empty)
- SQLite persistence (single file)
- HTTP REST API
- MESH-ready: serves signed content when requested

### Nice-to-Have Features (v2)

- MESH announcement integration (auto-announce new records)
- TLS support
- Basic auth
- Prometheus metrics
- mDNS/Zeroconf discovery

## Technical Constraints

### Must Use

- Rust (stable toolchain)
- SQLite (bundled, minimal)
- Minimal HTTP framework (tiny-http or hyper directly)
- Single-threaded async acceptable

### Cannot Use

- Any search indexing (no FTS5, no vectors, no embedding models)
- External services
- More than 20MB RAM under normal operation
- Dependencies requiring system libraries

## Performance Requirements

- Binary size: <1MB (stripped, release build)
- RAM usage: <20MB working set
- Startup time: <500ms
- GET latency: <10ms
- Storage: ~500 bytes per lesson average (no index overhead)

## Target Hardware

Primary: Raspberry Pi Zero W (original)
- ARM1176JZF-S (single-core 1GHz)
- 512MB RAM
- SD card storage
- WiFi

Also targets:
- ESP32 (future, with no_std port)
- Any microcontroller with TCP stack
- Docker sidecar (tiny footprint)

## API Specification

```
POST /amp/store           - Store a record (201 Created)
GET  /amp/records/{id}    - Get record by ID (200 OK / 404)
DELETE /amp/records/{id}  - Delete record (204 No Content / 404)
GET  /amp/records         - List records (?type=&limit=&offset=)
GET  /amp/status          - Health and stats

POST /amp/search          - Returns 501 Not Implemented
                            Body: {"error": "search_not_supported", 
                                   "message": "This node does not support local search. Use MESH directory."}
```

## Database Schema

```sql
-- Minimal schema, no indexes beyond primary key
CREATE TABLE records (
    id TEXT PRIMARY KEY,
    type TEXT NOT NULL,
    data TEXT NOT NULL,  -- Full JSON blob
    created_at INTEGER NOT NULL
);

-- Optional: type index for filtered listing
CREATE INDEX idx_records_type ON records(type);
```

Note: No FTS tables, no vector tables, no search indexes.

## Configuration

```yaml
port: 8765
host: "0.0.0.0"
database: "./amp-storage.db"
log_level: "warn"  # Minimal logging for resource constraints
```

Or environment variables only (no config file parsing needed).

## Testing Requirements

- Unit tests for store/get/delete/list/status
- Verify search returns 501 correctly
- Memory usage tests (must stay under 20MB)
- Binary size verification (<1MB)
- Stress test: 1000 rapid store/get cycles

## Success Criteria

1. Binary under 1MB
2. RAM under 20MB with 1000 records
3. All storage operations work correctly
4. Search returns 501 (not 500, not empty 200)
5. Runs on Pi Zero W (original, not 2W)
6. Can store/retrieve 10,000 records
7. GET by ID in <10ms

## Comparison: amp-storage vs amp-pico vs amp-mini

| Feature | amp-storage | amp-pico | amp-mini |
|---------|-------------|----------|----------|
| Local search | ❌ | ✅ FTS5 | ✅ Semantic |
| Binary size | <1MB | <2MB | <20MB |
| RAM usage | <20MB | <50MB | <1.5GB |
| Hardware | Pi Zero v1 | Pi Zero 2W | Pi 4 |
| Discovery | MESH only | Local + MESH | Local + MESH |
| Use case | Content server | Edge search | Home server |

## Architecture

```
┌─────────────────────────────────┐
│         amp-storage             │
│  ┌─────────────────────────┐   │
│  │   HTTP Handler          │   │
│  │   (tiny-http, <100 LOC) │   │
│  └───────────┬─────────────┘   │
│              │                  │
│  ┌───────────▼─────────────┐   │
│  │   SQLite                │   │
│  │   (single table)        │   │
│  └─────────────────────────┘   │
└─────────────────────────────────┘

No search. No indexing. Just storage.
```

## Out of Scope

- Any form of search (keyword, semantic, or otherwise)
- MCP protocol
- File watching
- Batch operations
- Multi-database support
- Replication
