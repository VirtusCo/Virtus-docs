# DevLog — 09 Mar 2026 — SSE Streaming & RAG Pipeline

**Engineer:** Antony Austin
**Branch:** `prototype`
**Focus:** Real-time token streaming + knowledge base RAG retrieval

---

## Session Summary

Two major features added to the AI subsystem:

1. **SSE Token Streaming** — Real-time Server-Sent Events pipeline from model inference through HTTP server to Flutter GUI. Tokens now appear word-by-word as the model generates them (~39 tok/s).

2. **RAG Knowledge Base** — TF-IDF retrieval-augmented generation pipeline. 41 airport knowledge documents injected as context before inference, grounding the model's factual answers.

---

## 1. SSE Token Streaming

### Problem
AI server (`ai_server.py`) returned the full response only after inference completed. For 1–2s responses, users saw no feedback.

### Implementation

| Component | File | Change |
|-----------|------|--------|
| Inference Engine | `inference_engine.py` | Added `query_stream()` generator using `create_chat_completion(stream=True)` |
| Orchestrator | `orchestrator.py` | Added `process_query_stream()` yielding `{event, ...}` dicts |
| HTTP Server | `ai_server.py` | Added `POST /api/chat/stream` SSE endpoint, switched to `ThreadingHTTPServer` |
| Flutter Service | `ai_service.dart` | Added `chatStream()` async generator + `StreamEvent` class |
| Flutter Provider | `providers.dart` | SSE consumption, token accumulation, `_sseSubscription` lifecycle |

### SSE Event Protocol
```
event: adapter\ndata: {"adapter": "conversational"}\n\n
event: token\ndata: {"token": "Hello"}\n\n
event: token\ndata: {"token": " there"}\n\n
event: tool_call\ndata: {"tool_call": {...}}\n\n
event: tool_result\ndata: {"tool_name": "...", "data": {...}}\n\n
event: done\ndata: {"latency_ms": 1234, "tool_calls": [...]}\n\n
```

### Bug Fix: ThreadingHTTPServer
Single-threaded `HTTPServer` blocked when a client held an SSE connection open — health checks from other clients hung. Fixed by switching to `ThreadingHTTPServer`.

---

## 2. RAG Knowledge Base Retrieval

### Problem
Fine-tuned model answers conversational and tool-use queries well, but confabulates factual airport details (specific gate numbers, terminal layouts, transport options).

### Architecture Decision: TF-IDF vs Embeddings
- **sentence-transformers**: ~80 MB model + 200ms/query on RPi → too heavy
- **TF-IDF + keyword boost**: <1ms/query, 0 dependencies, pure Python → chosen

### Knowledge Base (5 JSON files, 41 documents)

| File | Category | Documents |
|------|----------|-----------|
| `airport_facilities.json` | facilities | 12 (restrooms, WiFi, medical, charging, etc.) |
| `airport_terminals.json` | terminals | 8 (T1-T3 overviews, gates, levels) |
| `airport_transport.json` | transport | 5 (taxi, metro, bus, car rental, parking) |
| `airport_dining_shopping.json` | dining | 6 (food court, coffee, vegetarian, duty-free) |
| `airport_services.json` | services | 10 (check-in, security, immigration, baggage, etc.) |

### Retriever (`rag_retriever.py`, ~440 lines)
- **Indexing**: Augmented TF × IDF, L2-normalised
- **Retrieval**: Cosine similarity + 0.15 keyword boost per match
- **Defaults**: top_k=3, min_score=0.05, max_context_chars=1200
- **Integration**: Context prepended before session history in `_build_context_string()`

### Integration Points

1. **`orchestrator.py`**: `__init__()` accepts optional `retriever` param. `_build_context_string()` calls `retriever.build_context(user_query)`.
2. **`ai_server.py`**: `load_engine()` creates `KnowledgeBaseRetriever()` and passes to orchestrator.

### Tests
- **30 unit tests** in `test/test_rag.py` — all passing
- Coverage: loading, tokenization, indexing, retrieval scoring, keyword boosting, category filtering, context building, edge cases, real KB integration tests

---

## Files Changed

| Action | File |
|--------|------|
| NEW | `porter_ai_assistant/rag_retriever.py` |
| NEW | `data/knowledge_base/airport_facilities.json` |
| NEW | `data/knowledge_base/airport_terminals.json` |
| NEW | `data/knowledge_base/airport_transport.json` |
| NEW | `data/knowledge_base/airport_dining_shopping.json` |
| NEW | `data/knowledge_base/airport_services.json` |
| NEW | `test/test_rag.py` |
| MOD | `porter_ai_assistant/inference_engine.py` (query_stream) |
| MOD | `porter_ai_assistant/orchestrator.py` (process_query_stream, RAG) |
| MOD | `scripts/ai_server.py` (SSE endpoint, ThreadingHTTPServer, RAG init) |
| MOD | `porter_gui/lib/services/ai_service.dart` (chatStream) |
| MOD | `porter_gui/lib/providers/providers.dart` (SSE consumption) |
| MOD | `CHANGES.md` (entries #39, #40) |

---

## Test Results

```
$ python3 -m pytest test/test_rag.py -v
30 passed in 0.05s
```

All existing tests continue to pass.
