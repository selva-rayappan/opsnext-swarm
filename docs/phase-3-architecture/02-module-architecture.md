# Module Architecture — OpsNext CRM

---

## Monorepo Structure

```
opsnext-crm/
├── apps/
│   ├── api/                          # NestJS backend
│   │   ├── src/
│   │   │   ├── main.ts               # Bootstrap entrypoint
│   │   │   ├── app.module.ts         # Root module
│   │   │   ├── modules/
│   │   │   │   ├── iam/              # Identity & Access Management
│   │   │   │   ├── tenant/           # Tenant Management
│   │   │   │   ├── contact/          # Contact Management
│   │   │   │   ├── account/          # Account Management
│   │   │   │   ├── lead/             # Lead Management
│   │   │   │   ├── pipeline/         # Pipeline & Opportunity
│   │   │   │   ├── activity/         # Activity & Task Management
│   │   │   │   ├── email/            # Email & Communication
│   │   │   │   ├── analytics/        # Reporting & Dashboards
│   │   │   │   └── notification/     # Notification Delivery
│   │   │   ├── common/               # Shared cross-cutting
│   │   │   │   ├── guards/           # AuthGuard, PermissionsGuard
│   │   │   │   ├── middleware/        # TenantMiddleware, CorrelationMiddleware
│   │   │   │   ├── interceptors/     # LoggingInterceptor, TransformInterceptor
│   │   │   │   ├── filters/          # AllExceptionsFilter
│   │   │   │   ├── decorators/       # @CurrentUser, @Permissions, @Tenant
│   │   │   │   ├── pipes/            # ValidationPipe, ParseUUIDPipe
│   │   │   │   └── utils/            # pagination, sorting helpers
│   │   │   ├── prisma/               # Prisma module (singleton service)
│   │   │   └── config/               # Zod-validated environment config
│   │   ├── test/                     # Integration tests (Jest)
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   └── migrations/
│   │   └── Dockerfile
│   │
│   └── web/                          # Next.js frontend
│       ├── app/                      # App Router pages & layouts
│       │   ├── (auth)/               # Login, register, forgot-password
│       │   ├── (dashboard)/          # Protected app routes
│       │   │   ├── dashboard/
│       │   │   ├── contacts/
│       │   │   ├── accounts/
│       │   │   ├── leads/
│       │   │   ├── pipeline/
│       │   │   ├── activities/
│       │   │   ├── emails/
│       │   │   ├── reports/
│       │   │   └── settings/
│       │   └── api/                  # Next.js Route Handlers (BFF)
│       ├── components/               # Reusable UI components
│       │   ├── ui/                   # Primitives (Button, Input, Modal, etc.)
│       │   ├── layout/               # Sidebar, Navbar, Breadcrumbs
│       │   ├── tables/               # DataTable, SortableColumn
│       │   ├── forms/                # RHF-integrated form components
│       │   ├── timeline/             # ActivityTimeline, TimelineItem
│       │   └── charts/               # KPI cards, bar charts, funnel chart
│       ├── lib/
│       │   ├── api/                  # Axios client + typed API functions
│       │   ├── auth/                 # Auth context, session management
│       │   └── utils/
│       └── Dockerfile
│
├── packages/
│   └── shared/                       # Shared TypeScript types and Zod schemas
│       ├── src/
│       │   ├── types/                # Entity interfaces shared between api + web
│       │   ├── schemas/              # Zod schemas (request validation)
│       │   └── constants/            # Enums, permission strings
│       └── package.json
│
├── docker-compose.yml
├── docker-compose.prod.yml
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
└── package.json                      # Turborepo root
```

---

## NestJS Module Anatomy (Standard Pattern)

Every module follows the same internal structure. This is the canonical pattern for all 10 modules.

```
modules/lead/
├── lead.module.ts             # Module definition: imports, providers, exports
├── lead.controller.ts         # HTTP route handlers (decorators, validation)
├── lead.service.ts            # Business logic orchestration
├── lead.repository.ts         # Prisma data access (all DB queries live here)
├── lead.events.ts             # Domain event types for this module
├── lead.event-handler.ts      # Subscribes to and handles events from other modules
├── dto/
│   ├── create-lead.dto.ts     # Validated input shape for POST
│   ├── update-lead.dto.ts     # Validated input shape for PATCH (PartialType)
│   ├── lead-filter.dto.ts     # Query params for list endpoint
│   └── lead-response.dto.ts   # Output shape (excludes internal fields)
├── entities/
│   └── lead.entity.ts         # TypeScript class matching Prisma model
└── __tests__/
    ├── lead.service.spec.ts   # Unit tests (mocked repository)
    └── lead.controller.spec.ts # Unit tests (mocked service)
```

---

## Module 1: IAM Module (`/modules/iam/`)

**Responsibility**: Authentication, session management, user CRUD, RBAC.

### Structure
```
iam/
├── iam.module.ts
├── auth/
│   ├── auth.controller.ts        # /auth/login, /auth/register, /auth/refresh, etc.
│   ├── auth.service.ts           # JWT issue, bcrypt compare, token rotation
│   ├── strategies/
│   │   └── jwt.strategy.ts       # Passport JWT strategy (RS256)
│   ├── guards/
│   │   ├── jwt-auth.guard.ts     # Validates JWT on protected routes
│   │   └── permissions.guard.ts  # Checks user permissions against route metadata
│   └── dto/
│       ├── login.dto.ts
│       ├── register.dto.ts
│       └── refresh-token.dto.ts
├── users/
│   ├── users.controller.ts       # CRUD for users within tenant
│   ├── users.service.ts
│   ├── users.repository.ts
│   └── dto/
├── roles/
│   ├── roles.controller.ts       # CRUD for roles
│   ├── roles.service.ts
│   ├── roles.repository.ts
│   └── dto/
├── teams/
│   ├── teams.controller.ts
│   ├── teams.service.ts
│   ├── teams.repository.ts
│   └── dto/
└── audit/
    ├── audit.service.ts          # Write-only audit log service
    ├── audit.repository.ts
    └── audit-log.interceptor.ts  # Intercepts all write ops, writes audit entry
```

### Key Service Methods
```typescript
// auth.service.ts
login(email: string, password: string, tenantSlug: string): Promise<AuthTokens>
refreshTokens(refreshToken: string): Promise<AuthTokens>
logout(userId: string, refreshToken: string): Promise<void>
forgotPassword(email: string): Promise<void>
resetPassword(token: string, newPassword: string): Promise<void>
inviteUser(dto: InviteUserDto, tenantId: string): Promise<void>

// users.service.ts
findAll(tenantId: string, filters: UserFilterDto): Promise<PaginatedResult<User>>
findById(userId: string, tenantId: string): Promise<User>
deactivate(userId: string, tenantId: string): Promise<void>
bulkReassign(fromUserId: string, toUserId: string, tenantId: string): Promise<ReassignResult>
```

### Exported Providers (available to other modules)
- `AuthGuard` (JwtAuthGuard)
- `PermissionsGuard`
- `AuditService`
- `CurrentUser` decorator

---

## Module 2: Tenant Module (`/modules/tenant/`)

**Responsibility**: Tenant provisioning, lifecycle, settings.

```
tenant/
├── tenant.module.ts
├── tenant.controller.ts          # Admin-only tenant CRUD
├── tenant.service.ts             # Provisioning orchestration
├── tenant.repository.ts
├── provisioning/
│   └── provisioning.service.ts   # Creates: Tenant + Admin User + Default Roles + Default Pipeline
└── dto/
    ├── create-tenant.dto.ts
    └── update-tenant-settings.dto.ts
```

### Provisioning Flow
```
createTenant(dto)
  1. Create Tenant record (status: active)
  2. Create Admin User (status: pending_invite)
  3. Create 5 system Role records with default permissions
  4. Assign Admin User → SuperAdmin role
  5. Create default Pipeline ("Sales Pipeline") + 6 default Stages
  6. Emit TenantProvisioned event
  7. [Event handler] Queue welcome email to admin
```

---

## Module 3: Contact Module (`/modules/contact/`)

**Responsibility**: Contact CRUD, ContactAccount relationships, contact relationships, merge.

```
contact/
├── contact.module.ts
├── contact.controller.ts
├── contact.service.ts
├── contact.repository.ts
├── merge/
│   └── contact-merge.service.ts   # Complex merge logic — isolated service
└── dto/
    ├── create-contact.dto.ts
    ├── update-contact.dto.ts
    ├── contact-filter.dto.ts
    └── merge-contacts.dto.ts
```

### Key Relationships in Repository
```typescript
// contact.repository.ts
findWithTimeline(contactId: string, tenantId: string): Promise<ContactWithTimeline>
// Joins: activities, emails, notes, tasks ordered by occurredAt DESC

findDuplicatesByEmail(email: string, tenantId: string): Promise<Contact[]>
// Checks for email match before save — soft warning, not block

mergeContacts(survivingId: string, mergedId: string, tenantId: string, actorId: string): Promise<Contact>
// Transaction: re-parent all relations → soft-delete merged → emit ContactsMerged
```

---

## Module 4: Account Module (`/modules/account/`)

**Responsibility**: Account CRUD, account-contact associations, account timeline.

```
account/
├── account.module.ts
├── account.controller.ts
├── account.service.ts
├── account.repository.ts
└── dto/
```

---

## Module 5: Lead Module (`/modules/lead/`)

**Responsibility**: Lead lifecycle from capture to conversion. ACL to Contact/Account/Pipeline on conversion.

```
lead/
├── lead.module.ts
├── lead.controller.ts
├── lead.service.ts
├── lead.repository.ts
├── conversion/
│   └── lead-conversion.service.ts  # Orchestrates: create Contact + Account + Opportunity
├── import/
│   └── lead-import.service.ts      # CSV parsing, field mapping, batch insert
└── dto/
    ├── create-lead.dto.ts
    ├── update-lead.dto.ts
    ├── lead-filter.dto.ts
    ├── convert-lead.dto.ts          # Optional contact/account/opportunity creation
    └── ingest-lead.dto.ts           # Public API ingest endpoint DTO
```

### Conversion Service Contract
```typescript
// lead-conversion.service.ts
convertLead(leadId: string, dto: ConvertLeadDto, tenantId: string, actorId: string): Promise<ConversionResult>
// dto.createContact?: CreateContactDto
// dto.createAccount?: CreateAccountDto
// dto.createOpportunity?: CreateOpportunityDto
// Returns: { contactId?, accountId?, opportunityId? }
// Marks lead.status = 'converted', sets convertedAt, sets converted*Id fields
// Emits LeadConverted domain event
```

---

## Module 6: Pipeline Module (`/modules/pipeline/`)

**Responsibility**: Pipeline + Stage configuration, Opportunity CRUD, Kanban view, forecasting.

```
pipeline/
├── pipeline.module.ts
├── pipelines/
│   ├── pipeline.controller.ts   # Pipeline CRUD + stage management
│   ├── pipeline.service.ts
│   └── pipeline.repository.ts
├── opportunities/
│   ├── opportunity.controller.ts  # Opportunity CRUD + stage changes
│   ├── opportunity.service.ts
│   └── opportunity.repository.ts
├── kanban/
│   └── kanban.service.ts         # Efficient kanban board data query
└── dto/
    ├── create-pipeline.dto.ts
    ├── create-opportunity.dto.ts
    ├── update-opportunity.dto.ts
    ├── change-stage.dto.ts
    └── kanban-query.dto.ts
```

### Stage Change Invariant
```typescript
// opportunity.service.ts
changeStage(opportunityId: string, newStageId: string, tenantId: string, actorId: string): Promise<Opportunity>
// Validates newStage.pipelineId === opportunity.pipelineId
// Sets opportunity.probability = newStage.probability (unless manually overridden)
// If newStage.isWon: sets status = 'won', closedAt = now()
// If newStage.isLost: requires lostReason in dto; sets status = 'lost', closedAt = now()
// Logs OpportunityStageChanged activity entry
// Emits OpportunityStageChanged or OpportunityClosedWon/Lost domain event
```

---

## Module 7: Activity Module (`/modules/activity/`)

**Responsibility**: Tasks, call/meeting logging, activity timeline, notifications trigger.

```
activity/
├── activity.module.ts
├── tasks/
│   ├── task.controller.ts
│   ├── task.service.ts
│   └── task.repository.ts
├── logs/
│   ├── activity-log.controller.ts   # Log call, meeting, note (manual)
│   ├── activity-log.service.ts
│   └── activity-log.repository.ts
├── timeline/
│   └── timeline.service.ts          # Aggregates all activity types for an entity
└── dto/
```

### Timeline Service
```typescript
// timeline.service.ts
getEntityTimeline(entityType: string, entityId: string, tenantId: string, pagination: PaginationDto): Promise<PaginatedTimeline>
// Unions: activities + notes + emails + tasks + audit entries for the entity
// Orders by occurredAt/createdAt DESC
// Returns typed items with actorName, timestamp, type, content
```

---

## Module 8: Email Module (`/modules/email/`)

**Responsibility**: Email logging, templates, BCC ingest stub.

```
email/
├── email.module.ts
├── email.controller.ts
├── email.service.ts
├── email.repository.ts
├── templates/
│   ├── template.controller.ts
│   ├── template.service.ts
│   └── template.repository.ts
├── ingest/
│   └── bcc-ingest.service.ts     # Parses inbound email, links to CRM entity
└── dto/
```

---

## Module 9: Analytics Module (`/modules/analytics/`)

**Responsibility**: Pre-built dashboards, reports, KPI aggregation, CSV export.

```
analytics/
├── analytics.module.ts
├── dashboards/
│   ├── dashboard.controller.ts
│   └── dashboard.service.ts      # Orchestrates KPI widget queries
├── reports/
│   ├── report.controller.ts
│   └── report.service.ts         # Runs parameterized report queries
├── metrics/
│   ├── pipeline.metrics.ts       # Pipeline value, win rate, avg deal size
│   ├── lead.metrics.ts           # Lead count, conversion rate, source analysis
│   ├── activity.metrics.ts       # Call/email/meeting counts per rep/period
│   └── team.metrics.ts           # Team performance leaderboard
└── export/
    └── export.service.ts         # CSV generation, queued via BullMQ
```

### Query Routing
Analytics read queries are routed to the PostgreSQL read replica (when configured) via a dedicated Prisma client instance.

---

## Module 10: Notification Module (`/modules/notification/`)

**Responsibility**: In-app and email notification delivery. Subscribes to domain events.

```
notification/
├── notification.module.ts
├── notification.controller.ts     # GET /notifications, PATCH (mark read)
├── notification.service.ts        # Create notification records
├── notification.repository.ts
├── delivery/
│   ├── in-app.service.ts          # Writes to notifications table; SSE/polling
│   └── email.service.ts           # Queues transactional email via BullMQ
├── handlers/
│   ├── lead-assigned.handler.ts   # @OnEvent('lead.assigned')
│   ├── task-due.handler.ts        # @OnEvent('task.due')
│   ├── opportunity-stage.handler.ts
│   └── ...                        # One handler per relevant domain event
└── templates/
    └── email-templates/           # Handlebars templates for notification emails
```

---

## Common Layer (`/common/`)

### Key Middleware

```typescript
// TenantMiddleware — runs on every authenticated request
// Reads tenantId from req.user (JWT claim)
// Attaches to req.tenantId
// Prisma client extension reads this and appends WHERE tenantId = ?

// CorrelationMiddleware — runs on every request
// Reads or generates X-Request-ID header
// Attaches to req.correlationId
// Logger reads this for structured log output
```

### Key Decorators

```typescript
@CurrentUser()           // Extract the authenticated user from request
@Permissions('leads:create')  // Declare required permissions on a route handler
@Tenant()                // Extract tenantId from request
@ApiPaginatedResponse()  // OpenAPI decorator for paginated list endpoints
@Public()                // Mark a route as unauthenticated (e.g., /auth/login)
```

### Pagination Pattern

All list endpoints accept and return:
```typescript
// Query params (via ListQueryDto):
{ page: number, limit: number (max 100), sortBy: string, sortOrder: 'asc'|'desc' }

// Response envelope:
{ data: T[], meta: { total: number, page: number, limit: number, totalPages: number } }
```

---

## API Route Map Summary

| Module | Base Path |
|--------|-----------|
| Auth | `/api/v1/auth` |
| Users | `/api/v1/users` |
| Roles | `/api/v1/roles` |
| Teams | `/api/v1/teams` |
| Tenants | `/api/v1/tenants` |
| Contacts | `/api/v1/contacts` |
| Accounts | `/api/v1/accounts` |
| Leads | `/api/v1/leads` |
| Pipelines | `/api/v1/pipelines` |
| Opportunities | `/api/v1/opportunities` |
| Tasks | `/api/v1/tasks` |
| Activities | `/api/v1/activities` |
| Emails | `/api/v1/emails` |
| Email Templates | `/api/v1/email-templates` |
| Analytics | `/api/v1/analytics` |
| Notifications | `/api/v1/notifications` |
| Audit Logs | `/api/v1/audit-logs` |
| Admin | `/api/v1/admin` |
| Health | `/api/health`, `/api/health/ready` |

OpenAPI spec auto-generated at `/api/docs` (Swagger UI) and `/api/docs-json` (raw JSON).
