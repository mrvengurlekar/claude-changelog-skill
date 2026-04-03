# Rate Limiter Rewrite to Firestore-Only with Reserve-Before-Rollback

**Date:** 2026-04-02
**Claude Model:** claude-opus-4-6 (1M context)
**Platform:** Windows 11 (win32)
**Trigger:** User reported usage in dev DB did not update correctly after /analyze calls

## What Changed
- rate_limiter.py: Complete rewrite from 400 lines to 140 lines
- main.py: Removed LLM call counting imports (reset_request_llm_counter, get_request_llm_count), simplified analyze endpoint
- agent.py: Removed _guard_llm_call, _GLOBAL_LLM_CALL_COUNT, _REQUEST_LLM_CALL_COUNT, contextvars, threading imports, llm_call_count from GraphState

## Why This Approach
The original rate limiter had three layers (in-memory dict, Firestore, configurable overrides) with ~400 lines. The in-memory layer caused confusion: it reset on cold starts, diverged across Cloud Run replicas, and used contextvars which don't work with LangGraph's threading model.

Root cause of the bug: contextvars.ContextVar doesn't propagate writes from child threads back to the parent async context. LangGraph runs sync node functions in a thread pool — so _guard_llm_call() incremented a thread-local copy that was discarded when the thread ended.

## Technical Environment
| Component | Version |
|-----------|---------|
| Python | 3.11-slim (Dockerfile) |
| fastapi | 0.111.0 |
| google-cloud-firestore | >=2.21.0,<3.0.0 |
| langgraph | 1.0.3 |
| langchain | 1.0.7 |
| langchain-google-vertexai | 3.0.3 |
| uvicorn | 0.30.1 |

## Key Technical Constraints
- LangGraph 1.0.x runs sync def node functions in a thread pool via asyncio.to_thread — this is NOT documented in LangGraph docs, discovered by debugging
- Python contextvars: child threads get a COPY of the context. Writes in child threads do NOT propagate back. This is by design (Python docs) but easy to miss.
- Firestore Increment() is atomic — safe for concurrent requests without transactions
- Cloud Run default concurrency=1 per instance, but the reserve-before-rollback pattern is safe even with higher concurrency

## Code Pattern
```python
# In main.py analyze_endpoint:
await check_llm_budget(user_id)        # Read-only check
await record_llm_usage(user_id)        # Increment(1) BEFORE agent runs
try:
    final_state = await asyncio.wait_for(agent.ainvoke(...), timeout=300.0)
except Exception as agent_err:
    await rollback_llm_usage(user_id)  # Increment(-1) on failure
    raise

# In rate_limiter.py:
async def record_llm_usage(user_id):
    await db.collection("usage").document(user_id).set({
        dk: {"llm_calls": firestore.Increment(1), "last_call": now},
        mk: {"llm_calls": firestore.Increment(1)},
    }, merge=True)

async def rollback_llm_usage(user_id):
    await db.collection("usage").document(user_id).set({
        dk: {"llm_calls": firestore.Increment(-1)},
        mk: {"llm_calls": firestore.Increment(-1)},
    }, merge=True)
```

## What Will Break If You Change This
- If you add in-memory caching back: usage will diverge across Cloud Run replicas and reset on cold starts
- If you move record_llm_usage AFTER agent.ainvoke: concurrent flood attack window reopens (100 requests can all pass check_llm_budget before any are recorded)
- If you use contextvars again for per-request counters: reads will return 0 (same bug)
- If subscriptionpackages collection doc is missing in Firestore: 500 error (intentional — no silent defaults)
- If you add fallback defaults: a misconfigured Firestore will silently allow unlimited usage instead of failing

## Files Changed
- [rate_limiter.py](../../DOAI_Cloud/Prototype/api_service/rate_limiter.py)
- [main.py](../../DOAI_Cloud/Prototype/api_service/main.py)
- [agent.py](../../DOAI_Cloud/Prototype/api_service/agent.py)

## Related
- docs/adr/001-contextvars-bug-langgraph-threading.md
- docs/adr/002-firestore-only-rate-limiter.md
