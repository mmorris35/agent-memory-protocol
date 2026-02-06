# Agent Memory Protocol (AMP)

> **Draft Specification v0.1** | February 2026

A standard protocol for persistent memory in AI agent systems. Enables agents to store, recall, and manage knowledge across sessions with interoperable backends.

## Abstract

AI agents face a fundamental limitation: context windows are finite and sessions are ephemeral. Current solutions are fragmented—each framework implements memory differently, creating vendor lock-in and preventing interoperability.

The Agent Memory Protocol (AMP) defines a standard interface for agent memory operations, allowing any compliant client to work with any compliant backend. Like MCP standardized tool integration, AMP standardizes knowledge persistence.

## Status

This document is a **draft specification** seeking community feedback. Reference implementation: [Nellie](https://github.com/mmorris35/nellie-rs).

---

## 1. Design Principles

### 1.1 Simplicity
Memory operations should be intuitive. Store, search, retrieve, delete. No PhD required.

### 1.2 Transport Agnostic
AMP defines operations, not transport. Implementations may use HTTP, MCP stdio, gRPC, or other mechanisms.

### 1.3 Semantic-First
Search is primarily semantic (embeddings-based), with optional filters. Exact-match is a special case, not the default.

### 1.4 Agent-Centric
Designed for how agents actually work: checkpointing state, learning lessons, searching code context, managing per-project knowledge.

### 1.5 Privacy by Default
Memory stores are assumed local/private unless explicitly configured otherwise. No cloud requirement.

---

## 2. Core Concepts

### 2.1 Memory Types

AMP defines three primary memory types:

| Type | Purpose | Typical Lifetime | Example |
|------|---------|------------------|---------|
| **Lesson** | Learned knowledge, patterns, gotchas | Long-term | "Always check for null before accessing .length" |
| **Checkpoint** | Agent state at a point in time | Medium-term | "Working on auth module, blocked on API key" |
| **Snippet** | Code/content with semantic index | Varies | Function definitions, documentation blocks |

Implementations MAY support additional types but MUST support at least `lesson`.

### 2.2 Memory Record

All memory records share common fields:

```typescript
interface MemoryRecord {
  id: string;              // Unique identifier (server-generated)
  type: "lesson" | "checkpoint" | "snippet" | string;
  content: string;         // Primary content (indexed for semantic search)
  
  // Metadata
  created_at: number;      // Unix timestamp (ms)
  updated_at?: number;     // Unix timestamp (ms)
  agent?: string;          // Agent identifier
  project?: string;        // Project/repo scope
  tags?: string[];         // User-defined tags
  
  // Type-specific fields
  [key: string]: any;
}
```

### 2.3 Lesson Record

```typescript
interface Lesson extends MemoryRecord {
  type: "lesson";
  title: string;           // Short descriptive title
  content: string;         // Detailed explanation
  severity?: "info" | "warning" | "critical";
  tags?: string[];
}
```

### 2.4 Checkpoint Record

```typescript
interface Checkpoint extends MemoryRecord {
  type: "checkpoint";
  agent: string;           // Required for checkpoints
  working_on: string;      // Current task description
  state?: {
    decisions?: string[];  // Key decisions made
    blockers?: string[];   // Current blockers
    artifacts?: string[];  // Files/outputs created
    flags?: string[];      // Status flags (IDLE, IN_PROGRESS, BLOCKED)
  };
  session_id?: string;     // Optional session correlation
}
```

### 2.5 Snippet Record

```typescript
interface Snippet extends MemoryRecord {
  type: "snippet";
  content: string;         // The code/text content
  file_path?: string;      // Source file path
  language?: string;       // Programming language
  start_line?: number;     // Line number in source
  end_line?: number;
  repo?: string;           // Repository identifier
}
```

---

## 3. Operations

### 3.1 Overview

| Operation | Description | Required |
|-----------|-------------|----------|
| `store` | Add or update a memory record | MUST |
| `search` | Semantic search across records | MUST |
| `get` | Retrieve record by ID | MUST |
| `delete` | Remove a record | MUST |
| `list` | List records with filters | SHOULD |
| `status` | Server health and statistics | SHOULD |

### 3.2 store

Adds a new record or updates an existing one (upsert by ID).

**Request:**
```json
{
  "operation": "store",
  "record": {
    "type": "lesson",
    "title": "Check array bounds before access",
    "content": "Always verify array indices are within bounds...",
    "tags": ["javascript", "arrays", "defensive"],
    "severity": "warning"
  }
}
```

**Response:**
```json
{
  "success": true,
  "id": "lesson_a1b2c3d4",
  "created": true
}
```

### 3.3 search

Performs semantic search with optional filters.

**Request:**
```json
{
  "operation": "search",
  "query": "how to handle null pointer exceptions",
  "type": "lesson",           // Optional: filter by type
  "tags": ["java"],           // Optional: filter by tags
  "project": "my-project",    // Optional: scope to project
  "limit": 10,                // Optional: max results (default: 10)
  "min_score": 0.7            // Optional: similarity threshold (0-1)
}
```

**Response:**
```json
{
  "success": true,
  "results": [
    {
      "id": "lesson_x1y2z3",
      "score": 0.92,
      "record": {
        "type": "lesson",
        "title": "Null safety patterns",
        "content": "...",
        "tags": ["java", "null-safety"]
      }
    }
  ],
  "total": 1
}
```

### 3.4 get

Retrieves a single record by ID.

**Request:**
```json
{
  "operation": "get",
  "id": "lesson_a1b2c3d4"
}
```

**Response:**
```json
{
  "success": true,
  "record": {
    "id": "lesson_a1b2c3d4",
    "type": "lesson",
    "title": "...",
    "content": "...",
    "created_at": 1707235200000
  }
}
```

### 3.5 delete

Removes a record.

**Request:**
```json
{
  "operation": "delete",
  "id": "lesson_a1b2c3d4"
}
```

**Response:**
```json
{
  "success": true,
  "deleted": true
}
```

### 3.6 list

Lists records with optional filters. For bulk enumeration, not semantic search.

**Request:**
```json
{
  "operation": "list",
  "type": "checkpoint",
  "agent": "radarr",
  "limit": 20,
  "offset": 0,
  "order": "desc"            // By created_at
}
```

**Response:**
```json
{
  "success": true,
  "records": [...],
  "total": 45,
  "has_more": true
}
```

### 3.7 status

Returns server health and statistics.

**Request:**
```json
{
  "operation": "status"
}
```

**Response:**
```json
{
  "success": true,
  "healthy": true,
  "version": "1.0.0",
  "stats": {
    "lessons": 142,
    "checkpoints": 58,
    "snippets": 12847,
    "embedding_model": "nomic-embed-text"
  }
}
```

---

## 4. Transport Bindings

AMP operations can be bound to various transports.

### 4.1 HTTP/REST

```
POST /amp/store
POST /amp/search  
GET  /amp/records/{id}
DELETE /amp/records/{id}
GET  /amp/records?type=lesson&limit=10
GET  /amp/status
```

### 4.2 MCP

Exposed as MCP tools:

```json
{
  "tools": [
    {"name": "amp_store", "description": "Store a memory record"},
    {"name": "amp_search", "description": "Search memory semantically"},
    {"name": "amp_get", "description": "Get record by ID"},
    {"name": "amp_delete", "description": "Delete a record"},
    {"name": "amp_list", "description": "List records with filters"},
    {"name": "amp_status", "description": "Get server status"}
  ]
}
```

### 4.3 CLI

```bash
amp store --type lesson --title "..." --content "..."
amp search "null pointer handling" --type lesson
amp get lesson_a1b2c3d4
amp delete lesson_a1b2c3d4
amp list --type checkpoint --agent radarr
amp status
```

---

## 5. Semantic Search Requirements

### 5.1 Embedding Model

Implementations MUST support semantic search via embeddings. The embedding model is implementation-defined but SHOULD be documented.

Recommended models:
- `nomic-embed-text` (open, local)
- `text-embedding-3-small` (OpenAI)
- `voyage-code-2` (optimized for code)

### 5.2 Similarity Metric

Cosine similarity is RECOMMENDED. Implementations MAY use other metrics but MUST document them.

### 5.3 Hybrid Search

Implementations MAY combine semantic search with keyword/BM25 search for improved results.

---

## 6. Scoping and Multi-Tenancy

### 6.1 Agent Scope

Checkpoints MUST be scoped to an agent identifier. Other types MAY be scoped.

### 6.2 Project Scope

Records MAY be scoped to a project/repository. Useful for code memory that should only surface in relevant contexts.

### 6.3 Multi-Agent Access

Implementations SHOULD support multiple agents accessing shared memory (e.g., team lessons) while maintaining agent-specific records (e.g., checkpoints).

---

## 7. Security Considerations

### 7.1 Local-First

AMP implementations SHOULD default to local storage. Cloud sync MUST be opt-in and explicit.

### 7.2 No Sensitive Data in Search

Implementations SHOULD NOT log or store raw queries in ways that could leak sensitive information.

### 7.3 Access Control

Multi-user implementations SHOULD support access control for shared memory stores.

---

## 8. Compatibility with Existing Systems

### 8.1 Nellie

Nellie's current API maps directly to AMP:

| Nellie Tool | AMP Operation |
|-------------|---------------|
| `add_lesson` | `store` (type: lesson) |
| `search_lessons` | `search` (type: lesson) |
| `add_checkpoint` | `store` (type: checkpoint) |
| `get_recent_checkpoints` | `list` (type: checkpoint) |
| `search_code` | `search` (type: snippet) |
| `get_status` | `status` |

### 8.2 MCP Memory Servers

Existing MCP memory servers can adopt AMP by implementing the standard operations. Wrapper adapters can bridge proprietary APIs.

### 8.3 Letta/Mem0/Zep

These systems have richer models (knowledge graphs, memory hierarchies). AMP provides a lowest-common-denominator interface; implementations MAY expose additional capabilities beyond the standard.

---

## 9. Future Considerations

### 9.1 Memory Hierarchies

Core memory vs. archival memory (MemGPT-style) may warrant standard representation.

### 9.2 Memory Sharing Protocol

Standard for syncing/federating memory across instances.

### 9.3 Forgetting

TTL, importance decay, and explicit retention policies.

### 9.4 Provenance

Tracking where memories came from (which conversation, which file, which agent).

---

## 10. Reference Implementation

**Nellie** (https://github.com/mmorris35/nellie-rs) serves as the reference implementation of AMP.

Features:
- Rust-based, single-binary deployment
- SQLite + embedded vector search
- MCP and HTTP transports
- Local Ollama embeddings (nomic-embed-text)

---

## Appendix A: Example Session

```bash
# Agent learns something
amp store --type lesson \
  --title "PostgreSQL connection pooling" \
  --content "Always use connection pooling in production. PgBouncer recommended." \
  --tags "postgresql,devops,performance"

# Later, agent searches for relevant knowledge
amp search "database connection issues production" --type lesson
# Returns: "PostgreSQL connection pooling" (score: 0.89)

# Agent checkpoints state
amp store --type checkpoint \
  --agent radarr \
  --working-on "Debugging auth flow" \
  --state '{"blockers": ["Missing API key for OAuth provider"]}'

# Agent resumes, retrieves checkpoint
amp list --type checkpoint --agent radarr --limit 1
```

---

## Appendix B: JSON Schema

Full JSON schemas for all record types and operations are available at:
`https://agentmemoryprotocol.org/schema/v0.1/` (placeholder)

---

## Acknowledgments

Inspired by:
- Model Context Protocol (Anthropic)
- MemGPT / Letta memory model
- Practical experience building Nellie

---

*Draft specification. Feedback welcome at [GitHub Issues](https://github.com/mmorris35/agent-memory-protocol).*
