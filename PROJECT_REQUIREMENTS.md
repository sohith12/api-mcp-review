# ReviewerAI — Project Requirements

**Version:** 0.1 (MVP specification)  
**Status:** Draft — pending approval before implementation  
**Product name:** ReviewerAI  
**Repository:** `api-mcp-review`

---

## 1. Problem Statement

Teams building and operating APIs, MCP (Model Context Protocol) servers, and LLM-backed services lack a single place to **connect**, **expose**, **load-balance**, **rate-limit**, and ** objectively review** those endpoints. Today they stitch together ngrok/Cloudflare Tunnel, Postman/Bruno, custom scripts, separate observability stacks, and ad-hoc model benchmarks—without unified scoring, historical comparison, or a gateway that enforces policy consistently at the edge.

**ReviewerAI** is a unified review, testing, and gateway platform: register arbitrary REST/GraphQL APIs, MCP servers, and AI model endpoints as **targets**, run health checks and benchmarks, score them on operational and quality dimensions, and optionally front them with a **secure tunnel**, **load balancer**, and **rate limiter** so inbound traffic is safe and measurable.

### Target Users

| Persona | Primary needs |
|--------|----------------|
| **Backend / platform engineers** | Health checks, latency SLIs, contract validation, failover across upstream instances, rate limits per environment |
| **AI / MCP developers** | Token usage, cost per call, MCP tool latency, side-by-side model comparison, tunnel for local MCP servers |
| **QA / release engineers** | Test suites against targets, regression on response shape, review scores before promotion |
| **Small teams / indie devs** | One console instead of five tools; quick tunnel + dashboard without operating Kubernetes |

**Out of scope for MVP (may come later):** billing/marketplace, multi-tenant SaaS billing, full WAF rule editor, automated remediation of failing upstreams.

---

## 2. Functional Requirements

### 2.1 Target registration and connectivity

- **FR-1.1** Users can register **targets** of types: `rest`, `graphql`, `mcp`, `llm` (OpenAI-compatible, Anthropic, or configurable base URL for local/Ollama/vLLM).
- **FR-1.2** Each target stores: display name, base URL(s), auth method (none, API key header, Bearer, mTLS config reference), optional OpenAPI/GraphQL schema URL or upload, and metadata tags (env, team).
- **FR-1.3** **MCP targets** support SSE/streaming transport configuration and stored credentials; gateway can proxy MCP HTTP/SSE paths (MVP: HTTP-based MCP; stdio MCP remains local-only with optional tunnel to a local bridge—documented limitation).
- **FR-1.4** **LLM targets** store model id, pricing hints (optional $/1M tokens for cost scoring), and default headers.
- **FR-1.5** Connection test on save: reachability, TLS validity, auth smoke request.

### 2.2 Testing, benchmarks, and validation

- **FR-2.1** **Health checks**: configurable interval, timeout, expected status/body rules; history of pass/fail.
- **FR-2.2** **Latency benchmarks**: run N requests, report min/max/mean and p50/p95/p99; optional warmup.
- **FR-2.3** **Schema / contract validation**: REST against OpenAPI (request/response where applicable); GraphQL introspection diff or persisted schema validation; MCP list-tools / ping validation.
- **FR-2.4** **Side-by-side comparison**: same input (or templated inputs) sent to 2+ targets; diff JSON/text; for LLMs, optional deterministic seed / snapshot of structure (not full semantic equality in MVP).

### 2.3 Review workflow and scoring

- **FR-3.1** **Review runs** aggregate signals over a window or on-demand: latency percentiles, error rate, uptime (from health checks), estimated **cost per call** (LLM), **token usage** (when exposed in response headers or parsed from body), and **correctness** against attached test suites (assertions on status, JSONPath, JSON Schema subset).
- **FR-3.2** **Composite score** (0–100) with configurable weights per dimension; default weights documented in product config.
- **FR-3.3** Persist review history; dashboard shows latest score and trend per target.
- **FR-3.4** Export review summary (JSON/CSV) for CI gates (MVP: API export; UI button optional).

### 2.4 Secure tunnel

- **FR-4.1** **Tunnel agents** (or embedded local CLI in Phase 2) establish outbound connection to ReviewerAI; no inbound port open on developer machine.
- **FR-4.2** Public tunnel hostname with **TLS termination** at the gateway edge.
- **FR-4.3** **Auth**: tunnel scoped tokens; optional **IP allowlist** per tunnel.
- **FR-4.4** Map tunnel routes to internal target or direct upstream URL; show tunnel status (connected / last heartbeat / bytes transferred).

**MVP assumption:** Single-region tunnel control plane co-located with gateway; one active connection per tunnel ID with reconnect semantics.

### 2.5 Load balancing

- **FR-5.1** Per logical target, configure **multiple upstream instances** (URL + weight + optional zone label).
- **FR-5.2** Strategies: **round-robin**, **least-connections**, **weighted round-robin** (MVP); sticky sessions deferred to Phase 3 unless required for GraphQL subscriptions.
- **FR-5.3** **Active health checks** on upstreams; unhealthy instances removed from pool; automatic re-admission after recovery threshold.
- **FR-5.4** Failover: if all upstreams unhealthy, return **503** with structured error (no unbounded retry storms).

### 2.6 Rate limiting

- **FR-6.1** Limits enforced at gateway on inbound traffic: **per API key**, **per client IP**, and **per target** (and optional per tunnel).
- **FR-6.2** Algorithms: **token bucket** (burst + sustained rate) and **sliding window** (configurable per rule); default token bucket for MVP simplicity with Redis backing.
- **FR-6.3** On exceed: **HTTP 429** with `Retry-After`, `X-RateLimit-*` headers; no process crash or unbounded queue growth (**graceful rejection**).
- **FR-6.4** Admin UI/API to define limit policies and attach to targets or global defaults.

### 2.7 Dashboard and API

- **FR-7.1** **Real-time metrics**: RPS, p50/p95/p99 latency, error rate, active connections, rate-limit rejections, tunnel up/down.
- **FR-7.2** **Historical trends** for the same metrics (MVP: last 7–30 days at 1m resolution; longer retention in Phase 2).
- **FR-7.3** **Targets view**: list targets, type, last health, latest review score, upstream pool status.
- **FR-7.4** **Control plane API** (REST + OpenAPI) for targets, reviews, limits, tunnels; **data plane** path proxies user traffic through gateway.
- **FR-7.5** Authentication for dashboard and control API: API keys + session login (MVP: API keys + simple email/password or SSO stub); RBAC deferred to Phase 3.

---

## 3. Non-Functional Requirements

| ID | Requirement | MVP expectation |
|----|-------------|-----------------|
| **NFR-1** | **Horizontal scalability** | Gateway nodes stateless; scale by adding instances behind LB |
| **NFR-2** | **Backpressure** | Timeouts, connection limits, 429 on rate limit; no unbounded in-memory buffering of bodies |
| **NFR-3** | **Observability** | Structured JSON logs; Prometheus metrics; trace IDs propagated (OpenTelemetry hooks in gateway) |
| **NFR-4** | **Zero-downtime deploys** | Rolling replacement of gateway instances; config via store + pub/sub invalidation |
| **NFR-5** | **Security** | TLS everywhere public; secrets in env/vault-compatible store; tunnel tokens rotatable; no secrets in logs |
| **NFR-6** | **Cost efficiency** | MVP runnable on one small VM + managed Redis; no mandatory K8s |
| **NFR-7** | **Availability** | MVP target 99.5% control plane; data plane best-effort aligned with upstreams |
| **NFR-8** | **Data durability** | Review results and config in PostgreSQL; metrics in TSDB; RPO ≤ 1h for config (backup policy documented) |

---

## 4. Traffic Budgets (Assumptions — Adjust Freely)

These numbers define **acceptance criteria** for MVP load testing and **architectural guardrails** for future scale. They are **defaults**, not hard limits of the product vision.

### 4.1 MVP / moderate-load target (must pass cleanly)

Initial build must handle the following on **recommended MVP topology** (see ARCHITECTURE.md): 2 gateway replicas, 1 Redis, 1 PostgreSQL, Prometheus + Grafana, single region.

| Metric | Default assumption | Notes |
|--------|-------------------|--------|
| **Sustained RPS** (through gateway data plane) | **500 RPS** | Mixed small JSON proxy; excludes multi-MB LLM streaming bodies from sustained average |
| **Peak burst RPS** | **2,000 RPS** for **60 s** | Token-bucket bursts absorbed; p99 gateway overhead &lt; 50 ms at p50 upstream 100 ms |
| **Concurrent connections** | **5,000** | Including idle keep-alive |
| **Active targets** | **200** registered; **50** actively receiving traffic | Config cache must not dominate memory |
| **Rate-limit checks** | **500 RPS** with **&lt; 2 ms p99** Redis latency budget per check (local/docker: relaxed to 5 ms) |
| **Review / benchmark jobs** | **10 concurrent** benchmark runs | Does not block data plane |
| **Metrics cardinality** | ≤ **50k** active time series | Label discipline enforced |

**Moderate-load validation:** k6 (or equivalent) script included in repo; pass criteria: error rate &lt; 0.1% excluding intentional 429 tests; no OOM; rate limit accuracy within 5% of configured rates.

### 4.2 Future enterprise-scale target (architecture must not preclude)

Design must allow growth toward (not necessarily day-one implement):

| Metric | Default assumption | Notes |
|--------|-------------------|--------|
| **Sustained RPS** | **100,000+ RPS** per region | Via gateway fleet + anycast/L7 LB |
| **Peak burst** | **500,000+ RPS** | Edge + regional token buckets; optional CDN for static dashboard |
| **Concurrent connections** | **1,000,000+** | Connection draining, SO_REUSEPORT / equivalent, autoscaling |
| **Regions** | **Multi-region** active-active or active-passive | Global rate-limit coordination via Redis Cluster / dedicated limiter service |
| **Metrics ingestion** | **Millions of samples/s** | Remote write sharding, Mimir/ClickHouse tier |
| **Tenants** | **10,000+** organizations | Strong tenant isolation, per-tenant cardinality limits |

No rewrite of the **core request path** (auth → rate limit → LB → proxy → metrics) should be required—only **scale-out**, **shard**, and **replace components** as documented in ARCHITECTURE.md.

---

## 5. Success Criteria for Step 2 (MVP Implementation)

After approval of this document and ARCHITECTURE.md:

1. Monorepo runs locally via `docker-compose` (Redis, metrics stack, gateway, dashboard).
2. End-to-end proxy path works for at least one REST target with API key auth and rate limiting.
3. Dashboard shows live RPS, latency percentiles, error rate, rate-limit rejections.
4. Targets view lists configured targets and placeholder or computed review scores.
5. k6 load test demonstrates sustained **500 RPS** and burst behavior with documented results.

---

## 6. Open Decisions (Defaults Chosen — Change if Needed)

| Topic | Default | Impact if changed |
|-------|---------|-------------------|
| Primary language for gateway | **Go 1.22+** | Rust if extreme perf; Node if team skill mismatch |
| Control plane DB | **PostgreSQL 16** | MySQL possible with schema port |
| Rate limit store | **Redis 7** | KeyDB/Dragonfly drop-in for some deployments |
| Metrics (MVP) | **Prometheus + Grafana** | Long-term store added in Phase 2 |
| Dashboard | **Next.js 14 (App Router) + React** | Vite SPA acceptable |
| Tunnel MVP | **Embedded WebSocket control channel in gateway** | Split tunnel service in Phase 2 for isolation |
| MCP proxy depth | **HTTP/SSE MCP only in MVP** | Full stdio bridge requires separate agent binary |
| Multi-tenancy | **Single org, multiple API keys** | Schema adds `tenant_id` everywhere in Phase 3 |

---

## 7. Glossary

- **Target:** A registered upstream system (API, MCP server, or LLM endpoint) under test and/or proxy.
- **Data plane:** High-volume proxied traffic through the gateway.
- **Control plane:** Configuration, reviews, dashboard API, tunnel registration.
- **Review run:** A scored evaluation pass over a target using configured tests and metrics windows.
