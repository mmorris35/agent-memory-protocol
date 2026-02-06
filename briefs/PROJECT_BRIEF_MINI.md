# PROJECT_BRIEF.md

## Basic Information

- **Project Name**: amp-mini
- **Project Type**: cli (HTTP server)
- **Primary Goal**: Lightweight AMP (Agent Memory Protocol) implementation with semantic search using all-MiniLM-L6-v2 embeddings. Designed to run on Raspberry Pi 4 or similar modest hardware.
- **Target Users**: Home server deployments, small teams, self-hosted AMP/MESH nodes
- **Timeline**: 2 weeks
- **Team Size**: 1

## Functional Requirements

### Key Features (MVP)

- Full AMP protocol compliance (store, search, get, delete, list, status)
- Semantic search via all-MiniLM-L6-v2 embeddings (384 dimensions)
- SQLite with sqlite-vec extension for vector storage
- FTS5 fallback for keyword search
- ONNX Runtime for embedding inference
- HTTP REST API on configurable port
- Embedding generation on store (async, non-blocking)
- Similarity search endpoint (find similar to record X)
- Batch embedding for bulk imports

### Nice-to-Have Features (v2)

- TLS support
- Basic authentication / API keys
- MESH protocol integration (signing, revocation, federation)
- Prometheus metrics
- Hybrid search (combine semantic + keyword scores)
- Embedding caching for repeated queries
- Quantized INT8 model option for smaller footprint

## Technical Constraints

### Must Use

- Rust (stable toolchain)
- SQLite with sqlite-vec extension
- ONNX Runtime (ort crate) for embeddings
- all-MiniLM-L6-v2 model (ONNX format)
- Axum HTTP framework
- Tokio runtime (multi-threaded)
- Serde for JSON serialization

### Cannot Use

- Python or Python-based ML frameworks
- GPU requirements (must run on CPU)
- More than 1.5GB RAM under normal operation
- External embedding services/APIs

## Model Specification

```yaml
model: sentence-transformers/all-MiniLM-L6-v2
format: ONNX
dimensions: 384
max_sequence_length: 256
size: ~80MB
quantization: FP32 (INT8 optional for v2)
```

Model should be bundled or downloaded on first run.

## Performance Requirements

- Binary size: <20MB (excluding model)
- Model size: ~80MB
- RAM usage: <1.5GB working set with 10K records
- Startup time: <5 seconds (model loading)
- Embedding latency: <50ms per text on Pi 4
- Search latency: <100ms for 10K records
- Storage: ~2KB per lesson (content + embedding)

## Target Hardware

Primary: Raspberry Pi 4 (4GB)
- ARM Cortex-A72 (quad-core 1.5GHz)
- 4GB RAM (2GB minimum)
- USB SSD recommended
- Ethernet networking

Must also run on:
- Raspberry Pi 5
- Any ARM64 with 2GB+ RAM
- Any x86_64 with 2GB+ RAM
- Apple Silicon (M1+)

## Build Requirements

```bash
# Native build
cargo build --release

# Cross-compile for Pi 4
cross build --release --target aarch64-unknown-linux-gnu

# Download model (one-time)
./scripts/download-model.sh
```

## API Specification

Full AMP protocol plus semantic extensions:

```
POST /amp/store           - Store record (generates embedding async)
POST /amp/search          - Semantic search (embeds query, vector search)
GET  /amp/records/{id}    - Get record by ID
DELETE /amp/records/{id}  - Delete record (and embedding)
GET  /amp/records         - List records with filters
GET  /amp/status          - Health, stats, model info

# Semantic extensions
POST /amp/similar/{id}    - Find records similar to given ID
GET  /amp/embedding/{id}  - Get raw embedding vector
POST /amp/embed           - Generate embedding for arbitrary text
```

## Database Schema

```sql
-- Main records table
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
    updated_at INTEGER,
    embedding_status TEXT DEFAULT 'pending'  -- pending, complete, failed
);

-- Vector storage (sqlite-vec)
CREATE VIRTUAL TABLE record_embeddings USING vec0(
    id TEXT PRIMARY KEY,
    embedding FLOAT[384]
);

-- FTS5 for keyword fallback
CREATE VIRTUAL TABLE records_fts USING fts5(
    title, content, tags,
    content='records',
    content_rowid='rowid'
);
```

## Embedding Pipeline

```
1. Record stored → embedding_status = 'pending'
2. Background task picks up pending records
3. Text extracted (title + content)
4. Tokenized, truncated to 256 tokens
5. ONNX inference → 384-dim vector
6. Vector stored in sqlite-vec
7. embedding_status = 'complete'
```

Search:
```
1. Query text → embedding
2. Vector similarity search (cosine)
3. Return top-K results with scores
```

## Configuration

```yaml
port: 8765
host: "0.0.0.0"
database: "./amp-mini.db"
model_path: "./models/all-MiniLM-L6-v2.onnx"
log_level: "info"
embedding_threads: 2
search_limit_default: 20
min_similarity: 0.5
```

## Testing Requirements

- Unit tests for all AMP operations
- Integration tests with real SQLite + embeddings
- Embedding quality tests (known similar pairs)
- Memory usage tests (must stay under 1.5GB)
- Cross-compilation smoke test on Pi 4
- Search relevance tests (semantic should beat keyword for conceptual queries)

## Success Criteria

1. All AMP protocol operations pass compliance tests
2. Semantic search finds conceptually similar content (e.g., "database issues" finds "connection pooling")
3. Runs on Pi 4 (2GB) with <1.5GB RAM
4. Search returns results in <100ms
5. Can store and search 10,000 lessons
6. Embedding generation <50ms per record
7. Clean shutdown preserves all data and embeddings

## Comparison: amp-pico vs amp-mini

| Feature | amp-pico | amp-mini |
|---------|----------|----------|
| Search type | Keyword (FTS5) | Semantic + Keyword |
| RAM usage | <50MB | <1.5GB |
| Hardware | Pi Zero | Pi 4+ |
| Binary size | <2MB | <20MB |
| Model size | 0 | ~80MB |
| "Understands" meaning | ❌ | ✅ |

## Out of Scope

- Code-aware chunking (use nellie-rs)
- File watching/indexing (use nellie-rs)
- MCP protocol (HTTP only for simplicity)
- GPU acceleration
- Multiple embedding models
- Fine-tuning or training
