# PROJECT_BRIEF.md

## Basic Information

- **Project Name**: amp-pico
- **Project Type**: cli (HTTP server)
- **Primary Goal**: Minimal AMP (Agent Memory Protocol) implementation using SQLite FTS5 for keyword search. Designed to run on resource-constrained hardware like Raspberry Pi Zero.
- **Target Users**: Edge deployments, IoT, personal knowledge bases, offline-first scenarios
- **Timeline**: 1 week
- **Team Size**: 1

## Functional Requirements

### Key Features (MVP)

- Full AMP protocol compliance (store, search, get, delete, list, status)
- SQLite FTS5 full-text search (keyword, boolean, phrase, prefix)
- Single-binary deployment with embedded SQLite
- HTTP REST API on configurable port
- JSON request/response format per AMP spec
- Graceful shutdown with database sync
- Health check endpoint
- Record types: lesson, checkpoint, snippet (extensible)

### Nice-to-Have Features (v2)

- TLS support (via rustls)
- Basic authentication
- MESH protocol integration (signed lessons, revocation)
- Prometheus metrics endpoint
- Systemd service file generator
- ARM32 support (Pi Zero v1)

## Technical Constraints

### Must Use

- Rust (stable toolchain)
- SQLite with FTS5 extension (bundled, not system)
- Axum or similar minimal HTTP framework
- Tokio runtime (single-threaded acceptable)
- Serde for JSON serialization

### Cannot Use

- Any embedding model or ML runtime
- External database services
- More than 50MB RAM under normal operation
- Dependencies requiring system libraries (fully static build)

## Performance Requirements

- Binary size: <2MB (stripped, release build)
- RAM usage: <50MB working set with 1000 records
- Startup time: <1 second
- Search latency: <50ms for 10K records
- Storage: ~1KB per lesson average

## Target Hardware

Primary: Raspberry Pi Zero 2W
- ARM Cortex-A53 (quad-core 1GHz)
- 512MB RAM
- SD card or USB storage
- WiFi networking

Must also run on:
- Any ARMv7+ (armhf, aarch64)
- Any x86_64
- Any system with glibc 2.17+ or musl

## Build Requirements

```bash
# Native build
cargo build --release

# Cross-compile for Pi Zero 2W
cross build --release --target aarch64-unknown-linux-musl

# Resulting binary should be <2MB
```

## API Specification

Implement AMP protocol exactly as specified:

```
POST /amp/store         - Store a record
POST /amp/search        - Search records (FTS5 query)
GET  /amp/records/{id}  - Get record by ID
DELETE /amp/records/{id} - Delete record
GET  /amp/records       - List records with filters
GET  /amp/status        - Health and stats
```

See: https://github.com/mmorris35/agent-memory-protocol

## Database Schema

```sql
CREATE TABLE records (
    id TEXT PRIMARY KEY,
    type TEXT NOT NULL,
    content TEXT NOT NULL,
    title TEXT,
    tags TEXT,  -- JSON array
    severity TEXT,
    agent TEXT,
    project TEXT,
    created_at INTEGER NOT NULL,
    updated_at INTEGER
);

CREATE VIRTUAL TABLE records_fts USING fts5(
    title,
    content,
    tags,
    content='records',
    content_rowid='rowid'
);

-- Triggers to keep FTS in sync
```

## Configuration

```yaml
# config.yaml or environment variables
port: 8765
host: "0.0.0.0"
database: "./amp-pico.db"
log_level: "info"
```

## Testing Requirements

- Unit tests for all AMP operations
- Integration tests with real SQLite database
- FTS5 search quality tests (boolean, phrase, prefix)
- Memory usage tests (must stay under 50MB)
- Cross-compilation smoke test

## Success Criteria

1. All AMP protocol operations pass compliance tests
2. Binary runs on Pi Zero 2W with <50MB RAM
3. Binary size under 2MB
4. Search returns results in <50ms
5. Can store and retrieve 10,000 lessons without issue
6. Clean shutdown preserves all data
7. Single static binary, no external dependencies

## Out of Scope

- Semantic/vector search (use amp-mini for that)
- MCP protocol support
- Web UI
- Multi-user authentication
- Replication/clustering
- File watching/indexing
