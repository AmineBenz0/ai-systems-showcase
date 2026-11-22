# Rezva: architecture

A multi-tenant SaaS that lets a business take, change and cancel bookings over **WhatsApp** through a conversational AI agent. The owner runs everything from web dashboards. Pre-launch; the code is private.

This page shows the system, the design patterns in it, and, most usefully, **the limitations I hit and how I fixed them**. Facts come from the project's own architecture docs, runbooks and chaos drill notes. Anything not yet built is labelled as such.

**Contents:** [System map](#1-system-map) · [One message, end to end](#2-one-message-end-to-end) · [Limitations and fixes](#3-limitations-and-how-i-fixed-them) · [Booking writes](#4-booking-writes-two-people-one-slot) · [Calendar sync](#5-calendar-sync-surviving-a-crash-mid-request) · [Patterns and tech](#6-patterns-and-technologies) · [Known limits](#7-known-limits-and-what-comes-next) · [Proof it recovers](#8-proof-it-recovers)

---

## 1. System map

Today it runs as a single-VM Docker Compose stack: right-sized for the first customers, with the scaling path written down (see [section 7](#7-known-limits-and-what-comes-next)).

```mermaid
flowchart TB
    WA["WhatsApp customer"]
    WEBU["Business owner, onboarding,<br/>platform operator"]

    subgraph EDGE["Edge"]
        CADDY["Caddy 2.8<br/>auto TLS, security headers,<br/>per-IP rate limits"]
    end

    subgraph WEB["Web apps - Next.js x5"]
        APPS["admin, client, onboarding,<br/>superadmin, landing page<br/>per-request CSP nonce"]
    end

    subgraph BE["Backend - FastAPI monolith"]
        API["REST API + WhatsApp webhook<br/>async SQLAlchemy, Pydantic v2<br/>canonical ErrorEnvelope<br/>shared package for wire formats"]
    end

    subgraph DATA["Data stores"]
        PG[("PostgreSQL 15<br/>row-level security<br/>outbox tables")]
        REDIS[("Redis 7<br/>Streams, L2 cache, idempotency keys,<br/>rate limits, locks, Pub/Sub")]
    end

    subgraph WK["Background workers - Python"]
        AIW["ai_turn_worker"]
        SEND["whatsapp_sender"]
        CSYNC["calendar_sync +<br/>reconciliation"]
    end

    subgraph AI["Agent service - FastAPI + LangGraph"]
        AGENT["Booking agent<br/>bounded tool-calling loop<br/>own database for turn logs"]
    end

    subgraph EXT["External services"]
        WAAPI["WhatsApp Business API"]
        LLM["LLM provider"]
        GC["Google Calendar"]
        EM["Resend email"]
    end

    subgraph OBS["Observability and safety nets"]
        direction LR
        O1["Prometheus + AlertManager<br/>to Slack, 7 Grafana dashboards"]
        O2["OpenTelemetry to Tempo,<br/>Sentry, JSON logs with PII redaction"]
        O3["Daily pg_dump to Backblaze B2<br/>with staleness alert"]
    end

    WA -->|webhook| CADDY
    WEBU --> APPS --> CADDY
    CADDY --> API
    API -->|reads, writes, RLS| PG
    API -->|XADD message| REDIS
    REDIS -->|XREADGROUP| AIW
    AIW -->|HTTP with circuit breaker| AGENT
    AGENT -->|tools call internal API| API
    AGENT --> LLM
    PG -->|outbox rows, SKIP LOCKED| SEND
    SEND -->|send reply| WAAPI
    WAAPI -.->|delivers to| WA
    PG -->|outbox rows| CSYNC
    CSYNC <--> GC
    API --> EM
    BE -.-> OBS
    WK -.-> OBS
```

---

## 2. One message, end to end

The webhook does the minimum and returns. All the slow, failure-prone work happens behind a queue.

```mermaid
sequenceDiagram
    autonumber
    participant C as Customer (WhatsApp)
    participant W as Webhook (FastAPI)
    participant R as Redis Streams
    participant K as AI turn worker
    participant M as Agent service (LangGraph)
    participant D as PostgreSQL
    participant S as Sender worker

    C->>W: message
    W->>W: verify HMAC-SHA256 signature
    W->>D: resolve tenant from phone_number_id
    W->>R: idempotency check on message_id
    Note over W,R: duplicate delivery from WhatsApp is dropped here
    W->>D: store message (audit trail)
    W->>R: XADD to ingress stream
    W-->>C: 200 OK straight away (target under 100 ms)

    K->>R: XREADGROUP (consumer group)
    K->>R: take per-session lock (SET NX EX)
    Note over K,R: one turn per conversation at a time,<br/>lock expires if the worker dies
    K->>M: run the turn (circuit breaker + timeout)
    M->>M: reason, then call tools in a bounded loop
    M->>D: check or change bookings (deterministic rules)
    M-->>K: reply
    K->>D: write reply to outbox (same transaction)
    K->>R: ACK

    alt agent service fails
        K->>R: NACK = ACK old entry + re-add with retry_count + 1 (backoff)
        Note over K,R: after max retries the message goes to the dead-letter queue and DLQ-depth alerts fire
    end

    S->>D: claim outbox rows (FOR UPDATE SKIP LOCKED)
    S->>C: send reply via WhatsApp API
```

---

## 3. Limitations and how I fixed them

Every row is something that was actually wrong, slow or risky, with the fix that shipped (or the plan, where it has not shipped yet).

```mermaid
flowchart LR
    subgraph REL["Reliability"]
        P1["Fire-and-forget HTTP to the agent:<br/>no retries, no queue visibility,<br/>no independent scaling"]:::problem
        F1["Redis Streams + consumer groups,<br/>retry with exponential backoff,<br/>dead-letter queue, depth and lag metrics"]:::fix
        P2["A failed turn could be lost or<br/>retried with stale stream state"]:::problem
        F2["NACK = ACK + re-add with retry_count,<br/>pending-entry retention,<br/>automatic recovery if the consumer group vanishes"]:::fix
        P3["Slow or dead dependency drags<br/>the whole request path down"]:::problem
        F3["Circuit breakers on all 6 outbound<br/>dependencies, one shared Prometheus metric"]:::fix
        P1 --> F1
        P2 --> F2
        P3 --> F3
    end

    subgraph CON["Concurrency and correctness"]
        P4["Two customers, one slot"]:::problem
        F4["Sorted advisory locks, then row locks,<br/>re-check, bounded retry,<br/>DB exclusion constraint as last line"]:::fix
        P5["Same booking submitted twice<br/>by retries"]:::problem
        F5["Idempotency at 3 layers: HTTP middleware,<br/>repository UPSERT, database constraint"]:::fix
        P6["Worker crashes while calling Google Calendar:<br/>was the event created?"]:::problem
        F6["Fenced claims with token + 5-minute lease,<br/>commit intent before the call,<br/>ambiguous crash goes to recovery_required"]:::fix
        P4 --> F4
        P5 --> F5
        P6 --> F6
    end

    subgraph PERF["Performance"]
        P7["Availability lookup hits several tables<br/>on every conversation turn"]:::problem
        F7["L1 in-memory 5 s + L2 Redis 30 s cache,<br/>single-flight, explicit invalidation on every write"]:::fix
        P8["A stuck cache lock hung requests<br/>for hours (incident, 2026-05-26)"]:::problem
        F8["Bounded lock wait (5 s), fall back to<br/>a direct fetch, metric on every timeout"]:::fix
        P7 --> F7
        P8 --> F8
    end

    subgraph OPS["Operations and safety"]
        P9["Docker health check stayed green while<br/>Postgres was down"]:::problem
        F9["Found in a chaos drill; switch orchestration<br/>to the readiness endpoint that checks dependencies"]:::fix
        P10["Cross-tenant data leak from an<br/>application bug"]:::problem
        F10["Row-level security with a per-transaction tenant id,<br/>covered by a property-based test"]:::fix
        P9 --> F9
        P10 --> F10
    end

    classDef problem fill:#fde8e8,stroke:#c0392b,color:#5a1a14
    classDef fix fill:#e6f6ea,stroke:#1e8449,color:#12391f
```

| Limitation | Why it hurt | Fix |
|---|---|---|
| Agent called through a fire-and-forget HTTP background task | No retries, no queue depth to watch, AI could not scale apart from the web process | Redis Streams ingress queue, consumer groups, backoff retries, DLQ, queue-depth and lag metrics |
| Failed turns and stream bookkeeping | A NACK could leave stale entries | NACK implemented as ACK + re-add with an incremented retry count (stream length capped); pending-entry retention; recovery when the consumer group is missing |
| Concurrent bookings for the same slot | Double booking | Advisory locks in sorted key order, then row locks, then re-read and re-check demand; bounded retries; a PostgreSQL exclusion constraint protects exclusive resources |
| Client or WhatsApp retries | Duplicate bookings | Idempotency keys enforced in HTTP middleware, in the repository UPSERT and in the database |
| Crash during a Google Calendar write | Unknown whether the event exists; blind retry could duplicate it | Outbox rows are claimed with a random token and a 5-minute lease; "mutation started" is committed before the call; a crash after that point is never replayed, it moves to `recovery_required` for an operator |
| Heavy availability reads | 30-60 ms of DB work per lookup, repeated as the agent browses dates | Two-level cache with single-flight and explicit invalidation (create, cancel, reschedule, schedule, vacation, blocked time); a bypass-rate alert catches a degraded cache |
| Cache lock deadlock (incident F8) | Requests hung for hours after a crashed task kept a lock | Lock acquire timeout with direct-fetch fallback, timeout counter, runbook with the two failure shapes (spike vs stuck) |
| Liveness vs readiness | Health check passed while the database was unreachable | Chaos drill exposed it; the fix is to probe the readiness endpoint (filed as a one-line change) |
| Cross-tenant risk | One bug could expose another business's rows | Row-level security, `SET LOCAL app.tenant_id` per transaction, property test |

---

## 4. Booking writes: two people, one slot

```mermaid
flowchart TD
    A["Booking request (create, update or cancel)"] --> B["Idempotency check<br/>HTTP middleware + repository UPSERT"]
    B -->|already processed| R1["Return the original result"]
    B --> C["Open transaction"]
    C --> D["1. Advisory locks, sorted key order<br/>staff identity, asset identity or the tenant space pool"]
    D --> E["2. Lock affected resource rows, sorted"]
    E --> F["3. Lock the booking row"]
    F --> G["4. Re-read subject, re-check overlapping demand and capacity"]
    G -->|slot taken| H["Safe 'slot unavailable' answer<br/>to the caller and the agent"]
    G -->|free| I["Write booking + outbox records<br/>in the same transaction"]
    I --> J["PostgreSQL exclusion constraint<br/>last line of defence for exclusive resources"]
    J --> K["Commit, then invalidate availability cache entries"]
    G -.->|subject changed under us| L["Roll back and retry<br/>bounded retry limit"]
    L -.-> C
```

Sorting every lock acquisition is what prevents deadlocks between two requests that touch the same staff, asset or space pool in different orders.

---

## 5. Calendar sync: surviving a crash mid-request

Google Calendar is an external system I cannot roll back, so the worker records its intent before acting and never replays an action whose outcome is unknown.

```mermaid
stateDiagram-v2
    [*] --> Queued : booking change writes an outbox row in the same transaction
    Queued --> Claimed : worker claims it with a random token and a 5-minute lease
    Claimed --> MutationStarted : commit intent with the token, then call Google
    Claimed --> Queued : crash before the call, safe to retry after the lease expires
    MutationStarted --> Done : Google confirms while token and lease are still valid
    MutationStarted --> RecoveryRequired : crash or timeout after the call began, never sent again automatically
    RecoveryRequired --> Done : operator reconciles from the recovery view
    Done --> [*]
```

---

## 6. Patterns and technologies

| Pattern | Where it is used | Technology |
|---|---|---|
| Queue-based load levelling, competing consumers | WhatsApp ingress to AI workers | Redis Streams, consumer groups |
| Dead-letter queue, retry with exponential backoff | Failed agent turns | Redis Streams, DLQ-depth alerts |
| Transactional outbox | WhatsApp replies, calendar writes | PostgreSQL, `SKIP LOCKED` claims |
| Fencing token + lease | Calendar worker claims | PostgreSQL |
| Idempotency keys (three layers) | Webhooks and booking writes | FastAPI middleware, UPSERT, DB constraints |
| Circuit breaker | All 6 outbound dependencies, agent calls | Unified Prometheus metric |
| Cache-aside with single-flight | Availability reads | In-memory L1 + Redis L2 |
| Distributed lock | One agent turn per conversation | Redis `SET NX EX` |
| Advisory locking in a fixed order | Booking writes | PostgreSQL |
| Row-level security multi-tenancy | Every tenant table | PostgreSQL 15, property test |
| Bounded agent loop with deterministic tools | Booking agent | LangGraph service, rules in code |
| Canonical error envelope, shared wire formats | API and agent service | Pydantic v2, `shared` package |
| Config invalidation fan-out | Tenant config changes | Redis Pub/Sub |

**Stack:** Python · FastAPI · async SQLAlchemy + asyncpg · Pydantic v2 · Alembic · LangGraph · PostgreSQL 15 · Redis 7 · Next.js (5 apps) · Caddy · Docker Compose · GitHub Actions (20+ workflows, including E2E, load, migration-cycle check, pip-audit, gitleaks, Semgrep, Trivy) · Prometheus · AlertManager · Grafana · OpenTelemetry + Tempo · Sentry · Backblaze B2 · Resend.

---

## 7. Known limits and what comes next

Written down on purpose: an architecture assessment (May 2026) lists what stops scaling and when it starts to matter. Items below are **planned, not built** unless stated.

| Limit today | Becomes a problem at | Planned fix |
|---|---|---|
| Single VM, vertical scaling only, no high availability | Roughly 50+ customers | Multi-node topology, read replica for dashboards and analytics |
| Cron-style workers run as a single replica; two replicas would double-fire them | First scale-out (5-10 customers) | Leader election with a PostgreSQL advisory lock per worker and tick |
| Direct connections to PostgreSQL: 4 processes x pool of up to 60 could exceed the 100-connection default | 5-10 customers | PgBouncer in transaction mode, smaller per-process pools, disable prepared-statement cache |
| No per-tenant cost tracking (LLM tokens are only global) | Pricing tiers, abuse detection | `cost_events` table fed by every LLM call, booking and WhatsApp send |
| Per-tenant rate quotas wired to only a few endpoints (noisy neighbour risk) | 5-10 customers | Tenant-scoped limits on the agent router and webhook ingress, cost-weighted |
| Secrets in `.env` files with tight permissions | First compliance request | The loader is already pluggable, so moving to a secrets manager is a config change |

---

## 8. Proof it recovers

Recovery primitives are only worth something if they fire under real failure. I ran chaos drills on the local stack (2026-05-27):

| Scenario | What happened |
|---|---|
| Postgres killed for 10 s | Readiness probe degraded correctly, webhook failed fast, backend ready again 11 s after restart, no corruption, no duplicates. Also exposed the health-check gap above. |
| Redis killed for 18 s | Streams and consumer group survived (AOF), next webhook answered in 61 ms, no duplicate booking, no lockout |
| Agent service killed for 22 s | Webhook still acknowledged in under 100 ms; worker did not crash, NACKed and retried; queue depth rose and drained after recovery |
| Worker killed mid-claim | Stale claim recovered by another pass |

All four executed scenarios passed. Network partition, full disk and the multi-replica scenarios are written but still need a staging environment.

---

*Earlier prototype: Rezva grew from a single-business barbershop calendar assistant (Streamlit, FastAPI, an LLM, MCP servers, Google Calendar). This is the multi-tenant rebuild.*
