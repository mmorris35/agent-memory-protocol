# AMP Reference Implementations

AMP is a protocol, not a product. Multiple implementations can coexist, targeting different hardware and use cases.

## Implementation Tiers

| Tier | Codename | Target Hardware | Search | RAM | Use Case |
|------|----------|-----------------|--------|-----|----------|
| **Tier 0** | amp-pico | Pi Zero 2W | Keyword (FTS5) | <50MB | Edge, IoT, personal |
| **Tier 1** | amp-mini | Pi 4 (2GB+) | Semantic | ~1GB | Home server, small team |
| **Tier 2** | nellie-rs | Dev server | Semantic + Code | 2GB+ | Developer, enterprise |

All tiers implement the same AMP protocol. Clients can't tell the difference (except search quality).

---

## Tier 0: amp-pico

> *"Runs on a potato"*

### Target Hardware

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| Board | Pi Zero 2W | Pi Zero 2W |
| RAM | 512MB | 512MB |
| Storage | 8GB SD | 32GB+ USB |
| Network | WiFi | WiFi/Ethernet |
| Cost | ~$15 | ~$25 |

### Technical Specifications

```yaml
binary_size: <2MB
ram_usage: <50MB working set
language: Rust
database: SQLite 3.x
search: FTS5 (full-text search)
transport: HTTP/1.1
embedding_model: none
```

### Search Capabilities

| Query Type | Supported | Example |
|------------|-----------|---------|
| Exact match | ✅ | `"connection pool"` |
| Boolean | ✅ | `rust AND cargo NOT nightly` |
| Prefix | ✅ | `postgre*` |
| Phrase | ✅ | `"parallel builds"` |
| Semantic | ❌ | "database issues" → "connection pooling" |

### API Endpoints

```
GET  /amp/status
POST /amp/store
POST /amp/search
GET  /amp/records/{id}
DELETE /amp/records/{id}
GET  /amp/records?type=lesson&limit=10
```

### Dependencies

```toml
[dependencies]
rusqlite = { version = "0.31", features = ["bundled"] }
axum = "0.7"
tokio = { version = "1", features = ["rt", "macros"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

Zero external services. Single binary. Single SQLite file.

### Limitations

- No semantic search (keyword only)
- No code-aware chunking
- No embedding generation
- Limited concurrent connections

### Ideal For

- Personal knowledge base
- Edge/IoT deployments
- Offline-first scenarios
- Resource-constrained environments
- "I just want to store and retrieve lessons"

---

## Tier 1: amp-mini

> *"Semantic search on a Pi 4"*

### Target Hardware

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| Board | Pi 4 (2GB) | Pi 4 (4GB) / Pi 5 |
| RAM | 2GB | 4GB |
| Storage | 32GB USB SSD | 128GB+ USB SSD |
| Network | Ethernet | Ethernet |
| Cost | ~$50 | ~$80 |

### Technical Specifications

```yaml
binary_size: ~15MB (includes ONNX runtime)
ram_usage: ~1GB working set
language: Rust
database: SQLite 3.x + sqlite-vec
search: Semantic (vector similarity) + FTS5 fallback
transport: HTTP/1.1
embedding_model: all-MiniLM-L6-v2 (ONNX)
embedding_dimensions: 384
```

### Search Capabilities

| Query Type | Supported | Example |
|------------|-----------|---------|
| Exact match | ✅ | `"connection pool"` |
| Boolean | ✅ | `rust AND cargo` |
| Semantic | ✅ | "database issues" → "PostgreSQL pooling" |
| Similarity | ✅ | "find similar to lesson X" |

### Embedding Model

**all-MiniLM-L6-v2** chosen for:
- Size: 80MB (vs 274MB for nomic-embed-text)
- Speed: <30ms per embedding on CPU
- Quality: Strong MTEB benchmark scores
- Compatibility: ONNX export, runs on ARM64

```
Model: sentence-transformers/all-MiniLM-L6-v2
Format: ONNX (quantized INT8 optional)
Dimensions: 384
Max tokens: 256
```

### RAM Breakdown

```
Component              RAM Usage
─────────────────────────────────
OS overhead            ~200MB
ONNX Runtime           ~100MB
MiniLM model           ~100MB
SQLite + sqlite-vec    ~200MB (base)
Vector index           ~50MB per 10K lessons
Working headroom       ~300MB
─────────────────────────────────
Total (10K lessons)    ~1GB
```

### API Endpoints

Same as amp-pico, plus:

```
POST /amp/similar/{id}    # Find similar records
GET  /amp/embedding/{id}  # Get raw embedding vector
```

### Dependencies

```toml
[dependencies]
rusqlite = { version = "0.31", features = ["bundled"] }
sqlite-vec = "0.1"
ort = "2.0"                    # ONNX Runtime
axum = "0.7"
tokio = { version = "1", features = ["rt-multi-thread", "macros"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

### Limitations

- Smaller embedding model than Nellie-RS (trade-off: size vs quality)
- No code-aware chunking
- No file watching/indexing
- Single-node only (no built-in replication)

### Ideal For

- Home server knowledge base
- Small team shared memory
- Self-hosted AMP node
- MESH network participant
- "I want semantic search without a big server"

---

## Tier 2: nellie-rs

> *"Full-featured developer tool"*

Reference implementation for developers and enterprises. See [nellie-rs repository](https://github.com/mmorris35/nellie-rs).

### Target Hardware

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| Machine | Any x86_64/ARM64 | Mac Mini M1+ / Linux server |
| RAM | 4GB | 8GB+ |
| Storage | SSD | NVMe SSD |
| Network | Gigabit | Gigabit |

### Additional Features (vs amp-mini)

- Larger embedding model (nomic-embed-text, 768 dimensions)
- Code-aware chunking and indexing
- File system watching
- MCP protocol support
- Project/repository scoping
- Agent checkpointing

---

## Compliance

All tiers MUST implement the core AMP operations:

| Operation | amp-pico | amp-mini | nellie-rs |
|-----------|----------|----------|-----------|
| `store` | ✅ | ✅ | ✅ |
| `search` | ✅ (FTS5) | ✅ (semantic) | ✅ (semantic) |
| `get` | ✅ | ✅ | ✅ |
| `delete` | ✅ | ✅ | ✅ |
| `list` | ✅ | ✅ | ✅ |
| `status` | ✅ | ✅ | ✅ |

A client using the AMP protocol should work with any tier. Search *quality* varies, but the API is identical.

---

## Choosing a Tier

```
Do you have a Pi Zero or similar?
  └─► amp-pico

Do you have a Pi 4+ or small server?
  └─► amp-mini

Are you a developer who wants code search + MCP?
  └─► nellie-rs

Are you building something custom?
  └─► Implement the AMP spec yourself
```

---

*The protocol is the constant. The implementation fits your hardware.*
