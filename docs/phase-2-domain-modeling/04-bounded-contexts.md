# Bounded Contexts — OpsNext CRM

---

## BC-01: Identity & Access Management (IAM)

**Classification**: Generic Subdomain  
**Type**: Upstream context — all other contexts depend on it

### Responsibilities
- Authenticate users (email/password, future: OAuth/OIDC)
- Issue and validate JWT access tokens and refresh tokens
- Enforce RBAC: role assignment, permission resolution
- Manage user lifecycle (invite, activate, deactivate)
- Session management (token rotation, invalidation)

### Aggregate Roots Owned
- `User`
- `Role`
- `RefreshToken`

### Ubiquitous Language
- **Authentication**: Verifying identity (who are you?)
- **Authorization**: Verifying permission (what can you do?)
- **Principal**: The authenticated user making a request
- **Claim**: A piece of verified data in a JWT (userId, tenantId, roleId)

### Upstream Dependencies
- Tenant Management (must exist before users can be created)

### Downstream Consumers
- All other contexts consume `userId` and `tenantId` from IAM JWT claims
- No context calls IAM APIs at runtime — they trust the JWT

### Integration Pattern
**Open Host Service**: IAM publishes a well-defined JWT structure. All contexts validate the JWT independently using the public key — no runtime IAM calls on protected requests.

### API Boundary
```
POST   /api/v1/auth/register
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh
POST   /api/v1/auth/logout
POST   /api/v1/auth/forgot-password
POST   /api/v1/auth/reset-password
GET    /api/v1/auth/me
```

---

## BC-02: Tenant Management

**Classification**: Generic Subdomain  
**Type**: Upstream context for all tenant-scoped data

### Responsibilities
- Provision new tenants (create Tenant record + admin user + default roles + default pipeline)
- Manage tenant lifecycle (active, suspended, cancelled)
- Store tenant-level configuration (plan, settings, theme, timezone, locale)
- Enforce plan-level feature gates

### Aggregate Roots Owned
- `Tenant`
- `Team`

### Ubiquitous Language
- **Tenant**: An organization subscribed to OpsNext CRM
- **Provisioning**: The act of creating a fully initialized Tenant with default data
- **Plan**: The subscription tier that gates feature access
- **Workspace**: The tenant's isolated environment in OpsNext CRM

### Upstream Dependencies
- None (root context)

### Downstream Consumers
- All contexts carry `tenantId` on every entity
- Feature gate checks consult Tenant.plan

### Integration Pattern
**Shared Kernel**: `tenantId` (UUID) is the universal shared key. Every context carries it. The Tenant entity is never replicated — other contexts reference it by ID only.

### API Boundary
```
POST   /api/v1/tenants                    (admin provisioning)
GET    /api/v1/tenants/:id
PATCH  /api/v1/tenants/:id
GET    /api/v1/tenants/:id/settings
PATCH  /api/v1/tenants/:id/settings
GET    /api/v1/users                      (user management)
POST   /api/v1/users/invite
PATCH  /api/v1/users/:id
DELETE /api/v1/users/:id
GET    /api/v1/teams
POST   /api/v1/teams
PATCH  /api/v1/teams/:id
```

---

## BC-03: Contact & Account Management

**Classification**: Core Subdomain  
**Type**: Central master data — referenced by all sales contexts

### Responsibilities
- Manage the Account (company) master record
- Manage the Contact (person) master record
- Model Contact ↔ Account associations (M:N with primary)
- Track contact relationships (org chart, stakeholder map)
- Provide the activity timeline for both Account and Contact

### Aggregate Roots Owned
- `Account`
- `Contact`
- `ContactRelationship`

### Ubiquitous Language
- **Account**: A business entity (customer, prospect, partner)
- **Contact**: An individual person at an Account
- **Primary Account**: The main organization a Contact belongs to
- **Stakeholder Map**: The set of Contacts and their roles at an Account
- **Merge**: Combining two duplicate Contact records into one, preserving all history

### Upstream Dependencies
- IAM (users as owners, assignees)
- Tenant Management (tenantId)

### Downstream Consumers
- Lead Management: Lead.accountId, Lead.contactId
- Sales Pipeline: Opportunity.accountId, Opportunity.primaryContactId
- Communication: Email.contactId, Email.accountId
- Analytics: reads Account and Contact counts

### Integration Pattern
**Customer/Supplier**: Lead and Opportunity contexts are downstream consumers. They reference Account/Contact by ID. They do not own or mutate Account/Contact records.

**Anti-Corruption Layer** on Lead Conversion: When a Lead is converted, the Lead context calls the Contact & Account context's public API to create Contact/Account records. The Lead does not directly write to those tables.

### API Boundary
```
GET/POST       /api/v1/accounts
GET/PATCH/DEL  /api/v1/accounts/:id
GET            /api/v1/accounts/:id/contacts
GET            /api/v1/accounts/:id/timeline

GET/POST       /api/v1/contacts
GET/PATCH/DEL  /api/v1/contacts/:id
POST           /api/v1/contacts/:id/merge
GET            /api/v1/contacts/:id/timeline
GET/POST/DEL   /api/v1/contacts/:id/relationships
```

---

## BC-04: Lead Management

**Classification**: Core Subdomain  
**Type**: Top-of-funnel entry point

### Responsibilities
- Capture leads from multiple sources (manual, CSV, API, BCC email)
- Assign leads to users/teams (manual or round-robin)
- Track lead status through the qualification lifecycle
- Execute lead conversion (create Contact/Account/Opportunity)
- Provide lead-level activity timeline

### Aggregate Roots Owned
- `Lead`

### Ubiquitous Language
- **Lead Capture**: The act of creating a new Lead record
- **Qualification**: Assessing a Lead against defined criteria (BANT etc.)
- **Conversion**: The irreversible act of promoting a qualified Lead into Contact + Account + Opportunity
- **Disqualification**: Closing a Lead as not worth pursuing, with a reason
- **Unassigned Queue**: Leads without an owner, visible to managers

### Upstream Dependencies
- IAM (owner/assignee)
- Tenant Management
- Contact & Account (via ACL on conversion)

### Downstream Consumers
- Sales Pipeline: receives Opportunity created from Lead conversion
- Analytics: lead source, conversion rate, time-to-qualify metrics

### Integration Pattern
**Anti-Corruption Layer (ACL)** on conversion: Lead context calls Contact & Account context's creation APIs. Lead stores returned IDs as `convertedContactId`, `convertedAccountId`, `convertedOpportunityId`.

### API Boundary
```
GET/POST       /api/v1/leads
GET/PATCH/DEL  /api/v1/leads/:id
POST           /api/v1/leads/:id/assign
POST           /api/v1/leads/:id/qualify
POST           /api/v1/leads/:id/disqualify
POST           /api/v1/leads/:id/convert
POST           /api/v1/leads/import         (CSV upload)
GET            /api/v1/leads/:id/timeline
```

---

## BC-05: Sales Pipeline

**Classification**: Core Subdomain  
**Type**: Revenue engine — the heart of the CRM

### Responsibilities
- Define and manage Pipelines and their Stages
- Manage Opportunity lifecycle through pipeline stages
- Track deal values, close dates, and probability
- Support Kanban and list views for pipeline management
- Record closed-won and closed-lost outcomes with reasons
- Provide pipeline analytics: total value, weighted pipeline, win rate

### Aggregate Roots Owned
- `Pipeline`
- `Stage`
- `Opportunity`

### Ubiquitous Language
- **Pipeline**: The sales process workflow with ordered stages
- **Stage**: A discrete step in the pipeline (e.g., "Proposal Sent")
- **Opportunity**: An active deal being worked through the pipeline
- **Win Probability**: The % chance of closing at a given stage
- **Weighted Pipeline**: Sum of (Amount × Probability) across open opportunities
- **Close Date**: The expected date an opportunity will close
- **Closed Won**: An opportunity successfully closed as a deal
- **Closed Lost**: An opportunity that did not close, with a reason

### Upstream Dependencies
- IAM (owner)
- Tenant Management
- Contact & Account (referenced by ID — not owned)

### Downstream Consumers
- Analytics & Reporting: pipeline value, win rate, forecast
- Notifications: stage change events trigger notifications

### Integration Pattern
**Published Language**: Opportunity stage changes emit `OpportunityStageChanged` events, consumed by the Notification and Analytics contexts.

### API Boundary
```
GET/POST       /api/v1/pipelines
GET/PATCH/DEL  /api/v1/pipelines/:id
GET/POST       /api/v1/pipelines/:id/stages
PATCH          /api/v1/pipelines/:id/stages/reorder

GET/POST       /api/v1/opportunities
GET/PATCH/DEL  /api/v1/opportunities/:id
POST           /api/v1/opportunities/:id/stage
GET            /api/v1/opportunities/:id/timeline
GET            /api/v1/pipeline-view/:pipelineId   (kanban data)
```

---

## BC-06: Activity & Task Management

**Classification**: Supporting Subdomain  
**Type**: Cross-cutting — serves all entity contexts

### Responsibilities
- Manage Tasks (create, assign, track, complete)
- Log calls and meetings against any CRM entity
- Deliver reminders for due/overdue tasks
- Aggregate activity into timeline views per entity
- Provide "my tasks" and "team tasks" views

### Aggregate Roots Owned
- `Task`
- `Activity` (log entries — immutable once created)

### Ubiquitous Language
- **Activity**: Any logged interaction (call, meeting, note, email, stage change)
- **Task**: A future action item with a due date and assignee
- **Timeline**: Chronological view of all activities on a CRM record
- **Overdue**: A task past its due date that is not completed

### Upstream Dependencies
- IAM (assignee, creator)
- Tenant Management
- Contact & Account, Lead, Pipeline (linked entity IDs)

### Downstream Consumers
- Notifications: task due/overdue events
- Analytics: activity cadence metrics

### Integration Pattern
**Conformist** for entity links: Activity uses polymorphic `(entityType, entityId)` to link to any context. No strong coupling.

### API Boundary
```
GET/POST       /api/v1/tasks
GET/PATCH/DEL  /api/v1/tasks/:id
POST           /api/v1/tasks/:id/complete
GET            /api/v1/tasks/my              (assignee = me)
GET            /api/v1/activities            (timeline queries)
POST           /api/v1/activities/calls
POST           /api/v1/activities/meetings
```

---

## BC-07: Email & Communication

**Classification**: Supporting Subdomain  
**Type**: Communication history and templates

### Responsibilities
- Log email communications manually or via BCC-to-CRM
- Store email history linked to Contact/Account/Lead/Opportunity
- Manage email templates with merge fields
- Provide communication timeline per entity

### Aggregate Roots Owned
- `Email`
- `EmailTemplate`

### Ubiquitous Language
- **Logged Email**: An email record manually or automatically captured in the CRM
- **BCC Address**: The unique per-user address that captures inbound email via BCC
- **Merge Field**: A template placeholder replaced with record data at send time
- **Communication Timeline**: Chronological email history on a CRM record

### Upstream Dependencies
- IAM (logged by user)
- Tenant Management
- Contact & Account, Lead, Pipeline (linked entity IDs)

### Downstream Consumers
- Analytics: email activity metrics
- Activity timeline (emails appear in entity timelines)

### Integration Pattern
**Published Language**: Email logged events are published to the Activity timeline via the `ActivityCreated` event.

### API Boundary
```
GET/POST       /api/v1/emails
GET            /api/v1/emails/:id
GET/POST       /api/v1/email-templates
GET/PATCH/DEL  /api/v1/email-templates/:id
POST           /api/v1/email-templates/:id/preview
```

---

## BC-08: Analytics & Reporting

**Classification**: Supporting Subdomain  
**Type**: Read-only aggregate views; no domain logic

### Responsibilities
- Serve pre-built dashboards (executive, sales, lead, activity)
- Generate on-demand reports (pipeline, win-loss, rep activity)
- Support data export (CSV)
- Scheduled report delivery (email)

### Aggregate Roots Owned
- `Dashboard` (configuration only — widget layout, filters)
- `Report` (saved report configuration)

### Ubiquitous Language
- **Widget**: A single visualization or KPI card on a dashboard
- **Metric**: A computed value (win rate, pipeline value, avg deal size)
- **Dimension**: A grouping or filter axis (rep, team, stage, period)
- **Forecast**: Projected revenue from open opportunities likely to close

### Upstream Dependencies
- All contexts (reads via database read replica)
- IAM (scope reports to tenant, filter by user/team)

### Downstream Consumers
- None (terminal context)

### Integration Pattern
**Conformist**: Analytics reads data directly from the primary database (or a read replica). No event sourcing in v1. Aggregate queries run as optimized SQL.

### API Boundary
```
GET  /api/v1/dashboards
GET  /api/v1/dashboards/:id
GET  /api/v1/reports
GET  /api/v1/reports/:id/run
GET  /api/v1/reports/:id/export
POST /api/v1/reports/:id/schedule
GET  /api/v1/metrics/pipeline
GET  /api/v1/metrics/leads
GET  /api/v1/metrics/activities
GET  /api/v1/metrics/team
```

---

## Context Integration Summary

```
              ┌──────────────┐
              │  Tenant Mgmt │ (upstream root)
              └──────┬───────┘
                     │ tenantId (shared kernel)
              ┌──────▼───────┐
              │     IAM      │ (upstream — JWT issuer)
              └──────┬───────┘
                     │ userId, roleId (JWT claims)
     ┌───────────────┼──────────────────┐
     ▼               ▼                  ▼
┌──────────┐  ┌──────────────┐  ┌────────────────┐
│  Lead    │  │ Contact &    │  │    Pipeline    │
│  Mgmt   ─┼─►│ Account Mgmt│  │  & Opportunity │
│          │  │  (ACL)       │  │                │
└────┬─────┘  └──────────────┘  └───────┬────────┘
     │                                  │
     └──────────┬───────────────────────┘
                │ entity IDs (polymorphic links)
     ┌──────────▼──────────────────────────────┐
     │        Activity & Task Management        │
     │        Email & Communication             │
     └──────────────────────┬──────────────────┘
                            │ events + data reads
                   ┌────────▼───────────┐
                   │ Analytics &        │
                   │ Reporting          │
                   └────────────────────┘
```
