# ADD — Iteration 3 (AIDAP) — ATAM & Iteration 3 Deliverables

This document captures the Architecture Tradeoff Analysis Method (ATAM) work, risk analysis, prototypes, experiments, and artifacts for Iteration 3 of AIDAP. It describes drivers, refinement targets, measurable acceptance criteria, experiments to run, mitigations for identified risks, sensitivity points, and a concrete artifact list to include in the repository.

Status: draft — update with experiment results, logs, dashboards, and links to committed artifacts.

---

## 1. Iteration 3 Overview

Iteration 3 objective: evaluate and refine architecture (ATAM), mitigate the highest risks, consolidate quality attributes validated through Iteration 2 prototypes, and demonstrate measurable improvements in retrieval quality, tenant isolation, and ingestion reliability.

New / refined drivers
- Functional
  - Improve retrieval quality and latency for RAG responses
  - Add admin dashboards for ingestion & model usage
  - Harden LMS adapters for error conditions
  - Support multi-tenant deployment
- Quality Attributes (≥3)
  - Performance — reduce retrieval + generation latency under load
  - Security/Compliance — enforce tenant isolation + auditing
  - Availability — ensure ingestion pipeline resilience

---

## 2. Architectural Drivers for ATAM

Key drivers to guide ATAM analysis and experiments:
- Performance: targets and evaluation method (P50/P95/P99)
- Security/Compliance: tenant isolation, auditability (FERPA)
- Availability & Reliability: ingestion reliability, DLQ behavior
- Cost efficiency: multi-tenant consolidation vs isolation tradeoffs

Sample driver matrix (for reference)

| Driver | Priority | Measurable Goal |
|---|---:|---|
| Retrieval Performance | High | Retrieval P95 < 200ms; end-to-end P95 < 2s under target load |
| Tenant Isolation (Security) | High | 0 successful cross-tenant reads in synthetic tests |
| Ingestion Reliability | High | No unhandled failures across 10k ingestion jobs |
| Cost Efficiency | Medium | Cost per tenant <= threshold (document in cost model) |

---

## 3. Elements to Refine (Iteration 3)

Refinement targets:
- Vector Retrieval Tier: hybrid search, metadata-index optimization
- Ingestion / Embedding Workers: adaptive batching, configurable backoff
- API Gateway & Auth: tenant-aware validation and quotas
- Deployment Topology: per-tenant namespace vs shared cluster evaluation
- Monitoring / Admin Dashboards: ingestion job viewer, DLQ monitor, model usage

Design patterns and concepts to use:
- Hybrid search (semantic + keyword rerank)
- Circuit-breaker for LMS adapter failures
- Tenant-aware routing & schema partitioning (row-level security or namespaces)
- Priority / QoS queues for faculty vs automated ingestion
- Structured logging + trace propagation (OpenTelemetry)

---

## 4. Instantiation — Concrete Iteration 3 Enhancements

| Component | Iteration 3 Enhancements |
|---|---|
| Vector DB | Hybrid retrieval; metadata index optimizations; weighted scoring |
| Ingestion Workers | Adaptive batching, dynamic scheduling, advanced retry backoff |
| Core Service | Tenant-aware RAG prompt templates; tenant-scoped filtering |
| Gateway / Auth | Tenant boundaries encoded/validated in JWT claims; quota enforcement |
| Admin UI | Ingestion job viewer, failure metrics, audit log viewer |

---

## 5. Acceptance Criteria & Experiments (measurable)

Define precise, reproducible acceptance criteria and how to test them.

Performance / Latency
- Targets:
  - Vector retrieval: P50 < 100ms; P95 < 200ms; P99 < 400ms (vectorDB query + network)
  - End-to-end response (retrieval + LLM + serialization): P50 < 800ms; P95 < 2s
- Experiments:
  - Run load tests (k6 / locust) at representative QPS levels (e.g., 50, 200, 1000 concurrent users).
  - Compare retrieval approaches:
    - semantic-only
    - keyword-only (BM25)
    - hybrid (keyword filter → semantic rerank) with weighting α
  - Measure precision@K, recall@K, and latency metrics for each approach.
- Artifacts:
  - Load test scripts, raw results (.csv), graphs for P50/P95/P99, comparison table showing recall improvement (e.g., +22%).

Security / Multi-tenancy
- Targets:
  - Zero cross-tenant reads in randomized pen-tests.
  - Complete audit log entries for every data access (tenant_id, user_id, resource_id, decision).
- Experiments:
  - Synthetic negative tests attempting cross-tenant access by forging / altering JWT tenant claim.
  - RBAC tests covering faculty vs admin vs student actions.
- Artifacts:
  - security test scripts, example audit logs, unit tests (tests/security/test_tenant_isolation.py).

Availability / Ingestion Reliability
- Targets:
  - No data loss across 10k ingestion messages with configured retry policy.
  - DLQ contains only messages after maxRetries; DLQ alerts fire on growth.
- Experiments:
  - Run 10k ingestion jobs with injected transient failures; verify processing & DLQ behavior.
  - Measure throughput, queue lag, and worker utilization.
- Artifacts:
  - ingest stress scripts, MQ config snippets, DLQ contents, charts showing no unhandled failures.

Cost / Tradeoff Analysis
- Targets:
  - Provide a cost model comparing per-tenant isolation vs shared multi-tenant deployment.
- Experiments:
  - Estimate embedding storage, vectorDB cost, compute for hybrid retrieval, and per-tenant overhead.
- Artifacts:
  - Cost model (spreadsheet / markdown) and suggested mitigation (pruning, cold storage).

---

## 6. Risks, Mitigations, Detection & Recovery

List of main risks, and for each include mitigation, detection, and recovery.

1) Ingestion Reliability Risk (MQ configuration)
- Mitigation: rate-limiting producers, per-tenant sharding, DLQ + alerting, idempotent job handlers.
- Detection: monitor unacked message count, queue lag, worker errors, DLQ growth.
- Recovery: re-enqueue DLQ after review; ensure idempotency using content-hash.

2) Security Risk (tenant scoping)
- Mitigation: validate tenant_id in JWT at gateway; row-level security in DB or per-tenant schemas; automated tests.
- Detection: monitoring & alerts on cross-tenant access attempts; audit log analysis.
- Recovery: revoke compromised credentials; isolate tenant data; emergency token revocation.

3) Cost Risk (hybrid retrieval)
- Mitigation: adaptive hybrid mode; run full hybrid only when necessary; index pruning and tiered storage.
- Detection: index size and query cost alerts; monitor per-query CPU/GPU usage.
- Recovery: switch to semantic-only or keyword-only fallback; prune indices.

4) Performance Risk (metadata filtering slows top-K)
- Mitigation: precompute filter cardinalities; adapt query plan (filter-first vs vector-first) and sample-based decisions.
- Detection: track vectorDB query latency by filter cardinality; alert on threshold breaches.
- Recovery: fallback to relaxed filters or broadened candidate sets.

---

## 7. Sensitivity Points & Parameter Sweeps

Identify parameters where small changes cause big behavior changes and how to test them.

- Vector DB index configuration (HNSW params, efConstruction, efSearch)
  - Test plan: sweep index parameters and record recall vs latency; save results in a CSV matrix.
- MQ retry/backoff settings (initialBackoff, backoffFactor, maxRetries)
  - Test plan: run backlog growth experiments with injected transient failures and observe queue lag and DLQ.
- JWT claim structure and validation logic
  - Test plan: fuzz test JWT content; ensure default deny for missing/invalid tenant_id.
- Batch size thresholds for embedding workers
  - Test plan: sweep batch sizes; measure GPU/CPU utilization and throughput.

Record sensitivity matrices showing parameter value → performance/recall/throughput changes.

---

## 8. Hybrid Retrieval Pseudocode & Scoring

Suggested hybrid retrieval flow (example)

1. Keyword filter (BM25 or inverted-index) with tenant/course restriction → candidate set C (size M)
2. Compute embeddings for query (and optionally reuse candidate embeddings)
3. Compute semantic similarity for candidates in C
4. Combined scoring:
   - combinedScore = α * semanticScore + (1 - α) * normalizedBM25Score
   - Tune α in experiments (e.g., 0.6)
5. Rerank and return top-K

Pseudo scoring formula
```
combinedScore = alpha * cosine_sim(query_vec, candidate_vec) + (1 - alpha) * normalize(bm25_score)
```

Document chosen α and experimental results (precision/recall vs α).

---

## 9. Example Pseudo-configurations

Example MQ retry policy (YAML pseudo)
```yaml
mq:
  maxRetries: 5
  initialBackoffMs: 1000
  backoffFactor: 2.0
  dlqEnabled: true
  dlqTopic: ingestion-dlq
```

Example JWT tenant check (pseudocode)
```python
def validate_request(req):
    token = extract_jwt(req)
    assert verify_signature(token)
    tenant_id = token.claims.get("tenant_id")
    if not tenant_id:
        raise Unauthorized("tenant claim missing")
    if not resource_belongs_to_tenant(req.resource_id, tenant_id):
        raise Forbidden("tenant mismatch")
    return tenant_id
```

Hybrid retrieval parameters sweep (CSV columns)
- index_param_1, index_param_2, candidate_set_size, alpha, recall@10, precision@10, p95_latency_ms

---

## 10. Metrics, Alerts & Dashboards (examples)

Suggested metrics & alerting rules:
- Retrieval latency histogram (P50/P95/P99) — alert if P95 > 200ms
- End-to-end response latency (P95 > 2s) — alert
- Ingestion success rate (1h window) — alert if < 99.9%
- DLQ growth rate — alert if DLQ size increases > 10% per minute
- Cross-tenant allow events — alert on any occurrence
- Cost anomaly — alert if monthly cost increase > 20%

Dashboard panels to include:
- Retrieval latency (P50/P95/P99) across tenants
- Ingestion success rate and queue lag
- DLQ size & recent failures
- Tenant usage: queries per tenant, cost per tenant
- Security: audit log viewer for access events

---
