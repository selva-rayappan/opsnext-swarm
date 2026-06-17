# Success Metrics — OpsNext CRM

Success metrics define how we know OpsNext CRM is delivering value. They are organized by audience and measured on a defined cadence.

---

## Business KPIs

These metrics define commercial success for OpsNext CRM as a SaaS product.

### Revenue Metrics

| Metric | Definition | Baseline | 6-Month Target | 12-Month Target |
|--------|-----------|----------|----------------|-----------------|
| Monthly Recurring Revenue (MRR) | Total subscription revenue per month | $0 | $50,000 | $180,000 |
| Annual Recurring Revenue (ARR) | MRR × 12 | $0 | $600,000 | $2,160,000 |
| MRR Growth Rate | (MRR this month − last month) / last month × 100 | — | ≥ 15% MoM | ≥ 10% MoM |
| Average Revenue Per Account (ARPA) | MRR ÷ paying tenants | — | > $800/mo | > $1,200/mo |

### Retention Metrics

| Metric | Definition | Target |
|--------|-----------|--------|
| Gross Revenue Retention (GRR) | (MRR retained from existing customers, excl. expansion) / beginning MRR | ≥ 92% monthly |
| Net Revenue Retention (NRR) | (MRR retained + expansion − churn) / beginning MRR | ≥ 105% |
| Customer Churn Rate | Tenants cancelled ÷ total active tenants | < 3% monthly |
| Logo Retention Rate | % of tenants active at start of period still active at end | ≥ 95% monthly |

### Unit Economics

| Metric | Definition | Target (12M) |
|--------|-----------|--------------|
| Customer Acquisition Cost (CAC) | Total sales + marketing spend ÷ new customers | < $1,500 |
| Customer Lifetime Value (LTV) | ARPA ÷ Monthly Churn Rate | > $18,000 |
| LTV:CAC Ratio | LTV ÷ CAC | ≥ 3:1 |
| CAC Payback Period | CAC ÷ (ARPA × gross margin %) | < 12 months |
| Gross Margin | (MRR − COGS) ÷ MRR | ≥ 70% |

### Growth Metrics

| Metric | Definition | Target (12M) |
|--------|-----------|--------------|
| Active Paying Tenants | Tenants with active paid subscription | ≥ 150 |
| New Tenants / Month | Net new paid tenant activations | ≥ 20/month |
| Time to Value (TTV) | Signup → first meaningful action (lead created / opportunity added) | < 4 hours |
| Free-to-Paid Conversion Rate | % of free trial tenants converting to paid | ≥ 20% |
| Trial Duration (median) | Days from signup to conversion decision | < 14 days |

---

## Product KPIs

### Activation & Onboarding

| Metric | Definition | Target |
|--------|-----------|--------|
| Day-1 Activation Rate | % of new users who create ≥ 1 entity on signup day | ≥ 45% |
| 7-Day Activation Rate | % of new tenants completing "active setup" within 7 days (≥ 5 contacts OR ≥ 1 lead OR ≥ 1 opportunity) | ≥ 65% |
| Onboarding Completion Rate | % of tenants who: add users + configure pipeline + import/create ≥ 10 contacts | ≥ 50% |
| Invite Acceptance Rate | % of sent invitations accepted within 48 hours | ≥ 70% |

### Engagement

| Metric | Definition | Target |
|--------|-----------|--------|
| Daily Active Users (DAU) | Unique users with ≥ 1 API call per day | Grow ≥ 10% MoM |
| Weekly Active Users (WAU) | Unique users active ≥ 1 day/week | ≥ 75% of licensed seats |
| DAU/MAU Ratio (Stickiness) | Daily actives ÷ Monthly actives | ≥ 35% |
| Average Session Duration | Mean time from first to last API call per session | ≥ 9 minutes |
| Feature Breadth (Modules Active) | % of tenants using ≥ 4 of 7 modules | ≥ 55% by M6 |

### Core Feature Adoption

| Metric | Definition | Target |
|--------|-----------|--------|
| Leads Created / Tenant / Month | Volume signal for SDR workflow adoption | ≥ 50 |
| Opportunities Created / Tenant / Month | Pipeline health signal | ≥ 20 |
| Activities Logged / Rep / Day | Calls + emails + meetings logged | ≥ 5 |
| Tasks Completed On Time | % completed by or before due date | ≥ 78% |
| Pipeline Coverage Ratio | Total open pipeline ÷ monthly quota target | ≥ 3× |
| Dashboard Visits / Tenant / Week | Manager engagement signal | ≥ 3 unique users |

### Satisfaction

| Metric | Definition | Target |
|--------|-----------|--------|
| Net Promoter Score (NPS) | Quarterly in-app survey (0–10 recommend score) | ≥ 45 |
| Customer Satisfaction Score (CSAT) | Post-interaction survey | ≥ 4.3 / 5.0 |
| Support Ticket Volume | Tickets per 100 MAU per month | < 5 |
| Time to First Response | Support P1 issues | < 2 hours |
| Feature Request Fulfillment | % of Top 10 user requests shipped next 2 quarters | ≥ 70% |

---

## Technical KPIs

### Reliability

| Metric | Definition | Target |
|--------|-----------|--------|
| Platform Uptime | % availability rolling 30 days | ≥ 99.9% |
| P0 Incident Frequency | Service-down incidents per month | ≤ 1 |
| Mean Time to Recovery (MTTR) | Avg time to full recovery after incident | < 30 minutes |
| Mean Time Between Failures (MTBF) | Avg time between P0 incidents | > 30 days |
| Error Rate | 5XX responses ÷ total responses | < 0.1% |

### Performance

| Metric | Target |
|--------|--------|
| API p95 latency (reads) | < 300ms |
| API p95 latency (writes) | < 500ms |
| DB p99 query time | < 200ms |
| Frontend LCP | < 2,500ms |
| Frontend Lighthouse score | ≥ 85 |

### Engineering Health

| Metric | Definition | Target |
|--------|-----------|--------|
| Deployment Frequency | Production deploys per week | ≥ 2 |
| Change Failure Rate | Deployments causing P1/P0 incidents ÷ total deployments | < 5% |
| Lead Time for Changes | Commit merged → deployed to production | < 24 hours |
| Backend Test Coverage | Line coverage (Jest) | ≥ 75% |
| E2E Test Coverage | Critical path scenarios covered | 100% of 8 critical paths |
| Open P0 Bugs | Critical bugs in backlog | 0 |
| Open P1 Bugs | High-severity bugs in backlog | ≤ 3 |
| Security CVEs (critical/high) | Open CVEs in production dependencies | 0 critical, 0 high |
| Tech Debt Ratio | Sprint capacity spent on debt reduction | ≥ 20% |

---

## Launch Readiness Checklist

### Feature Completeness Gate

- [ ] Module 1 (IAM): Auth, RBAC, User/Team management fully functional
- [ ] Module 2 (Contact & Account): Full CRUD, activity timeline, notes, attachments
- [ ] Module 3 (Lead): Capture, assignment, qualification, conversion
- [ ] Module 4 (Pipeline): Kanban + list views, stage management, win/loss
- [ ] Module 5 (Activity): Tasks, call logging, meeting logging, in-app notifications
- [ ] Module 6 (Email): Manual email logging, BCC capture stub, templates
- [ ] Module 7 (Reports): Executive dashboard + 3 pre-built reports + CSV export
- [ ] Multi-tenant onboarding flow (self-service signup → first login) < 5 minutes

### Quality Gate

- [ ] Zero P0 bugs outstanding
- [ ] Backend coverage ≥ 75%
- [ ] E2E tests pass for all 8 critical paths (register, login, create lead, convert lead, create opportunity, close deal, run report, admin user provisioning)
- [ ] Load test at 200 concurrent users per tenant within NFR-PERF targets
- [ ] Security audit: OWASP Top 10 all addressed
- [ ] npm audit: zero critical/high CVEs

### Operational Gate

- [ ] Docker Compose production configuration validated
- [ ] GitHub Actions CI/CD pipeline operational (all stages pass)
- [ ] Health check endpoints responding correctly
- [ ] Database backup automated and restore tested
- [ ] Runbook written for: restart, backup restore, emergency rollback, user data export
- [ ] Error monitoring (Sentry) configured and tested
- [ ] External uptime monitoring configured with alerting

---

## Metric Review Cadence

| Frequency | Metrics | Owner | Format |
|-----------|---------|-------|--------|
| Daily | Error rate, uptime, active incidents, queue depth | Engineering on-call | Automated alert |
| Weekly | DAU/WAU, new tenants, activation rate, support tickets, open bugs | Product + Engineering | 15-min standup |
| Monthly | MRR/ARR, churn, NPS, CSAT, feature adoption, tech debt | Product + Engineering | Monthly review doc |
| Quarterly | ARR, NRR, LTV:CAC, roadmap delivery, win/loss | Leadership + Board | Quarterly business review |
