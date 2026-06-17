# Product Vision — OpsNext CRM

---

## Vision Statement

> **OpsNext CRM exists to give every growth-stage B2B sales team the operating system they need to win customers systematically — without the complexity, cost, or rigidity of legacy enterprise CRM.**

---

## Mission Statement

We build the infrastructure layer for revenue teams: a unified, intelligent, and API-first platform that connects people, data, and process so that sales teams can focus entirely on selling.

---

## The Problem We Solve

Modern B2B sales teams are drowning in tool sprawl. Leads live in spreadsheets. Pipeline is tracked in Notion. Emails are logged manually — or not at all. When a rep leaves, their context leaves with them.

Legacy CRMs (Salesforce, Dynamics) are too expensive, too complex, and too slow to configure. SMB tools (Pipedrive, HubSpot free tier) lack the depth serious sales teams need. The middle market is underserved.

**OpsNext CRM fills this gap**: enterprise-grade capability, modern-stack simplicity, SaaS economics.

---

## Strategic Pillars

### 1. Unified Customer Record
Every contact, account, lead, opportunity, email, call, and note lives in one place. The customer record is the single source of truth — always complete, always current.

### 2. Process-Driven Sales Execution
Teams sell better when they follow a repeatable process. OpsNext enforces pipeline discipline: required fields at stage transitions, task cadences, and visibility into stuck deals.

### 3. Real-Time Visibility
Managers see every deal, every rep, every activity in real time — no chasing for updates, no Friday afternoon pipeline reviews that take two hours.

### 4. API-First, Integrations-Ready
Every UI action is backed by a versioned REST API. Webhooks emit events for every meaningful state change. OpsNext integrates with the tools teams already use, rather than replacing them.

### 5. Security & Tenant Isolation by Design
Multi-tenant from day one. Every tenant's data is architecturally isolated. RBAC is granular. Audit logs are immutable. Compliance is a feature, not a sprint.

### 6. Speed to Value
New teams are productive in hours, not weeks. Self-service onboarding, sensible defaults, and guided setup mean zero implementation cost.

---

## Value Proposition

| Persona | Core Value |
|---------|-----------|
| **Sales Rep** | Never lose context. Every customer touchpoint — logged, linked, searchable. |
| **Sales Manager** | Real-time pipeline health. Catch problems before they become misses. |
| **CRM Admin** | Full configurability without code. Roles, pipelines, fields — all self-service. |
| **VP Sales** | Accurate forecast, clean data, and board-ready reporting in one platform. |
| **Developer / Ops** | API-first, Docker-native, extensible. Fits any tech stack. |

---

## Target Market & Ideal Customer Profile (ICP)

### Primary ICP
| Dimension | Criteria |
|-----------|---------|
| Company size | 50 – 500 employees |
| Revenue stage | $2M – $50M ARR |
| Industry | B2B SaaS, Professional Services, Technology, FinServ |
| Sales motion | Inside sales, outbound-led, or PLG with sales overlay |
| Sales team size | 3 – 50 reps |
| Pain state | Outgrown spreadsheets or Pipedrive; frustrated by Salesforce cost/complexity |
| Tech profile | Engineering team exists; comfortable with Docker, APIs |

### Secondary ICP
| Dimension | Criteria |
|-----------|---------|
| Company size | 500 – 2,000 employees |
| Use case | Multi-pipeline (new business + renewals), territory management |
| Buying trigger | Salesforce contract renewal, M&A consolidation, RevOps hire |

### Buying Committee
| Role | Concern |
|------|---------|
| VP Sales / CRO | Will it improve forecast accuracy and reduce ramp time? |
| RevOps / Sales Ops | Can I configure it without engineering? Does it have an API? |
| CTO / IT | Is it secure? Can we self-host? Is it compliant? |
| CFO | What's the TCO vs. Salesforce? |

---

## Competitive Landscape & Positioning

### Positioning Statement
> For B2B sales teams of 5–200 reps who need enterprise CRM depth without enterprise complexity, **OpsNext CRM** is the modern, API-first sales platform that gives you full pipeline visibility, multi-tenant isolation, and total configurability — at a fraction of the cost of Salesforce, with none of the implementation pain.

### Competitive Differentiation Matrix

| Capability | Salesforce | HubSpot CRM | Pipedrive | **OpsNext CRM** |
|-----------|-----------|-------------|-----------|-----------------|
| Multi-tenant SaaS architecture | Enterprise add-on | No | No | **Native** |
| API-first (full parity) | Partial | Partial | Limited | **Yes** |
| Self-hostable | No | No | No | **Yes (Docker)** |
| Modern stack (NestJS / Next.js) | No (Apex/Visualforce) | Partial | No | **Yes** |
| Setup time | 3–6 months | Days | Hours | **Hours** |
| Cost at 50 seats | $7,500+/mo | $2,400/mo | $1,200/mo | **Competitive** |
| Granular RBAC | Yes | Limited | Limited | **Yes** |
| Immutable audit logs | Yes (extra cost) | No | No | **Yes (built-in)** |
| White-label / OEM | No | No | No | **Roadmap** |

---

## Product Principles

1. **The customer record is sacred.** Nothing that touches a customer is ever permanently deleted without explicit admin action and a 30-day recovery window.

2. **Convention over configuration — but configuration is always possible.** Sensible defaults ship out of the box. Everything is overridable by admins without code.

3. **API parity is non-negotiable.** If you can click it in the UI, you can call it via the API. No GUI-only features, ever.

4. **Tenant isolation is structural, not procedural.** It cannot be misconfigured away. `tenantId` is enforced at the ORM layer on every query.

5. **Audit everything.** Every write operation produces an immutable audit log entry. Trust is built on traceability.

6. **Fail safely.** External service failures (email provider, storage) degrade gracefully. Core CRM functionality never goes down because SendGrid is slow.

7. **Ship fast, stay correct.** CI/CD gates on lint + type-check + tests + security scan. Speed without quality gates is just debt.

8. **No dark patterns.** Data export is always available. Cancellation is instant. Portability is a right, not a feature request.

---

## Product Roadmap Phases

| Phase | Name | Modules | Status |
|-------|------|---------|--------|
| **v1.0** | Foundation | Auth/IAM, Tenant, Contacts, Accounts, Leads, Pipeline, Basic Tasks | In planning |
| **v1.1** | Activity | Meetings, Calls, Email Logging, Reminders, Notifications | Following v1.0 |
| **v1.2** | Intelligence | Reports, Dashboards, Win/Loss Analysis, Activity Analytics | Following v1.1 |
| **v2.0** | Scale | Async event bus (Redis Streams), Calendar sync, API Keys, Webhooks v2 | Post-launch |
| **v2.1** | AI | Lead scoring, Deal health signals, Forecast AI | Future |

---

## Out of Scope for v1.0

- Native mobile apps (responsive web covers tablet/mobile)
- Built-in VoIP / power dialer
- Marketing automation / campaign sequences
- CPQ (Configure, Price, Quote)
- Contract management / e-signature
- Customer support ticketing (separate product)
- Social media monitoring / integration
- AI/ML features (v2.1+)
- Calendar sync (Google/Outlook) — stub interface in v1, full sync in v2.0
