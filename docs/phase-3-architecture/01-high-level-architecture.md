# High-Level Architecture — OpsNext CRM

---

## System Context

OpsNext CRM is a multi-tenant SaaS platform. External actors interact with it through well-defined boundaries.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Internet                                    │
│                                                                     │
│  ┌──────────────┐    HTTPS     ┌───────────────────────────────┐   │
│  │   Browser    │◄────────────►│     CDN (static assets)       │   │
│  │  (Next.js)   │              └───────────────────────────────┘   │
│  └──────┬───────┘                                                   │
│         │ HTTPS/REST                                                 │
│         ▼                                                           │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │               OpsNext CRM Platform                       │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                     │
│  ┌────────────┐  API Key     ┌──────────────────────────────────┐  │
│  │ Web Forms  │─────────────►│ Lead Ingest API (/api/v1/leads/  │  │
│  │ Zapier     │              │                 ingest)          │  │
│  │ Ext Tools  │              └──────────────────────────────────┘  │
│  └────────────┘                                                     │
│                                                                     │
│  ┌────────────┐  Webhook     ┌──────────────────────────────────┐  │
│  │  Slack     │◄────────────│ Outbound Webhooks                │  │
│  │  HubSpot   │              └──────────────────────────────────┘  │
│  │  Zapier    │                                                     │
│  └────────────┘                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Component Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                    OpsNext CRM Platform                                │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │                    PRESENTATION LAYER                            │ │
│  │                                                                  │ │
│  │  Next.js 14 (App Router)          Tailwind CSS                  │ │
│  │  TypeScript                       TanStack Query (data fetching) │ │
│  │  React Hook Form (forms)          Zustand (client state)        │ │
│  │                                                                  │ │
│  │  Pages: Auth, Dashboard, Contacts, Accounts, Leads, Pipeline,   │ │
│  │         Activities, Emails, Reports, Settings                   │ │
│  └─────────────────────────────┬────────────────────────────────── ┘ │
│                                │ HTTP REST (JSON)                     │
│  ┌─────────────────────────────▼──────────────────────────────────┐  │
│  │                     API GATEWAY LAYER                          │  │
│  │                                                                │  │
│  │  • Request validation (DTO pipes)                              │  │
│  │  • JWT verification (AuthGuard)                                │  │
│  │  • RBAC enforcement (PermissionsGuard)                         │  │
│  │  • Tenant isolation middleware (TenantMiddleware)              │  │
│  │  • Rate limiting (ThrottlerGuard)                              │  │
│  │  • Correlation ID injection                                    │  │
│  │  • Request/response logging                                    │  │
│  └─────────────────────────────┬──────────────────────────────── ─┘  │
│                                │                                      │
│  ┌─────────────────────────────▼──────────────────────────────────┐  │
│  │                     APPLICATION LAYER (NestJS)                  │  │
│  │                                                                 │  │
│  │  ┌──────────┐ ┌────────────┐ ┌──────────┐ ┌─────────────────┐ │  │
│  │  │   IAM    │ │  Contact & │ │   Lead   │ │    Pipeline &   │ │  │
│  │  │ Module   │ │  Account   │ │  Module  │ │  Opportunity    │ │  │
│  │  │          │ │  Module    │ │          │ │  Module         │ │  │
│  │  └──────────┘ └────────────┘ └──────────┘ └─────────────────┘ │  │
│  │                                                                 │  │
│  │  ┌──────────┐ ┌────────────┐ ┌──────────┐ ┌─────────────────┐ │  │
│  │  │ Activity │ │   Email &  │ │Analytics │ │  Tenant Mgmt    │ │  │
│  │  │  Module  │ │   Comms    │ │  Module  │ │  Module         │ │  │
│  │  │          │ │  Module    │ │          │ │                 │ │  │
│  │  └──────────┘ └────────────┘ └──────────┘ └─────────────────┘ │  │
│  │                                                                 │  │
│  │  Shared: EventEmitter2 bus │ Bull queue producer │ Audit hook  │  │
│  └─────────────────────────────┬───────────────────────────────── ┘  │
│                                │                                      │
│  ┌─────────────────────────────▼──────────────────────────────────┐  │
│  │                     INFRASTRUCTURE LAYER                        │  │
│  │                                                                 │  │
│  │  Prisma ORM ──────────────► PostgreSQL (primary + read replica) │  │
│  │  PgBouncer (connection pool)                                    │  │
│  │                                                                 │  │
│  │  Redis ── Sessions ── Rate limit counters ── Cache              │  │
│  │        └── BullMQ queues (email, export, webhooks, import)      │  │
│  │                                                                 │  │
│  │  S3-compatible storage (attachments, exports)                   │  │
│  │  SMTP / SendGrid (transactional email)                         │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Technology Stack

### Frontend

| Technology | Version | Purpose |
|-----------|---------|---------|
| Next.js | 14.x (App Router) | Full-stack React framework; SSR + client routing |
| TypeScript | 5.x | Type safety across the full stack |
| Tailwind CSS | 3.x | Utility-first styling; design system tokens |
| TanStack Query | 5.x | Server state management; caching; optimistic updates |
| React Hook Form | 7.x | Performant form state management |
| Zustand | 4.x | Lightweight client-side global state (auth, UI state) |
| Radix UI | latest | Accessible headless component primitives |
| Zod | 3.x | Runtime schema validation (shared with backend) |
| Axios | 1.x | HTTP client with interceptors |
| Playwright | 1.x | End-to-end browser testing |

**Why Next.js App Router**: Server components reduce client bundle size for data-heavy views (reports, timelines). Route handlers provide BFF patterns. Built-in image optimization and caching.

### Backend

| Technology | Version | Purpose |
|-----------|---------|---------|
| NestJS | 10.x | Modular Node.js framework; DI container; decorators |
| TypeScript | 5.x | Strict type safety |
| Prisma ORM | 5.x | Type-safe DB client; migration management |
| PostgreSQL | 16.x | Primary relational database |
| Redis | 7.x | Sessions, caching, rate limiting, BullMQ broker |
| BullMQ | 4.x | Redis-backed job queue for async processing |
| EventEmitter2 | 6.x | In-process domain event bus |
| Passport.js | 0.6.x | Authentication middleware (JWT strategy) |
| class-validator | 0.14.x | DTO validation |
| Helmet | 7.x | HTTP security headers |
| winston | 3.x | Structured JSON logging |
| Jest | 29.x | Unit and integration testing |

**Why NestJS**: Module system maps cleanly to bounded contexts. DI container enables testable service composition. Decorator-driven OpenAPI generation. Production-proven at scale.

### Infrastructure

| Technology | Purpose |
|-----------|---------|
| Docker | Container packaging for all services |
| Docker Compose | Local dev and production orchestration (v1) |
| GitHub Actions | CI/CD pipeline |
| PgBouncer | PostgreSQL connection pooling |
| Nginx | Reverse proxy; TLS termination; load balancing |
| Let's Encrypt | Automated TLS certificate management |

---

## Request Flow — Authenticated API Call

```
Browser                Next.js           NestJS API         PostgreSQL     Redis
  │                      │                   │                  │             │
  │── GET /leads ────────►│                  │                  │             │
  │                      │── fetch /api ─────►│                  │             │
  │                      │   Authorization:  │                  │             │
  │                      │   Bearer <JWT>    │                  │             │
  │                      │                  │── validateJWT ───────────────►│
  │                      │                  │◄── userId, tenantId, role ────│
  │                      │                  │                  │             │
  │                      │                  │── checkPermission │             │
  │                      │                  │   leads:read      │             │
  │                      │                  │                  │             │
  │                      │                  │── TenantMiddleware│             │
  │                      │                  │   req.tenantId    │             │
  │                      │                  │                  │             │
  │                      │                  │── SELECT * FROM   │             │
  │                      │                  │   leads WHERE     │             │
  │                      │                  │   tenantId = ?    │             │
  │                      │                  │── ───────────────►│             │
  │                      │                  │◄── rows ──────────│             │
  │                      │                  │                  │             │
  │                      │◄── 200 JSON ──────│                  │             │
  │◄── rendered page ────│                  │                  │             │
```

---

## Cross-Cutting Concerns

### Correlation IDs
Every incoming request receives a `X-Request-ID` header (generated if absent). This ID propagates through all log lines, downstream service calls, and error responses for distributed tracing.

### Tenant Context
The `TenantMiddleware` extracts `tenantId` from the validated JWT on every authenticated request and attaches it to `request.tenantId`. The Prisma tenant middleware reads this value and appends `WHERE tenantId = ?` to every query.

### Error Handling
All errors are caught by a global `AllExceptionsFilter` and returned in RFC 7807 Problem Details format:
```json
{
  "type": "https://opsnext.io/errors/not-found",
  "title": "Resource Not Found",
  "status": 404,
  "detail": "Lead OPS-abc123 not found",
  "correlationId": "req-uuid"
}
```

### Structured Logging
Every request logs: method, path, statusCode, duration, tenantId (masked), userId (masked), correlationId. No PII in logs.

### Observability
- **Metrics**: Prometheus-compatible `/metrics` endpoint (request rate, latency histograms, DB pool usage)
- **Health**: `/api/health` (liveness), `/api/health/ready` (readiness — includes DB + Redis checks)
- **Tracing**: Correlation ID on all requests; full OpenTelemetry in v2

### Async Processing
Non-critical operations run via BullMQ queues to decouple API latency from external service latency:

| Queue | Jobs |
|-------|------|
| `email` | Send transactional emails (invitations, resets, notifications) |
| `export` | CSV/JSON data exports |
| `import` | CSV lead/contact imports |
| `webhook` | Outbound webhook delivery with retry |
| `report` | Scheduled report generation and email delivery |

---

## Architecture Decisions Record (ADR)

### ADR-001: Monorepo with NestJS backend + Next.js frontend
**Decision**: Single Git repo with `apps/api` (NestJS) and `apps/web` (Next.js) and `packages/shared` (types, validators).  
**Rationale**: Atomic commits across full stack. Shared TypeScript types eliminate API contract drift. Single CI pipeline.  
**Trade-off**: Larger repo; must enforce build isolation per app.

### ADR-002: Row-level multi-tenancy via tenantId column
**Decision**: Every tenant-scoped table carries a `tenantId UUID NOT NULL` column. Prisma middleware auto-injects the filter.  
**Rationale**: Simpler than schema-per-tenant (no migration fan-out). Simpler than separate databases (no connection pool explosion). Works with standard PostgreSQL RLS.  
**Trade-off**: Requires disciplined enforcement; a bug could expose cross-tenant data. Mitigated by PostgreSQL RLS as a second layer.

### ADR-003: EventEmitter2 for in-process domain events (v1)
**Decision**: Domain events dispatched synchronously via EventEmitter2 within the NestJS process.  
**Rationale**: Zero infrastructure dependency. Simpler debuggability. Sufficient for v1 single-instance deployment.  
**Trade-off**: Events are lost if the process crashes mid-dispatch. Not suitable for distributed deployments. Migration path to BullMQ/Redis Streams is planned for v2.

### ADR-004: Prisma as the sole ORM
**Decision**: All database access via Prisma Client. No raw SQL except for aggregate queries in analytics (Prisma's `$queryRaw` with parameterized inputs only).  
**Rationale**: Type-safe queries. Auto-generated types from schema. Migration management. Soft-delete and tenant filtering via Prisma middleware.  
**Trade-off**: Prisma's query planner is less flexible than raw SQL for complex analytics. Mitigated by `$queryRaw` for read-only aggregate queries.

### ADR-005: Next.js App Router with Server Components
**Decision**: Use App Router with React Server Components for data-fetching pages; Client Components for interactive forms and real-time UI.  
**Rationale**: Server components reduce JS bundle size. Built-in streaming. Route handlers enable BFF patterns.  
**Trade-off**: App Router is newer — some ecosystem libraries still require Client Component wrappers.

### ADR-006: RS256 JWT (asymmetric key pair)
**Decision**: JWT signed with RSA private key; verified with public key.  
**Rationale**: Public key can be distributed to other services (e.g., a future worker service) without sharing the signing secret. More secure than HS256 shared secret.  
**Trade-off**: Key management is slightly more complex. Mitigated by automated key rotation on deployment.
