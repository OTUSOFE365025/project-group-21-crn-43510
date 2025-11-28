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
- Ingestion reliability (R1): The ingestion pipeline depends on MQ settings; misconfigured retries/backoff can either drop messages or produce large backlogs. Mitigation: DLQ, idempotent handlers, per-tenant throttling, operator alerts.
- Tenant scoping failure (R2): Mistakes in tenant claim validation (JWT) or DB access controls could cause cross-tenant data leakage. Mitigation: gateway-level claim validation, DB RLS or per-tenant schemas, security tests.
- Vector DB filter-induced latency (R3): Heavy metadata constraints can yield low candidate sizes or force expensive filter handling. Mitigation: profile filter cardinalities, pre-filter then vector-sim on candidate set, index tuning.
- Hybrid retrieval cost (R4): Combining keyword + vector strategies increases compute/storage. Mitigation: adaptive hybrid mode and cost monitoring, prune old embeddings.
- LMS adapter fragility (R5): External LMS API changes or rate limits. Mitigation: circuit-breaker pattern, retries, cached results and fallbacks.

### Non-Risks (tested & validated)
- Autoscaling (HPA) responsiveness: Iteration 2 prototype verified HPA reacts to CPU/latency metrics.
- Streaming LLM responses: Iteration 2 load tests showed streaming can handle 1k concurrent users without degradation.
- Circuit-breakers for LMS: Prototype simulations validated graceful fallback messaging during LMS outage simulations.

### Sensitivity Points (small change → big effect)
- Vector index parameters (HNSW efSearch/efConstruction): small parameter shifts can swing P95 latency and recall.
- MQ retry settings (initialBackoff, backoffFactor, maxRetries): small changes change backlog growth behavior significantly.
- JWT claims format and validation: minimal deviations (missing tenant claim) flip authorization decisions.
- Embedding batch size thresholds: changes affect GPU throughput and latency.

### Tradeoffs (selected)
- Performance vs Cost: Hybrid retrieval increases precision but raises compute and storage cost.
- Security vs Modifiability: Per-tenant schemas and strict RBAC improve isolation but add development and ops complexity.
- Availability vs Performance: Aggressive retries increase reliability at the cost of potential message storms and temporary performance hits.
- Simplicity vs Flexibility: Shared multi-tenant cluster is cheaper and simpler operationally; per-tenant namespaces increase isolation but complicate deployment and management.

---



---

## D. Do these ATAM deliverables get satisfied?

Short answer: Partially — the ATAM artifacts are present in this document, but to be considered satisfied for assessment the ATAM must be backed by empirical evidence and repository artifacts. Below is the pass checklist and what remains to be produced/committed.
