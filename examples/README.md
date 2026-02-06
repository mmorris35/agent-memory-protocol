# AMP Examples

Example request and response payloads for the Agent Memory Protocol.

## Files

| File | Description |
|------|-------------|
| `store-lesson.json` | Store a lesson record |
| `store-checkpoint.json` | Store an agent checkpoint |
| `search-request.json` | Semantic search query |
| `search-response.json` | Example search results |
| `status-response.json` | Server status response |

## Usage

These examples work with any AMP-compliant server:

```bash
# Store a lesson
curl -X POST http://localhost:8765/amp/store \
  -H "Content-Type: application/json" \
  -d @store-lesson.json

# Search
curl -X POST http://localhost:8765/amp/search \
  -H "Content-Type: application/json" \
  -d @search-request.json
```
