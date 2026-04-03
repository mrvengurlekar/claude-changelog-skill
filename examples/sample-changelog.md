# Rate Limiter Rewrite to Database-Only with Reserve-Before-Rollback

**Date:** 2026-04-02
**Trigger:** User reported usage counters not updating correctly after API calls

## Claude Session Details
| Parameter | Value |
|-----------|-------|
| Model | Claude Opus 4.6 |
| Model ID | claude-opus-4-6 |
| Context Window | 1M tokens |
| Knowledge Cutoff | May 2025 |
| Platform | Windows 11 (win32) |
| Mode | default |
| Agent | primary (code written directly, Explore subagent used for config research) |
| Agent Model | same as above |

## What Changed
- rate_limiter.py: Complete rewrite from 400 lines to 140 lines
- main.py: Removed LLM call counting imports, simplified analyze endpoint
- agent.py: Removed per-call counters, contextvars, threading imports

## Why This Approach
The original rate limiter had three layers (in-memory dict, database, configurable overrides) with ~400 lines. The in-memory layer caused confusion: it reset on cold starts, diverged across replicas, and used contextvars which don't work with LangGraph's threading model.

Root cause of the bug: contextvars.ContextVar doesn't propagate writes from child threads back to the parent async context. LangGraph runs sync node functions in a thread pool — so the counter incremented a thread-local copy that was discarded when the thread ended.

## Project Dependencies
| Component | Version |
|-----------|---------|
| Python | 3.11-slim (Dockerfile) |
| fastapi | 0.111.0 |
| google-cloud-firestore | >=2.21.0,<3.0.0 |
| langgraph | 1.0.3 |
| langchain | 1.0.7 |
| uvicorn | 0.30.1 |

## Key Technical Constraints
- LangGraph 1.0.x runs sync def node functions in a thread pool via asyncio.to_thread — this is NOT documented in LangGraph docs, discovered by debugging
- Python contextvars: child threads get a COPY of the context. Writes in child threads do NOT propagate back. This is by design (Python docs) but easy to miss.
- Firestore Increment() is atomic — safe for concurrent requests without transactions
- Cloud Run default concurrency=1 per instance, but the reserve-before-rollback pattern is safe even with higher concurrency

## Code Pattern
```python
# Reserve usage BEFORE running the expensive operation:
await check_budget(user_id)          # Read-only check
await record_usage(user_id)          # Increment(1) BEFORE work runs
try:
    result = await asyncio.wait_for(agent.run(...), timeout=300.0)
except Exception:
    await rollback_usage(user_id)    # Increment(-1) on failure
    raise

# Database-level atomic increment:
async def record_usage(user_id):
    await db.collection("usage").document(user_id).set({
        daily_key: {"calls": firestore.Increment(1), "last_call": now},
        monthly_key: {"calls": firestore.Increment(1)},
    }, merge=True)

async def rollback_usage(user_id):
    await db.collection("usage").document(user_id).set({
        daily_key: {"calls": firestore.Increment(-1)},
        monthly_key: {"calls": firestore.Increment(-1)},
    }, merge=True)
```

## What Will Break If You Change This
- If you add in-memory caching back: usage will diverge across replicas and reset on cold starts
- If you move record_usage AFTER the agent: concurrent flood attack window reopens (100 requests can all pass the check before any are recorded)
- If you use contextvars again for per-request counters: reads will return 0 (same bug)
- If the subscription config doc is missing in the database: 500 error (intentional — no silent defaults)
- If you add fallback defaults: a misconfigured database will silently allow unlimited usage instead of failing

## Files Changed
- [rate_limiter.py](../../src/rate_limiter.py)
- [main.py](../../src/main.py)
- [agent.py](../../src/agent.py)

## Related
- docs/adr/001-contextvars-bug-threading.md
- docs/adr/002-database-only-rate-limiter.md
