# Agent Memory Protocol (AMP)

> **A standard protocol for persistent memory in AI agent systems**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Spec Version](https://img.shields.io/badge/spec-v0.1--draft-orange.svg)](SPECIFICATION.md)

---

## The Problem

AI agents are smart but forgetful. Every session starts from zero. Context windows are finite. Hard-won lessons vanish when the conversation ends.

Current solutions are fragmented:
- **Letta/MemGPT** — Great hierarchical model, framework-specific
- **Mem0** — Memory-as-a-service, proprietary API
- **Zep** — Knowledge graphs, their own protocol
- **MCP memory servers** — Dozens of incompatible implementations

There's no standard. No interoperability. Vendor lock-in everywhere.

## The Solution

**Agent Memory Protocol (AMP)** defines a standard interface for agent memory operations. Like [MCP](https://modelcontextprotocol.io) standardized tool integration, AMP standardizes knowledge persistence.

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Claude    │     │    GPT-4    │     │   Gemini    │
│   Agent     │     │   Agent     │     │   Agent     │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                    ┌──────▼──────┐
                    │     AMP     │  ← Standard Protocol
                    │  Interface  │
                    └──────┬──────┘
                           │
       ┌───────────────────┼───────────────────┐
       │                   │                   │
┌──────▼──────┐     ┌──────▼──────┐     ┌──────▼──────┐
│   Nellie    │     │   Your DB   │     │  Cloud Svc  │
│  (Rust/SQL) │     │  (Postgres) │     │   (S3+λ)    │
└─────────────┘     └─────────────┘     └─────────────┘
```

**Any agent. Any backend. One protocol.**

---

## 🌡️ Killer Use Case: Sovereign IoT Sensor Network

AMP isn't just for AI agents. It's for **anything that produces data worth remembering**.

**The old way (cloud-dependent):**
```
Sensor → Cloud API → Their Database → Their Dashboard
               └── Your data on their server
               └── Internet required 24/7
               └── Monthly subscription forever
               └── "Service discontinued" email someday
```

**The AMP way (local-first):**
```
┌─────────────────────────────────────────────────────────┐
│                   Your Network                          │
│                                                         │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐        │
│  │Temp Sensor │  │ Humidity   │  │Power Meter │        │
│  │(amp-storage)│  │(amp-storage)│  │(amp-storage)│        │
│  │  $5 ESP32  │  │ $10 Pi Zero│  │  $5 ESP32  │        │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘        │
│        └───────────────┼───────────────┘               │
│                        ▼                                │
│        ┌───────────────────────────────┐               │
│        │  MESH Directory (optional)    │               │
│        │  Discovers all local sensors  │               │
│        └───────────────┬───────────────┘               │
│                        ▼                                │
│        Query from anywhere on your network:            │
│        $ mesh search "garage temperature"              │
│        $ mesh search "humidity > 60%"                  │
└─────────────────────────────────────────────────────────┘
```

**Each sensor runs amp-storage (Tier -1):**
- Stores readings locally with TTL (auto-expires old data)
- Serves data on demand
- No cloud dependency, no subscription, no data harvesting
- Runs on $5 hardware

**But here's the magic: it doesn't have to be on your local network.**

Combined with [MESH Protocol](https://github.com/mmorris35/mesh-protocol), data is **encrypted with keys you control**. Sensors can be anywhere—public internet, cellular, remote locations:

```
🏔️ Remote Cabin        🏠 Rental Property      🚗 Vehicle
   (satellite)            (4G LTE)              (cellular)
        │                      │                     │
        └──────────────────────┼─────────────────────┘
                               │
                        PUBLIC INTERNET
                        (data encrypted)
                               │
                        ┌──────▼──────┐
                        │    MESH     │
                        │  Directory  │
                        └──────┬──────┘
                               │
                    Only YOUR keys decrypt
                               │
                        ┌──────▼──────┐
                        │ Your Phone  │
                        │ Anywhere    │
                        └─────────────┘
```

- **Share access?** Give someone a key.
- **Revoke access?** Delete the key. Cryptographically enforced.
- **Sensor compromised?** Data's encrypted. Attacker gets noise.

**Zero-trust IoT. Your sensors, anywhere. Your data, encrypted. Your keys, your rules.**

This is the IoT we were promised.

---

## Core Concepts

### Memory Types

| Type | Purpose | Example |
|------|---------|---------|
| **Lesson** | Learned knowledge, patterns, gotchas | "Always null-check before .length" |
| **Checkpoint** | Agent state at a point in time | "Working on auth module, blocked on API key" |
| **Snippet** | Code/content with semantic index | Function definitions, docs |

### Operations

```
store    → Add or update a memory record
search   → Semantic search across records
get      → Retrieve record by ID
delete   → Remove a record
list     → Enumerate with filters
status   → Health check and stats
```

### Example: Store a Lesson

```json
{
  "operation": "store",
  "record": {
    "type": "lesson",
    "title": "PostgreSQL connection pooling",
    "content": "Always use connection pooling in production. PgBouncer or PgCat recommended for high-traffic apps.",
    "tags": ["postgresql", "devops", "performance"],
    "severity": "warning"
  }
}
```

### Example: Semantic Search

```json
{
  "operation": "search",
  "query": "database connection issues in production",
  "type": "lesson",
  "limit": 5
}
```

Returns semantically similar lessons—not keyword matching, actual understanding.

## Transport Bindings

AMP is transport-agnostic. Implementations exist for:

| Transport | Example |
|-----------|---------|
| **HTTP/REST** | `POST /amp/store`, `POST /amp/search` |
| **MCP** | `amp_store`, `amp_search` tools |
| **CLI** | `amp store --type lesson --title "..."` |

## Design Principles

1. **Simplicity** — Store, search, retrieve, delete. No PhD required.
2. **Semantic-First** — Search by meaning, not keywords.
3. **Local by Default** — Your memory stays on your machine unless you choose otherwise.
4. **Agent-Centric** — Built for how agents actually work: checkpoints, lessons, code context.

## Specification

📄 **[Full Specification](SPECIFICATION.md)** — Complete protocol definition with schemas and examples.

## Implementations

AMP has been battle-tested in production systems:

| Implementation | Status | Notes |
|----------------|--------|-------|
| **Nellie** | ✅ Production | Rust-based reference implementation. SQLite + vector search. |
| *(your impl here)* | — | PRs welcome! |

> *We've been running AMP-compliant systems for months, accumulating thousands of lessons and checkpoints across multiple AI agents. The protocol emerged from real-world needs, not theoretical design.*

## Quick Start

### For Agent Developers

Point your agent at any AMP-compliant server:

```python
import httpx

# Store a lesson
httpx.post("http://localhost:8765/amp/store", json={
    "operation": "store",
    "record": {
        "type": "lesson",
        "title": "API rate limits",
        "content": "OpenAI API has a 10k TPM limit on free tier..."
    }
})

# Search later
results = httpx.post("http://localhost:8765/amp/search", json={
    "operation": "search",
    "query": "rate limiting issues"
}).json()
```

### For Backend Developers

Implement the six core operations. See [SPECIFICATION.md](SPECIFICATION.md) for schemas.

Minimal viable implementation:
1. `store` — Write to any database
2. `search` — Embed query, cosine similarity against stored embeddings
3. `get` / `delete` / `list` — Standard CRUD
4. `status` — Return `{"healthy": true}`

## Roadmap

- [x] Core specification (v0.1)
- [ ] JSON Schema definitions
- [ ] OpenAPI spec for HTTP binding
- [ ] MCP server template
- [ ] Test suite for compliance
- [ ] Memory federation protocol (v0.2)

## Contributing

AMP is an open standard. We welcome:

- **Feedback** on the spec — [Open an issue](https://github.com/mmorris35/agent-memory-protocol/issues)
- **Implementations** in any language — Add to the implementations table
- **Extensions** for domain-specific needs — Propose via PR

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## FAQ

**Why not just use MCP's memory server?**

MCP is great for tools. But there's no standard *memory* protocol—each MCP memory server has a different API. AMP fills that gap.

**Why not use Letta/Mem0/Zep directly?**

They're excellent products with rich features. AMP is a *lowest-common-denominator* protocol that allows interoperability. Use their advanced features; expose AMP for compatibility.

**Is this trying to replace those systems?**

No. AMP is a protocol, not a product. Letta could expose an AMP interface. Mem0 could add AMP compatibility. Everyone benefits from standardization.

**Why semantic search as the default?**

Because agents think in concepts, not keywords. "How do I handle database connection issues?" should find "PostgreSQL connection pooling" even if the exact words don't match.

## License

MIT License. Use it, extend it, build on it.

---

<p align="center">
  <i>Built from real-world agent development experience.</i><br>
  <i>Standardize memory. Free the agents.</i>
</p>
