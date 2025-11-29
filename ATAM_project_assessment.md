# ADD — Iteration 3 (AIDAP) — ATAM Assessment & Artifacts

This document performs Iteration 3 ATAM activities for AIDAP: it contains (A) an ATAM risk assessment table, (B) descriptions of risks, non-risks, sensitivity points, and tradeoffs, and (C) an ATAM utility tree. It concludes with an evaluation of whether the ATAM deliverables are satisfied and a short checklist of evidence required to claim satisfaction.

Status: draft — update with empirical results, scripts, logs, and dashboards to finalize.

---

## Executive summary

Iteration 3 focuses on evaluating architecture (ATAM), validating major quality attributes via experiments from Iteration 2, and reducing high risks before moving to production. The three prioritized quality attributes addressed here are:
- Performance (retrieval latency and end-to-end response time)
- Security/Compliance (tenant isolation, auditability / FERPA)
- Availability / Reliability (ingestion pipeline resilience and DLQ behavior)

Below are the ATAM artifacts required by the assessment.

---

## A. ATAM Risk Assessment Table

| ID | Risk / Area | Severity (H/M/L) | Likelihood (H/M/L) | Impact | Mitigation | Detection | Recovery |
|---:|---|---:|---:|---|---|---|---|
| R1 | Ingestion reliability (MQ misconfig / DLQ overflow) | High | Medium | Lost or delayed documents, data inconsistency | DLQ + alerting, idempotent handlers, rate-limiting, per-tenant sharding | Monitor queue lag, unacked count, DLQ rate | Re-enqueue DLQ after operator review; reprocess idempotently |
| R2 | Tenant scoping failure (cross-tenant leakage) | High | Low-Medium | PII / FERPA breach, legal exposure | Validate tenant_id in gateway, row-level security (RLS) or tenant schemas; strong tests | Audit logs, anomaly detection on access events | Revoke credentials, isolate, forensic audit |
| R3 | Vector DB performance degradation with metadata filters | Medium-High | Medium | Retrieval latency spike, poor UX | Hybrid plan: pre-filter small candidate set; tune index params; partition indices | Monitor vector query latency vs filter cardinality | Fallback to relaxed filters or broaden candidate set |
| R4 | Hybrid retrieval cost growth (storage & CPU) | Medium | High (if indiscriminate) | Increased monthly costs | Adaptive hybrid (only when needed), cold storage/pruning | Monitor index size, per-query CPU/GPU cost | Switch to semantic-only or keyword-only modes for cost-sensitive tenants |
| R5 | LMS adapter failures (third-party API changes/outage) | Medium | High | Partial functionality loss (deadlines, announcements) | Circuit-breaker, retries with exponential backoff, adapter versioning | Adapter error rates, circuit open metrics | Fall back to cached data and degrade gracefully |
| R6 | JWT/claim sensitivity and token misuse | High | Low | Unauthorized access or privilege escalation | Strict signature validation, short TTL, token introspection | Token misuse detection, unusual activity alerts | Revoke tokens, rotate keys, invalidate sessions |
| R7 | Batch/worker throughput misconfiguration (over/under-utilization) | Medium | Medium | Inefficient resource usage, backlog growth | Adaptive batching, autoscaling, GPU node-pools for embeddings | Worker utilization, backlog metrics | Reconfigure batch sizes, scale workers, move heavy work to separate pool |

---

## B. Risks, Non-Risks, Sensitivity Points, and Tradeoffs

### Risks (expanded)
- Ingestion reliability (R1): The ingestion flow is sensitive to MQ configuration; incorrect retry or backoff settings can lead to message loss or major queue buildup. Mitigation: use DLQs, idempotent consumers, tenant-level throttling, and operational alerts.
- Tenant scoping failure (R2): Errors in validating tenant claims (JWT) or enforcing database access rules may expose data across tenants. Mitigation: enforce claims at the gateway, use DB row-level security or tenant-specific schemas, and run targeted security tests.
- Vector DB filter-driven latency (R3): Complex metadata filters can reduce candidate pools or push the system into costly filtering paths. Mitigation: analyze filter cardinality, apply pre-filtering before vector similarity search, and tune indexes.
- Hybrid retrieval overhead (R4): Using both keyword and vector retrieval boosts compute and storage usage. Mitigation: enable adaptive hybrid selection with cost monitoring and remove stale embeddings.
- LMS adapter brittleness (R5): External LMS APIs may change or impose rate limits. Mitigation: apply circuit breakers, retries, caching layers, and fallback behavior.

### Non-Risks (tested & validated)
- Autoscaling (HPA) responsiveness: Tests in Iteration 2 confirmed that the HPA correctly scales based on CPU and latency signals.
- Streaming LLM responses: Load testing in Iteration 2 demonstrated that streaming reliably supports ~1k concurrent users with no performance drop.
- LMS circuit-breakers: Prototype exercises validated that circuit-breakers provide smooth fallback messaging during simulated LMS outages.

### Sensitivity Points (small change → big effect)
- Vector index configuration (HNSW efSearch/efConstruction): Even minor tuning adjustments can heavily impact P95 latency and recall rates.
- MQ retry configuration (initialBackoff, backoffFactor, maxRetries): Small tweaks can meaningfully alter how quickly backlogs form or drain.
- JWT claim structure and validation: Slight formatting issues (e.g., missing tenant claim) can invert authorization results.
- Embedding batch-size limits: Adjusting batch size directly influences GPU efficiency, throughput, and response latency.

### Tradeoffs (selected)
- Performance vs Cost: Hybrid retrieval improves accuracy but increases computational and storage expenditure.
- Security vs Modifiability: Using per-tenant schemas with strict RBAC strengthens isolation but raises development and operational overhead.
- Availability vs Performance: More aggressive retry policies enhance robustness but risk message storms and short-lived performance dips.
- Simplicity vs Flexibility: A shared multi-tenant cluster is operationally simple and cost-effective, while per-tenant namespaces boost isolation at the expense of deployment and management complexity.

---

## C. ATAM Utility Tree

The utility tree below captures prioritized quality attributes and representative scenarios (leaf-level testable scenarios). The tree is prioritized by business importance and technical risk.

- Utility: Deliver correct, timely, and secure responses for university users
  - Quality Attribute: Performance (High priority)
    - Scenario P1: Retrieval latency for tenant-scoped queries has P95 < 200ms (vector query + network)
    - Scenario P2: End-to-end response (retrieval + LLM) P95 < 2s under target load (50–200 QPS)
    - Scenario P3: Under increased load (x10), system degrades gracefully (queueing/backpressure)
  - Quality Attribute: Security / Compliance (High priority)
    - Scenario S1: No cross-tenant data read allowed via API (0/1000 synthetic attempts)
    - Scenario S2: All access events produce audit entries with tenant_id, user, resource, action
    - Scenario S3: Compromised token can be revoked and auditable within 1 minute
  - Quality Attribute: Availability / Reliability (High priority)
    - Scenario A1: Ingestion pipeline processes 10k jobs with zero unhandled failures (with DLQ metric)
    - Scenario A2: When LMS is down, announcements are queued and delivered after recovery (no data loss)
    - Scenario A3: Worker autoscaling restores throughput within defined SLO after injected failure
  - Quality Attribute: Cost Efficiency (Medium)
    - Scenario C1: Hybrid retrieval mode only triggered when confidence threshold is low; cost delta < X%
  - Quality Attribute: Maintainability (Medium)
    - Scenario M1: Adapters can be replaced with < 2 hours downtime; tests cover integration surface

For each leaf scenario, attach:
- Acceptance criteria (numeric)
- Test harness / script to reproduce
- Sensitivity/impact notes

---

## D. Do these ATAM deliverables get satisfied?

Short answer: Partially — the ATAM artifacts are present in this document, but to be considered satisfied for assessment the ATAM must be backed by empirical evidence and repository artifacts. Below is the pass checklist and what remains to be produced/committed.
