# Agent Memory Protocol (AMP) v2.0 — Specification Draft

**Version:** 2.0.0-draft  
**Status:** Draft  
**Supersedes:** AMP v0.1.0 (backward-compatible extension)

## Summary of Changes from v0.1.0

AMP v1 defined **store and retrieve** — a flat memory model with semantic search across code chunks, lessons, and checkpoints.

AMP v2 adds **learn and evolve** — a hierarchical memory model with confidence scoring, temporal decay, knowledge graphs, autonomous learning cycles, and adaptive retrieval. v2 is a strict superset; all v1 tools and conformance levels remain valid.

---

## Core Concepts (New in v2)

### Memory Hierarchy

AMP v2 introduces three memory tiers, replacing the flat lessons model:

| Tier | Contains | Created By | Lifecycle |
|------|----------|------------|-----------|
| **World** | Universal facts, shared knowledge | Any agent (explicit or extracted) | Long-lived, high-confidence, shared |
| **Experiences** | Agent-specific events, observations, interactions | Automatic (retain cycle) | High-volume, decays over time |
| **Mental Models** | Synthesized understanding, patterns, insights | Automatic (reflect cycle) | Graduated from experiences, high-value |

Lessons (v1) map to the **World** tier. Checkpoints remain orthogonal (state snapshots, not knowledge).

### Assertions

The fundamental unit of knowledge in AMP v2 is an **assertion** — a typed, scored, temporally-tracked claim.

```json
{
  "id": "string — deterministic hash (content-addressable)",
  "tier": "world | experience | mental_model",
  "content": "string — the claim in natural language",
  "confidence": "number (0.0-1.0) — current effective confidence",
  "base_confidence": "number (0.0-1.0) — confidence at creation/last reinforcement",
  "decay_rate": "number (0.0-1.0, default 0.95) — decay per window",
  "decay_window": "number (seconds, default 604800) — decay period (1 week default)",
  "created_at": "integer — unix timestamp (when system recorded it)",
  "valid_from": "integer | null — unix timestamp (when fact became true in the world)",
  "valid_until": "integer | null — unix timestamp (when fact stopped being true)",
  "reinforced_at": "integer | null — last reinforcement timestamp",
  "reinforcement_count": "integer — times this assertion was independently confirmed",
  "entities": "string[] — extracted entity references",
  "relationships": "Relationship[] — typed edges to other assertions/entities",
  "source": "string | null — provenance (agent, interaction, reflection cycle)",
  "channel": "string | null — structured channel (working_summary, mistakes, etc.)",
  "supersedes": "string | null — ID of assertion this replaces (contradiction resolution)",
  "agent": "string | null — owning agent (null = shared/world)",
  "tags": "string[]"
}
```

**Effective confidence** is computed at query time:

```
effective(t) = base_confidence × (decay_rate ^ ((t - reinforced_at) / decay_window))
```

Assertions below a configurable threshold (default 0.1) are eligible for compaction or archival.

### Relationships (Knowledge Graph)

Assertions and entities connect via typed, weighted edges:

```json
{
  "source_id": "string",
  "target_id": "string",
  "relationship_type": "string — e.g. 'causes', 'contradicts', 'extends', 'requires', 'instance_of'",
  "weight": "number (0.0-1.0)",
  "created_at": "integer"
}
```

Standard relationship types:

| Type | Meaning |
|------|---------|
| `causes` | Source leads to or produces target |
| `contradicts` | Source conflicts with target |
| `extends` | Source builds on target |
| `requires` | Source depends on target |
| `instance_of` | Source is an example of target category |
| `related_to` | General association |
| `supersedes` | Source replaces target (newer/better) |
| `derived_from` | Source was synthesized from target(s) |

Implementations MAY define additional relationship types.

### Structured Channels

Assertions MAY be assigned to a **channel** — a named partition with distinct retrieval priority and lifecycle rules:

| Channel | Purpose | Priority | Decay |
|---------|---------|----------|-------|
| `working_summary` | Current session context | Highest (always retrieved) | Ephemeral (session-scoped) |
| `mistakes` | Error patterns, things that failed | High | Slow decay |
| `open_questions` | Unresolved items needing attention | High | No decay until resolved |
| `skills` | Learned capabilities, procedures | Medium | Slow decay |
| `preferences` | User/agent behavioral patterns | Medium | Very slow decay |
| (none) | General knowledge | Normal | Standard decay |

Implementations SHOULD retrieve high-priority channel assertions before general knowledge when assembling context.

### Learning Cycles

AMP v2 defines three autonomous cycles that implementations SHOULD support:

#### Retain (Real-time)

After every interaction, extract:
- Named entities (people, tools, concepts, files)
- Relationships between entities
- Assertions (claims, decisions, corrections)
- Channel assignments (was this a mistake? a preference? a skill?)

Input: interaction transcript  
Output: new Experience-tier assertions + graph edges

#### Reflect (Periodic)

Periodically synthesize patterns from accumulated Experiences:
- Cluster related experiences
- Identify recurring patterns → graduate to Mental Models
- Detect contradictions → lower confidence on one side, flag for resolution
- Surface knowledge gaps (expected entities/relationships that are missing)

Input: recent Experience assertions  
Output: new Mental Model assertions + contradiction flags + gap reports

#### Lint (Scheduled)

Periodic health maintenance:
- Compact: merge redundant low-confidence assertions into summaries
- Decay: archive assertions below threshold (bitemporal — never truly delete)
- Deduplicate: merge semantically-identical assertions, sum reinforcement counts
- Orphan detection: find entities with no relationships
- Consistency: flag logical contradictions in the graph

Input: full assertion store  
Output: compaction operations + health report

---

## New Required Tools (Level 3)

### Assertions

#### `memorize`

Extract and store knowledge from content. Zero-delay — returns immediately, extraction may continue asynchronously.

**Input:**
```json
{
  "content": "string (required) — text to extract knowledge from",
  "source": "string (optional) — provenance identifier",
  "agent": "string (optional) — agent performing memorization",
  "tier": "string (optional) — force tier (default: auto-classify)"
}
```

**Output:**
```json
{
  "assertions_created": "integer — number of assertions extracted",
  "entities_created": "integer — new entities added to graph",
  "relationships_created": "integer — new edges added",
  "ids": "string[] — IDs of created assertions"
}
```

#### `recall`

Adaptive multi-strategy retrieval.

**Input:**
```json
{
  "query": "string (required) — natural language query",
  "mode": "string (optional) — 'fast' | 'deep' | 'auto' (default: 'auto')",
  "limit": "integer (optional, default 10)",
  "tier": "string (optional) — filter by tier",
  "channel": "string (optional) — filter by channel",
  "agent": "string (optional) — filter by agent (null = include shared World)",
  "min_confidence": "number (optional, default 0.0) — minimum effective confidence",
  "as_of": "integer (optional) — unix timestamp for temporal query ('what did I know then?')"
}
```

**Retrieval modes:**
- `fast`: vector similarity + BM25 keyword, merged. Cheap, <100ms.
- `deep`: fast + graph traversal + temporal filtering + LLM reranking. Expensive, <2s.
- `auto`: select mode based on query complexity (implementation-defined heuristic).

**Output:**
```json
{
  "results": [
    {
      "id": "string",
      "tier": "string",
      "content": "string",
      "confidence": "number — effective confidence at query time",
      "relevance": "number — retrieval score (0-1)",
      "channel": "string | null",
      "entities": "string[]",
      "source": "string | null",
      "created_at": "integer",
      "reinforcement_count": "integer"
    }
  ],
  "mode_used": "string — which retrieval mode was applied",
  "strategies": "string[] — which strategies contributed (e.g. ['vector', 'bm25', 'graph'])"
}
```

#### `reinforce`

Strengthen an existing assertion (evidence seen again).

**Input:**
```json
{
  "id": "string (required) — assertion ID to reinforce",
  "confidence_boost": "number (optional, default: reset to base) — new base confidence",
  "source": "string (optional) — what triggered the reinforcement"
}
```

**Output:**
```json
{
  "id": "string",
  "new_confidence": "number",
  "reinforcement_count": "integer",
  "message": "string"
}
```

#### `contradict`

Flag a contradiction between two assertions.

**Input:**
```json
{
  "assertion_id": "string (required) — the assertion being challenged",
  "evidence": "string (required) — the contradicting evidence",
  "confidence": "number (optional, default 0.7) — confidence in the contradiction"
}
```

**Output:**
```json
{
  "resolution": "string — 'lowered' | 'superseded' | 'flagged_for_review'",
  "old_confidence": "number",
  "new_confidence": "number",
  "new_assertion_id": "string | null — if a new assertion was created"
}
```

#### `forget`

Archive an assertion (bitemporal — sets valid_until, does not delete).

**Input:**
```json
{
  "id": "string (required) — assertion ID to archive",
  "reason": "string (optional) — why this is being forgotten"
}
```

**Output:**
```json
{
  "id": "string",
  "archived_at": "integer",
  "message": "string"
}
```

### Knowledge Graph

#### `query_graph`

Traverse relationships in the knowledge graph.

**Input:**
```json
{
  "entity": "string (required) — entity or assertion ID to start from",
  "relationship_type": "string (optional) — filter by relationship type",
  "direction": "string (optional) — 'outgoing' | 'incoming' | 'both' (default: 'both')",
  "depth": "integer (optional, default 1) — traversal hops",
  "limit": "integer (optional, default 20)"
}
```

**Output:**
```json
{
  "nodes": [
    {
      "id": "string",
      "label": "string",
      "type": "string — 'assertion' | 'entity'",
      "confidence": "number | null"
    }
  ],
  "edges": [
    {
      "source": "string",
      "target": "string",
      "relationship_type": "string",
      "weight": "number"
    }
  ]
}
```

#### `detect_gaps`

Find missing expected knowledge.

**Input:**
```json
{
  "scope": "string (optional) — limit gap detection to a topic/entity",
  "limit": "integer (optional, default 10)"
}
```

**Output:**
```json
{
  "gaps": [
    {
      "description": "string — what knowledge is missing",
      "referenced_by": "string[] — assertion IDs that imply this should exist",
      "severity": "string — 'high' | 'medium' | 'low'"
    }
  ]
}
```

### Learning Cycles

#### `reflect`

Trigger a reflection cycle (normally runs autonomously, but can be invoked explicitly).

**Input:**
```json
{
  "scope": "string (optional) — limit reflection to recent experiences about a topic",
  "agent": "string (optional) — reflect for a specific agent"
}
```

**Output:**
```json
{
  "mental_models_created": "integer",
  "contradictions_found": "integer",
  "gaps_detected": "integer",
  "assertions_graduated": "integer — experiences promoted to mental models",
  "summary": "string — natural language summary of what was synthesized"
}
```

#### `compact`

Trigger memory compaction (merge redundant, archive decayed).

**Input:**
```json
{
  "min_age": "integer (optional, default 604800) — only compact assertions older than N seconds",
  "threshold": "number (optional, default 0.1) — archive below this effective confidence",
  "dry_run": "boolean (optional, default false) — preview without executing"
}
```

**Output:**
```json
{
  "archived": "integer — assertions archived (below threshold)",
  "merged": "integer — redundant assertions merged into summaries",
  "deduplicated": "integer — exact duplicates removed",
  "summary": "string"
}
```

### Proactive Recall

#### `predict_context`

Given an upcoming interaction or task description, predict what stored knowledge will be relevant — before the agent asks.

**Input:**
```json
{
  "context": "string (required) — upcoming task/query/interaction description",
  "limit": "integer (optional, default 5)",
  "include_decaying": "boolean (optional, default true) — surface decaying-but-relevant assertions"
}
```

**Output:**
```json
{
  "predictions": [
    {
      "id": "string",
      "content": "string",
      "confidence": "number",
      "relevance_reason": "string — why this was predicted as relevant",
      "decaying": "boolean — true if confidence is actively declining"
    }
  ]
}
```

---

## Updated Conformance Levels

| Level | Name | Requirements |
|-------|------|-------------|
| **1** | Store & Retrieve | v1 tools: search_code, lessons CRUD, checkpoints, get_status |
| **2** | Index & Watch | Level 1 + index_repo, diff_index, full_reindex, file watching |
| **3** | Learn & Evolve | Level 2 + memorize, recall, reinforce, contradict, forget, query_graph, reflect, compact |
| **4** | Anticipate | Level 3 + predict_context, detect_gaps, structured channels, adaptive retrieval modes |

Implementations MUST declare their conformance level in `get_status` output:

```json
{
  "status": "ok",
  "version": "2.0.0",
  "amp_level": 3,
  "capabilities": ["assertions", "graph", "reflect", "compact"]
}
```

---

## Backward Compatibility

AMP v2 is fully backward-compatible with v1:

- All v1 tools (`search_code`, `add_lesson`, `search_lessons`, `list_lessons`, `delete_lesson`, `add_checkpoint`, `get_recent_checkpoints`, `search_checkpoints`, `get_status`) remain valid and unchanged.
- `add_lesson` creates a World-tier assertion with default confidence 0.8 and no decay.
- `search_lessons` is equivalent to `recall` with `tier: "world"` and `mode: "fast"`.
- v1 clients work against v2 servers without modification.
- v2 clients degrade gracefully against v1 servers (check `amp_level` in status).

---

## Implementation Notes

### Model Requirements

AMP v2 requires LLM capabilities for:

| Cycle | Model Need | Can Be Local? |
|-------|-----------|---------------|
| Retain (extraction) | Entity/relationship extraction from text | Yes (small model, e.g. 7-14B) |
| Reflect (synthesis) | Pattern recognition, summarization | Yes (medium model, e.g. 30-70B) |
| Rerank (retrieval) | Relevance scoring | Yes (small model) |
| Predict (proactive) | Context prediction | Yes (medium model) |
| Contradict (resolution) | Logical reasoning | Preferably larger model |

The spec does NOT mandate which models are used. Implementations choose based on their deployment constraints. A Qwen Cloud implementation uses Qwen-Turbo/Plus/Max. A local implementation uses Ollama. The memory layer is model-agnostic.

### Storage Requirements

AMP v2 adds requirements beyond v1:

- Vector index for semantic search (unchanged)
- Full-text index for BM25 keyword search (new: required for multi-strategy)
- Graph structure for relationship traversal (new: in-memory or DB-backed)
- Temporal indexes on `created_at`, `valid_from`, `reinforced_at` (new)
- Content-addressable IDs (deterministic hashes) for merge safety

Recommended: SQLite (embedded) with FTS5 (BM25) + sqlite-vec (vectors) + in-memory graph (graphology, petgraph, or equivalent). This runs anywhere with no external dependencies.

### Multi-Agent Semantics

- **World** tier assertions are shared across all agents on the server.
- **Experience** and **Mental Model** assertions are agent-scoped by default.
- An agent MAY promote a Mental Model to World tier (sharing it).
- The `recall` tool with `agent: null` searches World + the requesting agent's tiers.
- Implementations SHOULD support memory banks (isolated namespaces) for multi-tenant deployments.

---

## Relationship to Other Protocols

| Protocol | Relationship |
|----------|-------------|
| **MCP** (Model Context Protocol) | AMP uses MCP as its primary transport. AMP defines WHAT tools exist; MCP defines HOW they're called. |
| **OpenAI Agents SDK** | AMP servers can be wired as tools in the Agents SDK. Memory is orthogonal to the agent runtime. |
| **LangChain/LlamaIndex** | AMP replaces bespoke memory modules with a standard interface. Any framework can call an AMP server. |
| **A2A** (Agent-to-Agent) | AMP is intra-agent memory. A2A is inter-agent communication. Complementary, not competing. |

---

## License

This specification is released under MIT License.
