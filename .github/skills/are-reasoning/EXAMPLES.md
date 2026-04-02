---
name: are-examples
description: "Real-world examples of ARE reasoning: debugging, design decisions, research questions."
---

# ARE Reasoning Examples

This file contains complete, worked examples of ARE reasoning across different domains.

## Example 1: Debugging — Intermittent Payment API Timeout

### The Problem

"Payment API timeouts on 5–10% of requests. Inconsistent, not reproducible locally."

---

## PHASE 1: Generator Proposes Hypotheses

### Hypothesis 1: Downstream Service Slow

**Assumption Chain:**
- A1: Payment API calls 3rd-party payment processor
- A2: Processor is occasionally slow (network, overload)
- A3: Timeouts are exactly 30s = client timeout, not server timeout

**Reasoning:**
1. Requests time out at 30s exactly (seen in logs)
2. This is a round number, suggests client-side timeout, not real hang
3. If payment processor goes down or is slow, external API calls hang
4. 5–10% failure rate consistent with external flakiness

**Solution:**
1. Add circuit breaker to payment processor calls
2. Reduce timeout from 30s to 5s
3. Implement retry with exponential backoff
4. Add monitoring on processor latency

**Initial Confidence: 0.7**

### Hypothesis 2: Connection Pool Exhaustion

**Assumption Chain:**
- A1: API uses connection pooling (size: 10)
- A2: Connections aren't being released properly
- A3: Under load (5–10 req/sec), pool runs dry

**Reasoning:**
1. Standard connection pooling bug
2. If connections leak, pool size matters
3. 5–10 requests accumulating could hit pool limit
4. New requests wait for timeout and fail

**Solution:**
1. Increase pool size from 10 to 50
2. Add connection leak detection
3. Monitor pool usage metrics
4. Set pool timeout to fail fast

**Initial Confidence: 0.65**

### Hypothesis 3: Garbage Collection Pause in JVM

**Assumption Chain:**
- A1: Payment API runs on JVM (Java)
- A2: Full GC can pause all threads for 100–500ms
- A3: GC happens randomly, causes request hang

**Reasoning:**
1. GC pauses are invisible to most monitoring
2. If a request hits GC pause + sees network timeout, it fails
3. 5–10% failure rate = ~50–100 req GC'd per 1000 requests
4. This matches JVM GC frequency under load

**Solution:**
1. Switch to low-pause GC (ZGC or Shenandoah)
2. Tune heap size and GC thresholds
3. Add GC pause monitoring
4. Set shorter timeouts to fail fast

**Initial Confidence: 0.55**

---

## PHASE 2: Discriminator Attacks Hypotheses

### Attack on Hypothesis 1: Downstream Service Slow

**Flawed Assumptions:**
1. Assumption: "Payment processor is the bottleneck"
   - But monitoring shows processor latency is <100ms Average
   - How do we know it's not a network hop instead?

2. Assumption: "Timeout is 30s client-side"
   - Never verified; could be 30s server-side too (load balancer)

**Logical Flaws:**
1. If processor is slow, why isn't latency uniformly high?
2. 5–10% is too low for external service flakiness (usually higher)

**Contradictions:**
1. Logs show timeout at exactly 30s, but no errors from processor
2. Would expect 4xx/5xx errors if processor was slow; don't see them

**Edge Cases:**
1. What if network path to processor is flaky (5–10% packet loss)?
2. What if auth token expires mid-request?

**Robustness Score: 0.6**

**Summary:** Hypothesis explains *some* symptoms but doesn't account for consistently low processor latency or lack of processor errors. Somewhat robust but needs more evidence.

---

### Attack on Hypothesis 2: Connection Pool Exhaustion

**Flawed Assumptions:**
1. Assumption: "Pool size is 10"
   - How was this determined? Never checked config

2. Assumption: "Connections leak"
   - No evidence of unclosed connections in code review
   - Database logs show normal connection churn

**Logical Flaws:**
1. If pool exhausted, we'd expect request queue buildup; don't see it
2. Connection pool should have been tuned for 10 req/sec (should be fine)

**Contradictions:**
1. Logs show average response time is <100ms (not slow)
2. If connections were exhausted, ALL requests would slow down, not just 5–10%

**Edge Cases:**
1. What if pool timeout is very long (60s), masking the real problem?
2. What if connections are only leaked under specific load patterns?

**Robustness Score: 0.45**

**Summary:** Foundational assumption (pool exhaustion) is not supported by evidence. Very low robustness.

---

### Attack on Hypothesis 3: Garbage Collection Pause

**Flawed Assumptions:**
1. Assumption: "GC pause happens exactly when request is inflight"
   - Very narrow race condition; low probability
   - JVM has hundreds of requests concurrent; most wouldn't hit GC pause

**Logical Flaws:**
1. JVM GC should be logged; we should see GC logs at timeout times
2. But we don't see corresponding GC pause logs

**Contradictions:**
1. We use G1GC, which has <50ms pauses (not enough to cause 30s timeout)
2. Heap utilization is only 60% (not under GC pressure)

**Edge Cases:**
1. What if we're experiencing a rare full GC bug in JVM version?
2. What if there's a custom GC hook causing pauses?

**Robustness Score: 0.5**

**Summary:** Logical chain is sound but contradicted by GC logs and heap metrics. Needs deeper investigation.

---

## PHASE 3: Refiner Strengthens Survivors

### Refining Hypothesis 1: Downstream Service Slow

**Changes:**

1. **Assumption: "Processor latency is <100ms"**
   - Discriminator critique: Never verified
   - What we did: Queried processor metrics for last 24h
   - Evidence: Processor latency 50ms–200ms (found some outliers >500ms!)
   - Updated: Processor IS sometimes slow, especially at 3pm UTC

2. **Assumption: "Timeout is exactly 30s client-side"**
   - Discriminator critique: Unverified
   - What we did: Checked application code
   - Evidence: Found configuration: `timeout: 30000` (30s)
   - Updated: Confirmed 30s is from client

3. **Edge case: Network packet loss to processor**
   - Discriminator suggested this
   - Investigation: Network path tests show 0.01% packet loss (extremely low)
   - Conclusion: Not network flakiness

**Refined Solution:**
1. Monitor processor latency at 3pm UTC window
2. Implement timeout escalation: Start with 5s, increase to 10s after 1 retry
3. Add explicit circuit breaker for processor
4. Capture processor response times in logs

**Refined Confidence: 0.75 → 0.8**

**Remaining Risks:**
- What if processor is occasionally dead (not slow)?
- What if our retry logic creates cascading failures?

---

## PHASE 4: Judge Ranks & Selects

### Final Judgment

**Ranking:**

| Criterion | H1 | H2 | H3 | Weight |
|-----------|----|----|----|----|
| Alignment | 0.8 | 0.4 | 0.5 | 30% |
| Reasoning | 0.85 | 0.6 | 0.7 | 25% |
| Robustness | 0.8 | 0.45 | 0.5 | 25% |
| Actionability | 0.9 | 0.85 | 0.7 | 15% |
| **TOTAL** | **0.84** | **0.52** | **0.60** | |

**Risk Analysis:**

| Hypothesis | Risk | If Wrong |
|-----------|------|----------|
| H1: Processor slow | Low | Deploy circuit breaker; no harm if wrong |
| H2: Pool exhaustion | Medium | Increase pool; might mask real issue |
| H3: GC pause | Medium | Tune GC; performance risk if doesn't help |

---

### 🏆 Winner: Hypothesis 1 — Downstream Service Intermittently Slow

**Confidence: 0.82**

**Why:**
- Explains 80% of symptoms (timeouts, not errors, 5–10% rate)
- Processor metrics confirm occasional slowness
- Solution is low-risk, easy to implement
- Survived discrimination with evidence-based refinement

**Alternative (Backup):**
- If processor slowness persists after circuit breaker: investigate auth token expiry (0.65 confidence)

**Next Steps:**
1. **Immediate**: Deploy circuit breaker (1h effort)
2. **Validation**: Monitor processor latency post-deployment; track timeout rate decline
3. **Fallback**: If timeouts don't drop >50%, investigate H2 (pool exhaustion) with actual metrics

---

## Example 2: Design Decision — Monolith vs Microservices

### The Problem

"App is growing. Should we migrate to microservices or stay monolithic?"

---

## PHASE 1: Generator

### Hypothesis 1: Full Microservices Migration

**Assumptions:**
- Team has DevOps capacity and skills
- Independent scaling justifies complexity overhead  
- Clear service boundaries exist

**Solution:** Extract all services (auth, payments, inventory, shipping)

**Initial Confidence: 0.6** (team isn't ready yet)

### Hypothesis 2: Stay Monolithic

**Assumptions:**
- Monolith's problems are organizational (deployment latency, team scaling), not technical
- Current stack can scale to 10x load

**Solution:** Improve deployment pipeline, add feature flags, implement load testing

**Initial Confidence: 0.65** (solves problems immediately)

### Hypothesis 3: Hybrid — Extract High-Volatility Services

**Assumptions:**
- Auth service changes frequently; worth extracting
- Inventory and shipping are tightly coupled; keep together

**Solution:** Extract auth only; plan future extractions; keep core monolithic

**Initial Confidence: 0.7** (balanced risk/reward)

---

## PHASE 2: Discriminator

### Attack on H1: Full Microservices

**Major Flaw:** Team has 0 Kubernetes experience; 6-month ramp-up required

**Risk:** Could fail mid-migration, leaving system broken

**Robustness: 0.4** (high organizational risk)

### Attack on H2: Stay Monolithic

**Minor Flaw:** Doesn't scale auth independently if it becomes bottleneck

**But:** Can be revisited later if needed

**Robustness: 0.75** (safe, reversible)

### Attack on H3: Hybrid

**Major Flaw:** Adds network calls between monolith and auth service

**Risk:** Could introduce latency

**But:** Can be mitigated with caching

**Robustness: 0.8** (balanced)

---

## PHASE 3: Refiner

### Refining H1

**Critique from discriminator:** Team lacks DevOps skills  
**New finding:** Hired 1 DevOps engineer; has 2 years Kubernetes experience  
**Updated solution:** This IS now viable; add 3-month ramp-up with smaller team  
**Confidence: 0.6 → 0.65** (still risky, but less so)

### Refining H3

**Critique:** Network calls could be latency bottleneck  
**Evidence:** Even with 50ms latency per call, throughput OK (100 req/sec → 5s max impact)  
**Updated solution:** Add caching layer between monolith and auth  
**Confidence: 0.7 → 0.8** (strengthened)

---

## PHASE 4: Judge Selects

**Winner: H3 — Hybrid (Confidence: 0.8)**

**Why:**
- Balances near-term needs (stay shipping monolithic) with future flexibility (auth extracted)
- Lower risk than full migration
- Can iterate: Extract one service, learn, repeat

**Next Steps:**
1. Extract auth service (1 sprint)
2. Monitor latency impact
3. If good, extract inventory next quarter
4. Re-evaluate full microservices in 12 months with team experience

---

---

## Example 3: Research Question — API Rate Limiting Strategy

### The Problem

"Which rate limiting strategy minimizes customer frustration while preventing abuse?"

---

## PHASE 1–2: Hypotheses & Attacks (Summary)

| Strategy | Confidence | Why |
|----------|-----------|-----|
| Token Bucket (smooth, fair) | 0.82 | Industry standard; handles bursts |
| Sliding Window (strict) | 0.65 | Predictable but feels unfair |
| Leaky Bucket (gentle) | 0.70 | Fair but unpredictable for users |

---

## PHASE 3: Refiner

**Evidence gathered:**
- Customer interviews: 70% prefer predictable limits (favor token bucket)
- Abuse pattern analysis: Attacks are sudden spikes (token bucket stops these)
- Cost analysis: Sliding window uses 3x memory (disfavor)

**Updated ranking:** Token Bucket (0.87), Leaky Bucket (0.72), Sliding Window (0.60)

---

## PHASE 4: Judge Selects

**Winner: Token Bucket (0.87 confidence)**

**Recommendation:** Implement token bucket with transparent communicationto customers about rate limits

---

This completes the ARE reasoning workflow across code debugging, architecture design, and research decisions.
