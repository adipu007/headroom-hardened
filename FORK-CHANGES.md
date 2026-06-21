# Fork changes — headroom-hardened

`headroom-hardened` is a fork of [chopratejas/headroom](https://github.com/chopratejas/headroom)
(`headroom-ai`), licensed under Apache-2.0. This file documents the modifications made in
this fork, as required by Apache-2.0 section 4(b).

All changes below are correctness/safety bug fixes found by auditing the core
compression, proxy, cache, and memory subsystems. No features were added or removed.

## Critical — data loss / wrong output

| File | Change |
|------|--------|
| `headroom/cache/semantic.py` | Semantic cache now scopes matches to the same conversation context (hash of messages before the final user turn). Previously matched on the final-query embedding alone, so two conversations ending in the same message (e.g. "continue") could receive each other's cached response. |
| `headroom/proxy/handlers/openai.py` | Fixed indentation defect in the Codex WebSocket memory relay: the buffer-flush block was at the `if not decided` level instead of inside the `elif response.completed` branch, and an unconditional `continue` made the memory tool-execution and pass-through phases dead code. |
| `headroom/transforms/kompress_compressor.py` | Words past the 512-token truncation limit in a chunk had no `word_id` and were silently dropped. They are now kept, preserving the "same answers" guarantee. |
| `headroom/memory/adapters/hnsw.py` | `index_batch` now sizes index capacity by the monotonic `_next_hnsw_id` (the actual max label) rather than the active entry count, preventing out-of-range label crashes after evictions. |
| `headroom/transforms/content_router.py` | A bare ```` ``` ```` code fence (no language) is no longer rewritten to ```` ```unknown ````; the fence is preserved without a bogus language tag. |

## High — error masking / malformed requests

| File | Change |
|------|--------|
| `headroom/proxy/server.py` | On the final retry of a persistent upstream 5xx, the real response is returned instead of raising a synthetic 500, so the true status/body/`retry-after` reaches the client. |
| `headroom/proxy/handlers/gemini.py`, `headroom/proxy/handlers/openai.py` | `response.json()` now also catches `JSONDecodeError`/`ValueError`, so an upstream 200 with a non-JSON body is no longer turned into a fabricated 502. |
| `headroom/proxy/handlers/openai.py` | Responses-API memory continuation is only adopted when it returns HTTP 200, and its `.json()` call is guarded — a failed continuation no longer overwrites a successful response. |
| `headroom/proxy/memory_handler.py` | When the memory backend is unavailable, an error `tool_result` is still emitted so the Anthropic continuation is not malformed (avoids HTTP 400 on the whole turn). Also added an `isinstance(block, dict)` guard in the Anthropic tool-call extractor. |
| `headroom/proxy/server.py` | `/v1/retrieve`, `/v1/telemetry/import`, and `/v1/retrieve/tool_call` now guard `await request.json()` and the object type, returning 400 instead of a 500 + stack trace on malformed bodies. |

## Medium

| File | Change |
|------|--------|
| `headroom/proxy/handlers/openai.py` | Passthrough streaming uncached-token math is now provider-aware (Anthropic `input_tokens` already excludes cache tokens; Gemini `promptTokenCount` includes them), fixing under-counting. |
| `headroom/proxy/handlers/batch.py`, `headroom/proxy/handlers/anthropic.py` | The per-item compression-failure path now increments `total_original_tokens` alongside `total_optimized_tokens`, keeping the savings denominator aligned. |
| `headroom/proxy/memory_handler.py` | Background dedup tasks are kept in a set with a done-callback so the event loop cannot garbage-collect them mid-flight. |
| `headroom/memory/traffic_learner.py` | Dedup eviction uses an insertion-ordered map so the oldest hash is evicted (a plain `set.pop()` was arbitrary), and the corresponding `_persisted_ids` entry is cleaned up to prevent a leak / stale re-sighting bump. |
| `headroom/cache/compression_store.py` | A true hash collision now preserves the existing entry instead of silently overwriting it (which could make `headroom_retrieve()` return the wrong content for an already-emitted marker). |

## Tests added

- `tests/test_semantic_cache_context_regression.py` — verifies the semantic cache does not serve a response across different conversation contexts. Runnable without native dependencies.
- `tests/test_bare_code_fence_regression.py` — verifies bare code fences are not rewritten to ```` ```unknown ````. Requires the native `headroom._core` extension (`maturin develop`).
