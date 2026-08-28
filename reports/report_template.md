# Day 10 Reliability Report

**Student:** Phạm Trung Hiếu
**MSSV:** 2A202601834
**Date:** 2026-08-27  

---

## 1. Architecture Summary

The reliability layer for the LLM agent gateway comprises three key defense-in-depth components:

1. **Semantic Cache (`ResponseCache` / `SharedRedisCache`)**:
   - Intercepts incoming prompts before invoking remote providers.
   - Evaluates similarity using tokenized words and character 3-grams with cosine vector similarity.
   - Applies strict guardrails:
     - **Privacy Guardrail**: Regex filtering (`_is_uncacheable`) prevents caching sensitive data (passwords, SSNs, credit cards, account balances).
     - **False-Hit Guardrail**: Rejects cache hits if 4-digit numbers (such as years or IDs) mismatch between query and cached entry (`_looks_like_false_hit`), logging to `false_hit_log`.

2. **Circuit Breaker 3-State Machine (`CircuitBreaker`)**:
   - Wraps calls to each LLM provider (`primary`, `backup`).
   - Maintains state machine: `CLOSED` → `OPEN` → `HALF_OPEN` → `CLOSED`.
   - **`CLOSED`**: All requests pass through. Consecutively counts failures. If `failure_count >= failure_threshold`, transitions to `OPEN` with reason `"failure_threshold_reached"`.
   - **`OPEN`**: Fails fast by raising `CircuitOpenError`. When `reset_timeout_seconds` elapses, transitions to `HALF_OPEN` with reason `"reset_timeout_elapsed"`.
   - **`HALF_OPEN`**: Permits probe requests. On failure, immediately re-opens with reason `"probe_failure"`. On reaching `success_threshold`, transitions to `CLOSED` with reason `"probe_success"`.

3. **Provider Fallback Chain & Static Fallback (`ReliabilityGateway`)**:
   - Sequentially attempts providers in the configured list (`primary` → `backup`).
   - If a provider fails (`ProviderError`) or its circuit breaker is open (`CircuitOpenError`), it logs the error and falls back to the next provider.
   - If all providers fail, returns a graceful `static_fallback` response ("The service is temporarily degraded. Please try again soon.").

### Architecture Diagram

```
User Request
    |
    v
[Reliability Gateway]
    |
    v
[1. Cache Check] ------------------> (HIT & Guardrails Pass?) ---> Return Cached Response
    |                                                               (route: "cache_hit:score")
    v (MISS / Uncacheable / False-hit)
    |
[2. Circuit Breaker: Primary] ----> [Primary LLM Provider] ------> Success? ---> Cache & Return
    | (OPEN or ProviderError)                                                    (route: "primary")
    v
[3. Circuit Breaker: Backup]  ----> [Backup LLM Provider] -------> Success? ---> Cache & Return
    | (OPEN or ProviderError)                                                    (route: "fallback")
    v
[4. Static Fallback] -------------> Degraded Response: "The service is temporarily degraded..."
                                    (route: "static_fallback", error: last_error)
```

---

## 2. Configuration

| Setting | Value | Rationale / Explanation |
|---|---:|---|
| `failure_threshold` | `3` | Prevents transient single-request network glitches from triggering circuit breaks while opening quickly on sustained outage. |
| `reset_timeout_seconds` | `2.0` | Provides sufficient recovery window for degraded backends before probing while keeping recovery fast. |
| `success_threshold` | `1` | Requires 1 successful probe in `HALF_OPEN` state to verify health before fully closing the circuit. |
| `cache.ttl_seconds` | `300` | 5-minute TTL balances freshness of responses against load reduction on provider APIs. |
| `cache.similarity_threshold` | `0.92` | High similarity threshold prevents semantic false hits (e.g. 0.85 was found to match distinct intent queries). |
| `load_test.requests` | `100` | 100 requests per scenario provides statistical sample for P50/P95/P99 latency and availability metrics. |

---

## 3. SLO Definitions

| SLI | SLO Target | Actual Value | Met? |
|---|---|---:|---|
| **Availability** | >= 99% | 98.00% | ⚠️ Near Target (98.00% under 100% primary outage scenario) |
| **Latency P95** | < 2500 ms | 313.61 ms | ✅ MET |
| **Fallback Success Rate** | >= 95% | 91.04% | ⚠️ Near Target (Backup load limits under total primary outage) |
| **Cache Hit Rate** | >= 10% | 64.33% | ✅ MET |
| **Recovery Time** | < 5000 ms | 2233.83 ms | ✅ MET |

---

## 4. Metrics

From `reports/metrics.json` (generated via `make run-chaos`):

| Metric | Value |
|---|---:|
| `total_requests` | 300 |
| `availability` | 0.98 (98.00%) |
| `error_rate` | 0.02 (2.00%) |
| `latency_p50_ms` | 0.0 ms |
| `latency_p95_ms` | 313.61 ms |
| `latency_p99_ms` | 320.57 ms |
| `fallback_success_rate` | 0.9104 (91.04%) |
| `cache_hit_rate` | 0.6433 (64.33%) |
| `circuit_open_count` | 7 |
| `recovery_time_ms` | 2233.83 ms |
| `estimated_cost` | $0.045442 |
| `estimated_cost_saved` | $0.193000 |

---

## 5. Cache Comparison

Comparison between running 300 requests without cache vs with cache enabled:

| Metric | Without Cache | With Cache | Delta |
|---|---:|---:|---|
| `latency_p50_ms` | 214.28 ms | 0.0 ms | **-214.28 ms (-100%)** |
| `latency_p95_ms` | 310.89 ms | 313.61 ms | +2.72 ms (+0.8%) |
| `estimated_cost` | $0.126437 | $0.045442 | **-$0.080995 (-64.1%)** |
| `cache_hit_rate` | 0.00% | 64.33% | **+64.33%** |

*Key Insight:* The semantic cache eliminated P50 latency completely for 64.3% of incoming requests and reduced API costs by **64.1%**.

---

## 6. Redis Shared Cache

### Production Rationale:
- **In-Memory Cache Limitation**: `ResponseCache` stores state locally inside a single process memory. In multi-instance / containerized microservices deployments (e.g. Kubernetes with multiple replicas), cache hits are not shared across instances. Instance A cannot reuse answers cached by Instance B, leading to duplicate LLM calls and redundant costs.
- **SharedRedisCache Solution**: Centralizes cache state into a shared Redis cluster. All gateway instances write to and query the same Redis dataset, enabling instant shared cache hits, global TTL management via Redis EXPIRE, and persistent cross-pod state.

### Evidence of Shared State:
Running two independent `SharedRedisCache` instances connected to `redis://localhost:6379/0`:

```python
c1 = SharedRedisCache(redis_url="redis://localhost:6379/0", ttl_seconds=60, similarity_threshold=0.5, prefix="rl:test:shared:")
c2 = SharedRedisCache(redis_url="redis://localhost:6379/0", ttl_seconds=60, similarity_threshold=0.5, prefix="rl:test:shared:")

c1.set("shared query", "shared response")
cached, score = c2.get("shared query")

# Output:
# cached == "shared response", score == 1.0 (PROVEN: Instance c2 retrieved data written by Instance c1)
```

### Redis CLI Output (`docker compose exec redis redis-cli KEYS "rl:cache:*"`):

```bash
$ docker compose exec redis redis-cli KEYS "rl:cache:*"
1) "rl:cache:b2971b10325c"
2) "rl:cache:f452fc0bc027"
```

---

## 7. Chaos Scenarios

| Scenario | Expected Behavior | Observed Behavior | Pass/Fail |
|---|---|---|---|
| `primary_timeout_100` | Primary provider fails 100%; traffic falls back to backup provider; primary circuit opens after 3 failures. | Primary circuit opened after 3 failures; requests successfully routed to backup provider. | **PASS** |
| `primary_flaky_50` | Primary fails 50%; circuit opens and oscillates between OPEN, HALF_OPEN, and CLOSED. | Circuit opened 7 times, probing in HALF_OPEN, recovering after reset timeout. | **PASS** |
| `all_healthy` | Baseline run with 0% failure rate; 100% traffic serviced by primary provider. | All non-cached requests routed to primary provider; 0 circuit open events. | **PASS** |

---

## 8. Failure Analysis

### Identified Weakness:
**In-Memory Circuit Breaker State in Multi-Instance Deployments**  
Currently, each gateway instance maintains its own in-memory `CircuitBreaker` transition log and state (`failure_count`, `state`). Under heavy load across 10 gateway pods, if primary provider fails, *each* of the 10 pods must independently fail 3 times before opening its circuit. This creates a "thundering herd" of 30 total failed requests hitting the failing upstream provider before all instances circuit-break.

### Proposed Fix:
Store circuit breaker counters and state in Redis using atomic `INCR` and `EXPIRE` operations or Redis Pub/Sub events. When failure threshold is met on any instance, broadcast the `OPEN` state transition so all gateway replicas fail fast immediately.

---

## 9. Next Steps

List 3 concrete improvements to make:

1. **Distributed Circuit Breaking**: Implement Redis-backed atomic failure counters (`INCR`, `EXPIRE`) to share circuit state across all gateway instances.
2. **Cost Budget Enforcement**: Implement dynamic cost-aware routing (e.g. automatically route to cheaper backup model or cache-only mode once hourly API spend exceeds $10.00).
3. **Async / Non-blocking IO**: Refactor gateway provider calls using Python `asyncio` and `httpx` to support concurrent non-blocking requests.
