# Non-Functional Requirements — OpsNext CRM

NFRs define how well the system performs its functions — quality attributes that are non-negotiable constraints on the design.

---

## NFR-PERF: Performance

### Response Time Targets

| Operation Type | p50 | p95 | p99 | Measurement |
|----------------|-----|-----|-----|-------------|
| Read — single resource | < 80ms | < 200ms | < 400ms | k6 load test |
| Read — list (≤ 100 records, paginated) | < 120ms | < 300ms | < 600ms | k6 load test |
| Read — list (100–1,000 records) | < 250ms | < 600ms | < 1,200ms | k6 load test |
| Write — create/update single entity | < 150ms | < 400ms | < 800ms | k6 load test |
| Full-text search (contacts, leads) | < 100ms | < 250ms | < 500ms | k6 load test |
| Dashboard — pre-aggregated data | < 500ms | < 1,500ms | < 2,500ms | k6 load test |
| CSV import — 1,000 rows | < 10s (async) | < 30s | < 60s | manual test |
| File upload — 25MB | < 15s | < 30s | < 45s | manual test (10Mbps) |

### Frontend Performance

| Metric | Target | Tool |
|--------|--------|------|
| Time to First Byte (TTFB) | < 200ms | Lighthouse |
| First Contentful Paint (FCP) | < 1,500ms | Lighthouse |
| Largest Contentful Paint (LCP) | < 2,500ms | Lighthouse |
| Time to Interactive (TTI) | < 3,000ms | Lighthouse |
| Cumulative Layout Shift (CLS) | < 0.1 | Lighthouse |
| Lighthouse Performance Score | ≥ 85 | Lighthouse CI |

### Throughput & Concurrency

| Scenario | Target |
|----------|--------|
| Concurrent users per tenant | 500 without degradation |
| Total concurrent tenants | 500 active tenants |
| Max API requests/second (platform-wide) | 10,000 rps (horizontal scaling) |
| Database connections (pooled) | 100 max per DB instance (PgBouncer) |

### Caching Strategy

| Layer | What is cached | TTL | Store |
|-------|---------------|-----|-------|
| Permission lookups | User's resolved permission set | 5 minutes | Redis |
| Pipeline stage config | Tenant's pipeline + stage list | 5 minutes | Redis |
| Dashboard aggregates | Pre-computed KPI values | 5 minutes | Redis |
| Static assets (JS/CSS) | All frontend bundles | 1 year (content hash) | CDN |
| API responses | Read-heavy, low-change data (roles, plans) | 10 minutes | Redis |

**Cache invalidation**: Explicit invalidation on write. No stale-while-revalidate for financial/CRM data.

---

## NFR-SEC: Security

### Authentication & Session

| Requirement | Specification |
|-------------|--------------|
| JWT algorithm | RS256 (asymmetric key pair) |
| Access token expiry | 15 minutes |
| Refresh token expiry | 30 days (single-use, rotated) |
| Refresh token storage | HttpOnly, Secure, SameSite=Strict cookie |
| Access token storage | In-memory (React state) — never localStorage |
| Password hashing | bcrypt, work factor ≥ 12 |
| Key rotation | JWT keypair rotated on deployment without service interruption |

### Transport & Encryption

| Requirement | Specification |
|-------------|--------------|
| TLS version | TLS 1.2 minimum; TLS 1.3 preferred |
| Certificate | Valid CA-signed certificate; auto-renewal via Let's Encrypt |
| HSTS | Strict-Transport-Security: max-age=31536000; includeSubDomains |
| Database encryption at rest | AES-256 (PostgreSQL transparent data encryption) |
| S3 bucket encryption | AES-256 SSE |
| Backup encryption | AES-256 |

### HTTP Security Headers

```
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

### Input Validation

| Layer | Mechanism |
|-------|-----------|
| API request bodies | NestJS class-validator + class-transformer DTOs |
| SQL injection | Parameterized queries via Prisma ORM — no raw SQL with user input |
| XSS in API responses | JSON responses only; no HTML in API layer |
| XSS in frontend | React JSX escaping (default); DOMPurify for any markdown/rich text rendering |
| File upload MIME | Server-side MIME type validation (magic bytes, not just Content-Type header) |
| Email validation | RFC 5322 regex + DNS MX check on critical paths |

### Rate Limiting

| Endpoint Group | Limit | Window | Response |
|----------------|-------|--------|---------|
| Auth endpoints (per IP) | 100 requests | 15 minutes | 429 + `Retry-After` |
| Login failures (per IP) | 5 failures | 15 minutes | 429 + backoff |
| API (authenticated user) | 1,000 requests | 1 minute | 429 + `X-RateLimit-*` headers |
| Lead ingest (per API key) | 100 requests | 1 minute | 429 |
| File upload (per user) | 10 files | 1 minute | 429 |

### CORS

```
Allowed origins: explicit allowlist (no wildcards in production)
Allowed methods: GET, POST, PATCH, DELETE, OPTIONS
Allowed headers: Authorization, Content-Type, X-Tenant-ID, X-Request-ID
Credentials: true (for cookie-based refresh token)
```

### OWASP Top 10 Mitigation Matrix

| # | Threat | Control |
|---|--------|---------|
| A01 | Broken Access Control | RBAC guard on every protected route; tenantId Prisma middleware |
| A02 | Cryptographic Failures | TLS 1.3; AES-256 at rest; RS256 JWT; bcrypt passwords |
| A03 | Injection | Prisma parameterized queries; class-validator on all inputs |
| A04 | Insecure Design | Multi-tenant isolation architectural; least-privilege defaults |
| A05 | Security Misconfiguration | Helmet.js; CORS allowlist; CSP; CI security header check |
| A06 | Vulnerable Components | npm audit in CI; Dependabot alerts; block on critical CVEs |
| A07 | Auth Failures | Token rotation; rate limiting; account lockout |
| A08 | Integrity Failures | SHA-256 webhook signatures; no eval/Function() |
| A09 | Logging & Monitoring | Structured logging; audit trail; Sentry error tracking |
| A10 | SSRF | No user-controlled URL fetching in v1 (webhook URLs allowlisted) |

---

## NFR-SCALE: Scalability

### Horizontal Scaling Requirements

| Component | Scaling Model | State |
|-----------|--------------|-------|
| NestJS API | Stateless horizontal scaling | Stateless (all state in Redis/DB) |
| Next.js Frontend | CDN-deployed static + SSR pods | Stateless |
| Background Workers | Queue-based, horizontally scalable | Job state in BullMQ/Redis |
| PostgreSQL | Primary + read replicas | Replicated |
| Redis | Single instance v1; Cluster v2 | Shared state |

### Data Volume Targets (per tenant, within NFR-PERF)

| Entity | Volume Ceiling |
|--------|---------------|
| Contacts | 500,000 |
| Accounts | 100,000 |
| Leads | 500,000 |
| Opportunities | 200,000 |
| Activities | 5,000,000 |
| Emails | 2,000,000 |

### Database Scaling

| Practice | Requirement |
|----------|-------------|
| Read replicas | Report and dashboard queries routed to read replica |
| Connection pooling | PgBouncer or Prisma pool; max 100 connections to primary |
| Query indexing | Index on all FK columns, tenantId, status, createdAt, and high-cardinality filters |
| Pagination | All list endpoints paginated (max 100/page); no unbounded queries |
| Soft delete | Indexes include WHERE deletedAt IS NULL partial indexes |

---

## NFR-AVAIL: Availability & Reliability

| Metric | Target | Notes |
|--------|--------|-------|
| Monthly uptime | ≥ 99.9% | < 44 min downtime / 30 days |
| RTO | < 1 hour | Time to recover full service after failure |
| RPO | < 1 hour | Max data loss in worst-case failure |
| MTTR | < 30 minutes | Mean time to restore after an incident |
| Deployment downtime | 0 minutes | Rolling updates with health checks |

### Resilience Patterns

| Pattern | Applied To |
|---------|-----------|
| Circuit breaker | External email provider; S3 storage; webhook delivery |
| Retry with backoff | Webhook outbound delivery (3 retries: 1s, 5s, 30s) |
| Queue-based async | Email sending, CSV import, report export, bulk operations |
| Health checks | Liveness: `/api/health`; Readiness: `/api/health/ready` (DB + Redis ping) |
| Graceful shutdown | SIGTERM handler: drain in-flight requests (30s grace), close DB pool |

---

## NFR-MAINT: Maintainability

### Code Quality Gates (CI Pipeline)

```
1. Lint (ESLint + Prettier) — fail on any error
2. TypeScript type check — no implicit any; strict mode
3. Unit tests (Jest) — fail if coverage drops below threshold
4. Integration tests — fail if any test fails
5. Build — fail if compile errors
6. Security scan (npm audit) — fail on critical CVEs
7. OpenAPI spec generation — fail if spec diff contains breaking changes
```

### Coverage Thresholds

| Layer | Minimum Coverage |
|-------|-----------------|
| Backend — NestJS services/controllers | 75% line coverage |
| Backend — Prisma repositories | 70% |
| Frontend — React components (unit) | 60% |
| E2E — Playwright critical paths | All 8 critical paths covered |

### Code Standards

| Practice | Tool |
|----------|------|
| Formatting | Prettier (tab width 2, single quotes) |
| Linting | ESLint with TypeScript-eslint plugin |
| Type safety | TypeScript strict mode; no `any` |
| API contracts | OpenAPI 3.0 spec auto-generated via `@nestjs/swagger` |
| DB migrations | Prisma Migrate only — no ad-hoc schema changes |
| Environment config | Zod-validated env schema at startup |
| Secrets | dotenv for local dev; environment variables in production |
| 12-Factor | Full compliance: config via env, stateless processes, disposable containers |

---

## NFR-UX: Usability & Accessibility

| Requirement | Standard / Target |
|-------------|------------------|
| Accessibility | WCAG 2.1 Level AA |
| Keyboard navigation | Full keyboard support for all interactive elements |
| Color contrast | ≥ 4.5:1 for normal text; ≥ 3:1 for large text |
| ARIA attributes | All interactive components have appropriate roles and labels |
| Screen reader | Tested with NVDA (Windows) and VoiceOver (macOS) |
| Responsive breakpoints | 768px (tablet), 1024px (desktop), 1440px (wide), 2560px (4K) |
| Loading states | Skeleton loaders on all async data fetches — no blank screens |
| Optimistic updates | Create/update operations reflected immediately; rolled back on error |
| Error states | Inline validation errors on forms; toast notifications for async errors |
| Empty states | Illustrated empty states with contextual CTA on all list views |
| Keyboard shortcuts | Cmd/Ctrl+K (global search), Cmd/Ctrl+N (new record), Esc (close modal) |

---

## NFR-OBS: Observability

### Logging

```json
// Every log line is structured JSON with these fields:
{
  "level": "info | warn | error | debug",
  "timestamp": "ISO 8601",
  "correlationId": "UUID (per request)",
  "tenantId": "UUID (when authenticated)",
  "userId": "UUID (when authenticated)",
  "service": "api | worker | scheduler",
  "module": "auth | leads | opportunities | ...",
  "message": "Human-readable description",
  "meta": { /* additional context */ }
}
```

**PII Rule**: Email addresses, phone numbers, and personal names must NOT appear in log output.

### Metrics (Prometheus-compatible)

| Metric | Type | Labels |
|--------|------|--------|
| `http_request_duration_seconds` | Histogram | method, route, status_code |
| `http_requests_total` | Counter | method, route, status_code |
| `db_query_duration_seconds` | Histogram | operation, model |
| `queue_job_duration_seconds` | Histogram | queue, job_type |
| `queue_job_failures_total` | Counter | queue, job_type |
| `active_tenants_total` | Gauge | plan |
| `active_users_total` | Gauge | — |

### Alerting Thresholds

| Alert | Condition | Severity |
|-------|-----------|---------|
| High error rate | 5XX errors > 1% over 5 minutes | Critical |
| Slow API | p95 latency > 1,000ms over 5 minutes | Warning |
| DB connection pool exhausted | Pool utilization > 90% | Critical |
| Queue backlog | Job queue depth > 1,000 | Warning |
| Uptime | Health check failure for > 2 minutes | Critical |

---

## NFR-COMP: Compliance

| Requirement | Mechanism |
|-------------|-----------|
| GDPR Art. 17 (erasure) | Cascade PII deletion; pseudonymization in audit logs |
| GDPR Art. 20 (portability) | Full tenant data export endpoint (JSON + CSV) |
| GDPR Art. 30 (records) | Processing records doc; consent field on Contact |
| Audit trail | Immutable append-only audit_logs table |
| Data retention | Configurable retention policies; automated archival job |
| Encryption at rest | AES-256 for DB and S3 |
| Access logging | Every API request logged with correlationId and userId |
| Change management | CI/CD gates enforce review + automated quality checks before merge |
