# ADD — Iteration 2 (AIDAP)

This single-file Architectural Design Document (ADD) contains all Iteration 2 artifacts for AIDAP (AI‑Powered Digital Assistant Platform) excluding diagrams (diagrams already exist in the repo). Iteration 2 extends and hardens Iteration 1 with ingestion, embedding pipeline, observability, security/compliance, and deployment automation.

Note: Visual diagrams for iteration 2 should be stored in Iteration 2 Diagrams/ as image files.

---

Table of contents
- Purpose & Goals
- Iteration 2 scope and deliverables
- Logical Architecture (component table)
- Sequence Methods (no diagrams included here)
- Deployment architecture (table)
- Quality attributes (iteration 2)
- Design decisions (iteration 2)
- Analysis & Progress

---

## Purpose & Goals

Iteration 2 goal: evolve the AIDAP proof-of-concept into a hardened, demonstrable platform with:
- Real LMS and calendar integrations (replace mocks with adapters hitting sandbox/test APIs)
- A production-ready retrieval pipeline (chunking, embedding, vector DB sync)
- Event-driven processing for async tasks (announcement scheduling, email notifications)
- Observability (metrics, logs, tracing) and CI to demonstrate quality attributes
- Security & compliance controls (FERPA awareness, encryption, RBAC)
- Deployment automation (k8s manifests, basic Helm chart/CI pipeline) and canary/blue-green strategy

Deliverables (Iteration 2)
- Updated logical architecture component table
- Sequence method tables for iteration-2 use cases
- Vector ingestion & sync pipeline description + sample script
- CI workflow to run tests and export artifacts
- Monitoring and tracing plan + sample dashboards/alert ideas
- Design decisions table updated with iteration-2 choices and justifications
- Analysis and progress table (evidence: links to code, tests, images, logs)

---

## Logical Architecture (component table, iteration 2)

| Component | Description | Responsibilities |
|---|---|---|
| API Gateway (Ingress + Kong/Traefik) | Single external entry point with auth, rate-limiting, WAF integration and observability | TLS termination, JWT validation, ingress policies, auth passthrough, metrics |
| Auth Service (OIDC/SSO) | Use institution SSO (SAML/OIDC) for authentication and provisioning | Token issuance/introspection, role mapping, session claims, integration with Vault |
| Edge Services (rate-limit, WAF) | Managed or cloud WAF in front of API | Protect quotas, block malicious traffic |
| Core Assistant Service | Stateless microservice orchestrating conversations | handleMessage(session, message), privacy filters, retrieval, LLM calls, RBAC enforcement |
| Session Service (Redis) | Stores short-term conversational state | Session context windows, TTL-based storage |
| Ingestion Service | Content fetching, parsing, chunking, metadata extraction | Enqueue embedding tasks, dedup detection |
| Embedding Worker | Batch worker for embedding creation | Consume queue, call embedding model, upsert to Vector DB |
| Vector DB | Stores embeddings with metadata filters | Similarity search with metadata filtering (courseId, docType) |
| LLM Service | Responsible for generation (OpenAI or self-hosted) | Streaming responses, RAG prompts |
| Message Queue | Event bus for async tasks | Ingest jobs, email notifications, retries, DLQ |
| Postgres | Authoritative store for users, enrollments, audit logs | ACID storage, relational queries |
| Mail Service | SendGrid/SES integration for announcements | Template rendering, retries, delivery receipts |
| Observability | Prometheus/Grafana/Jaeger | Metrics, dashboards, traces |
| Secrets Manager | Vault / Cloud secrets | Store API keys, DB credentials, encryption keys |
| Analytics Service | Usage aggregation and trend analysis | Enrollment trends, usage metrics |

Rationale (iteration 2):
- Move ingestion & embedding off the request path to avoid blocking user requests.
- Introduce message queue for decoupling and retries.
- Formalize observability and secrets management for production readiness.
- Keep Core stateless for autoscaling.

---

## Sequence Methods (diagrams are stored separately)

### Chosen iteration-2 use cases
1. Follow-up context: "What about the one after that?" (conversational context retention & reference)
2. Bulk scheduled announcement: Faculty schedules announcement to LMS + email
3. Ingestion & sync: Sync LMS course materials into Vector DB

### Sequence 1: Follow-up question context — Methods table

| Component | Methods / Endpoints |
|---|---|
| API | POST /api/v2/converse(session, message) |
| Core | Core.handleMessage(sessionId, message), Core.handleFollowUp(sessionId, message) |
| Session | Session.getSessionContext(sessionId), Session.appendToContext(sessionId, message) |
| VectorDB | VectorDB.retrieveContextualDocs(filters, topK) |
| AI | AI.generateResponse(prompt, context, streaming=True) |

Refer to ADD/iteration2/sequence_followup_context.png for the diagram.

---

### Sequence 2: Bulk scheduled announcement — Methods table

| Component | Methods / Endpoints |
|---|---|
| Core | scheduleAnnouncement(courseId, payload, schedule), updateAnnouncementStatus(id, status) |
| MQ | publish(topic, message) |
| LMS Adapter | createAnnouncement(courseId, payload) |
| Mail | enqueueCourseEmail(courseId, payload) |

Refer to ADD/iteration2/sequence_bulk_announcement.png for the diagram.

---

### Sequence 3: Ingestion & Sync pipeline — Methods table

| Component | Methods / Endpoints |
|---|---|
| Ingestion | listCourseResources(courseId), fetchAndParse(resourceUrl) -> chunks[] |
| MQ | enqueueEmbeddingJob(chunks, metadata) |
| Embedding | embedText(text), vdb.upsert(vector, metadata) |

Refer to ADD/iteration2/sequence_ingest_sync.png for the diagram.

Notes:
- Deduplication by content hash and TTL-based re-index policy.

---

## Deployment Architecture (iteration 2) — Deployment table

| Component | Environment / Notes |
|---|---|
| Cluster | Kubernetes (EKS/GKE/AKS) with node pools for CPU vs GPU |
| Embedding Workers | GPU node pool when self-hosting models; otherwise CPU with managed APIs |
| Vector DB | Managed (Pinecone/Milvus) or self-hosted Milvus based on cost/ops |
| Postgres | Managed cloud DB with read replicas |
| Redis | Clustered Redis for session/ephemeral context |
| Message Queue | Managed Kafka/RabbitMQ for durability |
| Observability | Prometheus (metrics), Grafana (dashboards), Jaeger (traces) |
| Secrets Manager | Vault or cloud secret store integrated with k8s |

Security & compliance notes:
- TLS for all traffic, mTLS between services when possible.
- Encrypted storage at rest and RBAC enforcement.
- Audit logging for data access (FERPA considerations).

Refer to ADD/iteration2/deployment_iter2.png for the deployment diagram.

---

## Quality Attributes — Iteration 2

| Attribute | Controls / Implementation | Evidence / Tests |
|---|---|---|
| Security & Compliance (FERPA) | SSO, RBAC, audit logs, data minimization, encryption | tests/test_rbac.py, docs/audit_sample.log |
| Scalability | HPA for Core, separate worker pools, autoscaling rules | k8s/hpa.yaml, scripts/load_test.sh |
| Availability | Retries, circuit-breaker, DLQs for MQ | tests/test_retry_and_circuit_breaker.py, logs/retry_demo.log |
| Observability | OTel traces, Prometheus metrics, Grafana dashboards | prometheus/metrics.yaml, grafana/dashboard.json |
| Performance | Caching in Redis, streaming responses | scripts/measure_latency.sh, artifacts/latency_report.md |
| Maintainability | Adapter pattern, test harness | tests/ folder, adapter READMEs |

Acceptance for "Exceeds":
- Add tests and sample logs/screenshots to demonstrate these attributes and commit CI to run them.

---

## Design Decisions (Iteration 2)

| ID | Decision | Alternatives | Trade-offs / Rationale | Status |
|---:|---|---|---|---|
| DD2-1 | Microservice decomposition (Core + Ingest + Embedding) | Monolith | Microservices add ops complexity but enable separate scaling and resilience | Chosen |
| DD2-2 | Managed Vector DB (Pinecone) | Self-hosted Milvus | Managed reduces ops; self-hosting reduces cost later but increases ops | Chosen (iter2) |
| DD2-3 | Message queue (Kafka) for ingest & scheduling | Direct sync calls | MQ provides durability, retries, decoupling | Chosen |
| DD2-4 | Streaming responses (SSE/WebSockets) | Polling | Streaming reduces perceived latency but adds UI complexity | Chosen |
| DD2-5 | OpenTelemetry for tracing | Vendor APM | OTel offers vendor neutrality; vendor APM offers richer out-of-box features | Chosen |

---

## Analysis & Progress (Iteration 1 → Iteration 2)

### Progress table

| Item | Iteration 1 | Iteration 2 plan/status | Proof / Link |
|---|---|---|---|
| Logical Architecture | Iter1 components | Extended with ingestion, MQ, Vault | ADD/iteration2/ (this file) |
| Sequences | Iter1 sequences | 3 new sequences for RAG + ingestion + scheduling | ADD/iteration2/sequence_*.png |
| Deployment | Dev deployment | Staging/prod + HPA + canary planned | k8s/ manifests to add |
| Quality Attributes | Basic security + latency | Extended to FERPA, tracing, scaling | tests/, prometheus/, grafana/ |
| Design Decisions | Iter1 DDs | Iter2 DDs added above | This file |
| Implementation | API skeleton | Ingest pipeline, MQ, embedding workers — in progress | src/ingest, src/embedding (create) |
| Tests & CI | Suggested tests | CI workflow to run tests/builds — to add | .github/workflows/ci.yml |

Short analysis:
- Iteration 2 emphasizes reliability and demonstrability: ingestion pipeline proofs, monitoring, and secure integrations to maximize rubric points.

---
