# Production Incident Analysis

---

## Investigation Overview

Correlated web.log and worker.log using request_id to identify missing orders. Analysis performed in `log_analysis.ipynb`.

### Investigation Process

- Parsed web.log and worker.log to extract structured data (timestamps, request IDs, endpoints, user IDs)
- Correlated web requests with worker completions using request_id to identify missing orders
- Analyzed timing patterns to determine when the problem started by comparing first missing vs last successful requests
- Examined endpoint-specific behavior to identify which endpoint was affected
- Investigated worker errors to find root cause indicators and correlate with missing orders
- Checked for correlations with slow database queries - found no significant spike at problem start
- Analyzed metrics-worker errors - determined they were unrelated (analytics upload timeouts to different service)
- Verified perfect correlation between worker errors and missing orders (100% overlap)

---

## 1. When did the problem start?

**`2026-07-02 14:32:40.073`**

### Evidence

- **First missing checkout:** `2026-07-02 14:32:40.073`
- **Last successful checkout:** `2026-07-02 14:32:31.237`
- **Request ID:** `16ce72300cf58a32`
- **First worker error:** `2026-07-02 14:32:42.692` (2.6s after missing request)

```log
2026-07-02 14:32:40.073 INFO [request] method=POST path=/checkout status=202 latency_ms=45 user_id=59787 request_id=16ce72300cf58a32
2026-07-02 14:32:42.692 ERROR [worker] upstream call failed request_id=16ce72300cf58a32 err=ECONNRESET upstream=10.0.3.44:8443 (retries exhausted)
```

---

## 2. Which endpoint is affected?

**`/checkout` endpoint only**

### Endpoint Breakdown

| Endpoint | Successful | Missing | Total |
|----------|-----------|---------|-------|
| `/orders`   | 7,998     | 0       | 7,998 |
| `/checkout` | 7,490     | 2,385   | 9,875 |
| **Total**   | **15,488** | **2,385** | **17,873** |

- All 2,385 missing orders are from `/checkout`
- `/orders` endpoint operated normally throughout incident

---

## 3. What do the failing requests have in common?

### Pattern

All failing requests are **POST `/checkout`** requests that fail due to upstream service connectivity issues.

### Key Characteristics

- **Total failures:** 2,385
- **Endpoint:** 100% from `/checkout`
- **Method:** POST
- **Status code:** 202 (accepted by web server)
- **Error:** `ECONNRESET` to upstream service

### Error Pattern

```log
ERROR [worker] upstream call failed request_id=<id> err=ECONNRESET upstream=10.0.3.44:8443 (retries exhausted)
```

### Common Factors

| Factor | Value |
|--------|-------|
| Upstream service | `10.0.3.44:8443` |
| Error type | `ECONNRESET` |
| Retry behavior | Exhausted |
| Start time | 14:32:40.073 |
| Duration | Through end of log period |

---

## 4. How many distinct users were affected?

**2,335 distinct users**

### User Impact

- **Total failed requests:** 2,385
- **Unique users affected:** 2,335
- **Average failures per user:** 1.02
- **Users with multiple failures:** ~50

---

## Root Cause Analysis

### Primary Cause

**Upstream service failure at `10.0.3.44:8443`**

### Supporting Evidence

1. **Perfect correlation:** 2,385 worker errors ↔ 2,385 missing orders (100% overlap)

2. **Endpoint specificity:**
   - `/checkout` → depends on `10.0.3.44:8443` → failed
   - `/orders` → does not use this service → worked normally

3. **Timing alignment:**
   - Problem start: 14:32:40.073
   - First error: 14:32:42.692 (+2.6s)
   - Consistent with worker processing latency

4. **Error consistency:**
   - All failures show identical error pattern
   - Same upstream service, same error type
   - No gradual degradation

### Unrelated Factors

- **Metrics-worker errors:** Pre-date incident, target different service (`analytics.internal:9092`)
- **Database slow queries:** No spike at problem start time, consistent before/after

### Conclusion

The upstream service at `10.0.3.44:8443` became unreachable at 14:32:40, causing all `/checkout` processing to fail. The `/orders` endpoint does not depend on this service and continued functioning normally.

---
