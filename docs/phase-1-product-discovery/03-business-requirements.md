# Business Requirements — OpsNext CRM

Business requirements define what the system must do from a commercial, operational, and compliance perspective — independent of technical implementation.

---

## Category: Multi-Tenancy

### BR-001: Logical Multi-Tenant Data Isolation
Every data record scoped to a tenant must be inaccessible to users of any other tenant. Isolation must be enforced at the application layer (ORM middleware) AND the database layer (PostgreSQL Row-Level Security). A misconfigured application layer must not expose another tenant's data.

**Acceptance**: Penetration test confirms cross-tenant data access is structurally impossible via normal API paths.

---

### BR-002: Self-Service Tenant Provisioning
A new organization must be able to sign up and have a fully operational CRM environment — including default pipeline, system roles, and an admin user account — without any manual intervention from OpsNext staff.

**Acceptance**: Signup-to-first-login < 5 minutes. Automated provisioning covers: Tenant record, Admin User, default roles (5 system roles), default pipeline (6 stages), tenant settings defaults.

---

### BR-003: Tenant Lifecycle Management
The system must support tenant status transitions: `active` → `suspended` → `cancelled`. Suspension blocks new record creation but preserves read access. Cancellation triggers a 30-day data retention window before scheduled purge.

**Acceptance**: Status transitions immediate. Suspended tenants cannot create/update records. Data purge is scheduled, not instant.

---

### BR-004: Per-Tenant Configuration
Each tenant must be independently configurable for: custom pipeline stages, custom fields on entities, email notification preferences, timezone/locale defaults, and branding (logo, color theme in v2).

**Acceptance**: Tenant A's custom pipeline does not appear in Tenant B. Custom fields are isolated to their creating tenant.

---

## Category: Subscription & Commercials

### BR-005: Plan-Based Feature Gating
Feature access must be governed by the tenant's subscription plan (free, starter, growth, enterprise). The data model must carry a `plan` field on Tenant. API-layer guards enforce plan limits (e.g., seat count, module access).

**Acceptance**: Free plan capped at 3 users. Starter plan blocks advanced reporting. Gate checks enforced server-side — not just UI-hidden.

---

### BR-006: Stripe Integration Readiness
The data model and provisioning flow must be designed to accommodate Stripe billing integration (subscription webhooks, plan upgrades/downgrades, invoicing) without schema changes. Billing is not implemented in v1 but the schema must support it.

**Acceptance**: Tenant table carries `stripeCustomerId`, `stripePriceId`, `subscriptionStatus` fields. Provisioning flow has a stub billing hook.

---

## Category: Security & Access Control

### BR-007: Role-Based Access Control (RBAC)
Every user must have exactly one role. Roles define a permission set. Permissions are expressed as `resource:action` strings. Custom roles must be configurable by tenant admins without engineering involvement.

**Acceptance**: Permission enforcement at API route level. Unauthorized requests return 403 with a structured error body. Frontend reflects permissions but is not the enforcement point.

---

### BR-008: Granular Permission Model
Permissions must support at minimum: per-resource read/write/delete control. Example: a CSM role with read-only access to leads but full access to accounts and contacts.

**Acceptance**: A custom role with `leads:read` only — user can list/view leads but receives 403 on create/update/delete. Verified in integration tests.

---

### BR-009: Immutable Audit Logging
Every create, update, and delete operation on a business entity must produce an audit log record containing: actor, tenant, entity type, entity ID, operation, timestamp, before-state, and after-state. Audit records must be append-only — no update or delete endpoint exists.

**Acceptance**: 100% write operations produce audit entries. No UPDATE/DELETE SQL is possible on the audit_logs table. Verified via DB constraint + application test.

---

### BR-010: Session Security
JWT access tokens expire in 15 minutes. Refresh tokens expire in 30 days and are single-use (rotated on each use). Revoked refresh tokens are tracked in Redis. Logout invalidates the current refresh token immediately.

**Acceptance**: Reuse of a refresh token after rotation returns 401. Logout immediately invalidates token. Verified in auth integration tests.

---

## Category: Compliance & Data Governance

### BR-011: GDPR — Right to Erasure
Upon a verified erasure request, all PII associated with a user or contact must be deleted (or pseudonymized) within 30 days. Audit log entries referencing the deleted subject must be retained but with PII fields nulled.

**Acceptance**: Erasure endpoint exists in admin. PII fields nulled. Audit log retains event structure with `[deleted]` placeholders. Cascade delete of all personal data verified.

---

### BR-012: GDPR — Data Portability
Tenant administrators must be able to request a full export of all tenant data in JSON or CSV format. Export must include: users, contacts, accounts, leads, opportunities, activities, emails, notes.

**Acceptance**: Export triggered via admin API. Delivered as downloadable ZIP within 24 hours (async job for large datasets). Export completeness verified by integration test.

---

### BR-013: GDPR — Data Processing Records
The system must maintain records of data processing activities: what PII is stored, for what purpose, under what legal basis, and for how long.

**Acceptance**: DPA documentation maintained. Consent field on Contact record (`marketingConsent: boolean`). Data retention policy documented and enforced by automated archival jobs.

---

### BR-014: SOC 2 Type II Design Alignment
The system design must align with SOC 2 Trust Service Criteria: Security (access controls, encryption), Availability (uptime, incident response), Confidentiality (data isolation, access logging), and Change Management (CI/CD gates, deployment audit).

**Acceptance**: Security controls documented. Encryption at rest (AES-256) and in transit (TLS 1.2+) confirmed. Change management pipeline enforces review + automated gates.

---

### BR-015: Data Residency Support
Infrastructure must be deployable in multiple cloud regions (EU-West, US-East, APAC). Tenant data must never leave the region it was provisioned in.

**Acceptance**: Docker Compose and deployment configuration parameterized by region. Database and storage are co-located per deployment. No cross-region data replication in v1.

---

## Category: Availability & Operations

### BR-016: 99.9% Uptime SLA
The platform must sustain 99.9% availability (< 44 minutes downtime per rolling 30-day window), excluding pre-announced maintenance windows of < 2 hours.

**Acceptance**: External uptime monitoring configured from day one. SLA tracking dashboard available. Zero-downtime deployment strategy validated.

---

### BR-017: Zero-Downtime Deployments
New versions must be deployable without service interruption. Rolling update strategy with health check validation before traffic is cut over.

**Acceptance**: Deployment process documented. Health check endpoints (`/api/health`, `/api/health/ready`) respond correctly before and after deployment. Load balancer config validated.

---

### BR-018: Automated Database Backups
Daily full database backups must be automated and tested. WAL (Write-Ahead Log) archiving for point-in-time recovery. Backup restoration tested monthly.

**Acceptance**: Backup job runs daily. Restoration procedure documented and tested in staging. RPO < 1 hour confirmed.

---

### BR-019: Observability
The system must emit structured logs (JSON), expose metrics (request rate, error rate, latency, DB pool usage), and support distributed tracing with correlation IDs.

**Acceptance**: Every request carries a correlation ID through all log lines. Metrics endpoint available for scraping. Error monitoring (Sentry or equivalent) integrated.

---

## Category: API & Integration

### BR-020: API-First Design
Every capability accessible via the UI must be accessible via a versioned REST API (`/api/v1/`). No GUI-only features. API is the contract — UI is a consumer of the API.

**Acceptance**: OpenAPI 3.0 specification auto-generated from NestJS decorators covers 100% of endpoints. All endpoints return structured error responses (RFC 7807).

---

### BR-021: Webhook Delivery
Tenants must be able to configure webhook endpoints that receive HTTP POST notifications for key domain events (lead created, opportunity stage changed, deal closed, etc.). Delivery includes retry logic with exponential backoff.

**Acceptance**: Webhook configuration UI per tenant. Events delivered within 30 seconds of occurrence. 3 retry attempts (1s, 5s, 30s). Delivery log queryable per tenant.

---

### BR-022: API Rate Limiting
All API endpoints must be rate-limited to prevent abuse and ensure fair usage across tenants. Rate limits are enforced per user and per tenant.

**Acceptance**: Authenticated users: 1,000 req/min. Auth endpoints (per IP): 100 req/min. Plan-based overrides configurable. 429 responses include `Retry-After` header.

---

## Category: Data Quality & Portability

### BR-023: Soft Delete & Recovery
Business entities must not be permanently removed on user delete action. Records are soft-deleted (archived with a `deletedAt` timestamp). Admins can recover soft-deleted records within 30 days.

**Acceptance**: All entity DELETE endpoints set `deletedAt` only. Default queries exclude soft-deleted records. Admin recovery endpoint available. After 30 days, eligible for hard-delete purge job.

---

### BR-024: Data Import (CSV)
Users must be able to import Contacts, Accounts, and Leads via CSV upload with a field-mapping UI. Imports must show a validation preview before committing, and report per-row errors after import.

**Acceptance**: CSV import wizard: upload → field map → preview (first 10 rows) → confirm → async import. Import job result: success count + error log downloadable as CSV.

---

### BR-025: File Attachment Storage
Files attached to CRM records must be stored in cloud object storage (S3-compatible). File access must use short-lived pre-signed URLs (1-hour expiry) to prevent unauthorized access.

**Acceptance**: Upload returns a pre-signed URL valid 1 hour. Direct public URL access returns 403. File size limit (25MB) enforced server-side. Tenant storage quota tracked.

---

## Requirements Priority Matrix

| ID | Category | Priority | Module |
|----|----------|----------|--------|
| BR-001 | Multi-Tenancy | P0 | Platform |
| BR-002 | Multi-Tenancy | P0 | Tenant Mgmt |
| BR-003 | Multi-Tenancy | P1 | Tenant Mgmt |
| BR-004 | Multi-Tenancy | P1 | Tenant Mgmt |
| BR-005 | Subscription | P1 | Platform |
| BR-006 | Subscription | P2 | Platform |
| BR-007 | Security | P0 | IAM |
| BR-008 | Security | P0 | IAM |
| BR-009 | Security | P0 | Audit |
| BR-010 | Security | P0 | IAM |
| BR-011 | Compliance | P0 | Platform |
| BR-012 | Compliance | P1 | Platform |
| BR-013 | Compliance | P1 | Platform |
| BR-014 | Compliance | P1 | Platform |
| BR-015 | Compliance | P2 | Infrastructure |
| BR-016 | Availability | P0 | Infrastructure |
| BR-017 | Availability | P0 | Infrastructure |
| BR-018 | Availability | P0 | Infrastructure |
| BR-019 | Availability | P1 | Infrastructure |
| BR-020 | API | P0 | Platform |
| BR-021 | Integration | P1 | Integration |
| BR-022 | API | P0 | Platform |
| BR-023 | Data Quality | P0 | Platform |
| BR-024 | Data Quality | P1 | Import |
| BR-025 | Storage | P1 | Storage |
