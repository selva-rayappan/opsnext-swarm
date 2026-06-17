# User Personas — OpsNext CRM

Six primary personas represent the full range of OpsNext CRM users. Each persona drives specific feature decisions and acceptance criteria.

---

## Persona 1: Alex Chen — Sales Development Representative (SDR)

**Age**: 24 | **Experience**: 18 months in sales | **Company**: 120-person B2B SaaS startup

> "I spend more time logging activities than actually selling. By the time I've updated the CRM, I've lost my momentum."

### Role Overview
Alex runs outbound prospecting sequences — cold email, LinkedIn, cold calls — to book discovery meetings for Account Executives. He manages 150–200 leads at any given time across various stages of outreach.

### Goals
- Hit 20 qualified meetings booked per month
- Keep lead status 100% current so the manager can see pipeline health
- Log every touchpoint in < 30 seconds — not 5 minutes
- Convert high-quality, warmed-up leads to AEs without losing context
- Get assigned the right leads, not duplicates of ones already contacted

### Pain Points (current state)
| Pain | Impact |
|------|--------|
| Manual call logging takes 4–5 minutes per call | Loses 30+ min/day; skips logging when busy |
| No clear "who to call today" view | Relies on spreadsheet task list outside the CRM |
| Duplicate leads get assigned — wastes time | Calls already-contacted prospects; embarrassing |
| No visibility into what happens post-handoff | Can't learn which lead quality → closed deals |
| Re-entering data when converting a lead | Time sink; introduces errors |

### Key Jobs to Be Done
1. **Log a call outcome** in under 30 seconds after hanging up
2. **See "my leads needing follow-up today"** without building a manual filter
3. **Capture a new lead** from a LinkedIn message in under 2 minutes
4. **Convert a lead to opportunity** with pre-filled data from the lead record
5. **Reassign a lead** to a teammate with full history attached

### Day-in-the-Life with OpsNext CRM
- 8:00 AM: Opens "My Leads" view filtered to `status: contacted, last activity > 2 days`. Works the follow-up list.
- 9:30 AM: Completes a call. Logs it with outcome + next step in 20 seconds from the lead's quick-log sidebar.
- 11:00 AM: Imports 30 new leads from a LinkedIn export CSV. Assigns source: "LinkedIn Outreach".
- 2:00 PM: Qualifies a lead after a good discovery call. Hits "Convert" — Contact, Account, and Opportunity pre-filled. AE notified.

### OpsNext Requirements Driven by Alex
- Quick-log sidebar (call/email/note) accessible from any record without page navigation
- "Needs follow-up" smart view: status = contacted, no activity in X days
- Duplicate detection on email before save
- Lead conversion pre-fills Contact/Account/Opportunity from Lead data
- CSV import with field mapping UI

### Success Criteria
- Call logging time: < 30 seconds
- Follow-up coverage: 90%+ of contacted leads actioned within 48 hours
- Zero duplicate lead assignments per week

---

## Persona 2: Jordan Rivera — Account Executive (AE)

**Age**: 29 | **Experience**: 4 years in B2B sales | **Company**: 200-person SaaS company

> "I need to know exactly where every deal stands, what the next step is, and which deals are at risk — all before my 9 AM pipeline call."

### Role Overview
Jordan owns a book of 30–50 active opportunities from qualification through close. She manages multi-stakeholder deals with $15K–$80K ACV, typical sales cycle 45–90 days.

### Goals
- Close $150K+ new ARR per quarter
- Maintain a 3× pipeline coverage ratio
- Keep every deal updated with a next step and close date
- Give the manager accurate, defensible forecast data

### Pain Points
| Pain | Impact |
|------|--------|
| No deal age indicator — old deals look the same as new | Misses at-risk deals until review |
| Proposal/notes in email thread, not CRM | Context lost if she's sick; manager has no visibility |
| Stage changes not tied to a required next action | Pipeline goes stale without pressure |
| Amount changes have no history | Can't explain to manager why Q forecast changed |

### Key Jobs to Be Done
1. **Morning pipeline review** — see all open deals, stage, amount, age, next step
2. **Update a deal after a call** — add note + update stage + set next task in one flow
3. **Attach a proposal PDF** to an opportunity
4. **Run "closing this month"** filter before forecast call
5. **Create an opportunity from a converted lead** — with account and contact pre-linked

### Day-in-the-Life with OpsNext CRM
- 8:50 AM: Opens pipeline Kanban. Deals with no activity in > 7 days are flagged amber. Prioritizes her day.
- 10:30 AM: Finishes a demo. Opens the opp, adds call note, moves stage to "Proposal," creates a follow-up task.
- 2:00 PM: Attaches proposal PDF to the opportunity record.
- 4:30 PM: Filters pipeline to close date = this month. Reviews and updates amounts before Friday forecast call.

### OpsNext Requirements Driven by Jordan
- Deal age indicator (days since last stage change)
- Kanban board with drag-to-change-stage
- Inline note + task creation on opportunity record
- Stage change with optional "reason" and required next-step task
- Amount change history tracked in activity timeline
- File attachment (PDF, DOCX) on opportunity

### Success Criteria
- Forecast accuracy: ±10% of actual quarterly close
- 100% of open opportunities have a next-step task set
- Deal stage hygiene: no opportunity stuck in same stage > 21 days without a note

---

## Persona 3: Morgan Taylor — Sales Manager

**Age**: 36 | **Experience**: 8 years sales, 3 years managing | **Team**: 6 AEs + 3 SDRs

> "I shouldn't have to chase my team for pipeline updates. The CRM should tell me everything I need to know."

### Role Overview
Morgan runs a 9-person inside sales team. She holds weekly 1:1 pipeline reviews, submits a Friday forecast to the VP, and coaches reps on deal strategy.

### Goals
- Hit team quota: $500K new ARR per quarter
- Identify underperforming reps 3+ weeks before end of quarter
- Produce an accurate weekly forecast in under 30 minutes
- Coach reps on specific stuck deals with full context

### Pain Points
| Pain | Impact |
|------|--------|
| Pipeline is fragmented per rep — no team view | Manual aggregation every Friday |
| Activity log is sparse — reps skip it | Can't tell if a deal is stuck or just not updated |
| Forecast conversation is about opinions, not data | Inaccurate forecasts miss by 30%+ |
| No easy way to flag a deal for coaching | Sticky notes and Slack messages — not in the CRM |

### Key Jobs to Be Done
1. **Team pipeline view** — all reps' deals, filterable by rep/stage/amount
2. **Activity dashboard** — calls, emails, meetings per rep for last 7/30 days
3. **Flag a deal for pipeline review** with a manager note
4. **Reassign a book of business** when a rep leaves
5. **Download Friday forecast report** in 2 clicks

### Day-in-the-Life with OpsNext CRM
- Monday: Reviews team activity dashboard. Two reps show < 5 activities last week — schedules coaching sessions.
- Wednesday 1:1: Opens Alex's pipeline filtered to Alex's deals. Drills into a stale opportunity. Adds a manager note: "Needs exec sponsor — action by Friday."
- Friday 3 PM: Opens Forecast report. Filters to current quarter. Downloads CSV for board deck. 25 minutes total.
- Friday 4 PM: SDR resigns. Bulk reassigns their 80 leads to remaining SDRs via the team reassignment tool.

### OpsNext Requirements Driven by Morgan
- Team pipeline view with rep/stage/amount/age columns
- Activity dashboard: call/email/meeting count per rep per period
- Manager note field on opportunity (distinct from rep notes)
- Bulk lead/opportunity reassignment with audit trail
- Forecast report with filters: rep, pipeline, close date range, stage

### Success Criteria
- Friday forecast prep time: < 30 minutes
- Team activity dashboard gives clear health signal (red/amber/green per rep)
- Zero pipeline surprises at quarter end

---

## Persona 4: Casey Williams — CRM Administrator / RevOps

**Age**: 32 | **Experience**: 5 years in RevOps | **Company**: 250-person B2B company

> "Every time I need to add a field or change a stage name, I have to file a ticket with engineering. That can't be the answer."

### Role Overview
Casey is the system owner for OpsNext CRM. She onboards new users, configures pipelines, manages roles and permissions, maintains data quality, and owns the CRM roadmap internally.

### Goals
- Configure the CRM to exactly match the company's sales process — no compromises
- Grant the right level of access to the right people without creating security gaps
- Keep data clean: deduplication, archival, mandatory fields at key stages
- Enable integrations with Slack, email provider, and reporting tools

### Pain Points
| Pain | Impact |
|------|--------|
| No audit log — can't investigate "who changed this?" | Compliance risk; can't debug data issues |
| Permissions are all-or-nothing per role | Can't grant "view leads but not edit accounts" |
| Configuring custom fields requires a developer | 2-week lead time for every field change |
| No bulk data tools | Cleaning up 500 stale leads takes days |

### Key Jobs to Be Done
1. **Create a custom dropdown field** on Lead without touching code
2. **Define a role** with read-only access to opportunities but full access to contacts
3. **View audit log** for a specific record: "who changed this lead's owner on Tuesday?"
4. **Export all contacts** belonging to a specific account to CSV
5. **Set a field as required** when moving an opportunity to "Proposal" stage
6. **Bulk archive** leads with no activity in > 180 days

### Day-in-the-Life with OpsNext CRM
- Monday: New SDR starts. Provisions user, assigns SDR role, sends invite. Done in 4 minutes.
- Tuesday: Sales Director requests a "Decision Maker Confirmed" checkbox on opportunities. Casey adds it in the admin panel, marks it required at "Negotiation" stage. Live in 5 minutes.
- Thursday: Data audit. Bulk-selects 200 leads with no activity in 6 months. Archives them. Exports a CSV for review.
- Friday: Compliance request. Pulls audit log for a specific contact record showing all changes in last 90 days.

### OpsNext Requirements Driven by Casey
- Custom field builder: text, number, date, dropdown, checkbox, multi-select
- Field-level stage requirements (required at specific pipeline stages)
- Granular permission builder per role
- Audit log queryable by entity, user, date range
- Bulk operations: archive, reassign, export
- User provisioning: invite → assign role → auto-email → one flow

### Success Criteria
- Any configuration change: completable without engineering involvement
- User provisioning and deprovisioning: < 5 minutes
- Audit log: 100% of write operations captured
- Role changes: effective immediately

---

## Persona 5: Riley Park — VP of Sales / CRO

**Age**: 44 | **Experience**: 15 years in sales leadership | **Company**: 400-person company, $30M ARR

> "I present to the board every quarter. If my forecast is off by more than 10%, I have a credibility problem. Right now, I have a credibility problem."

### Role Overview
Riley owns the revenue number. She manages a team of 4 managers, 30 reps. Her week is forecast calls, pipeline reviews, and board prep. She spends minimal time in the CRM — she needs the CRM to surface the right data fast.

### Goals
- Board-level forecast accuracy: ±5% of actual quarterly close
- Clear view of pipeline health across all teams and pipelines
- Win/loss visibility to drive hiring, coaching, and product feedback
- Reduce board prep time from 4 hours to 45 minutes

### Pain Points
| Pain | Impact |
|------|--------|
| Forecast is AE optimism, not data | Missed two quarters in a row |
| No visibility into which verticals/segments win | Can't prioritize ICP intelligently |
| Win/loss analysis is anecdotal | Can't drive product roadmap from CRM data |
| Dashboard data is stale — last refreshed manually | Can't trust the numbers on Friday afternoon |

### Key Jobs to Be Done
1. **Executive dashboard** — pipeline by stage, weighted pipeline, win rate, avg deal size, team activity
2. **Drill into a team or rep** without leaving the dashboard
3. **Win/loss analysis** filtered by industry, deal size, rep, quarter
4. **Set quota per rep** and view quota attainment in real time
5. **Export board-ready pipeline report** in 2 clicks

### OpsNext Requirements Driven by Riley
- Executive Dashboard: pre-built, always-on, auto-refreshing
- Win/Loss report with multi-dimensional filters
- Quota tracking per user (target vs. attained)
- Team performance view: rep leaderboard by revenue, activity, win rate
- One-click export of any report to CSV or printable format

### Success Criteria
- Forecast accuracy: ±5% of actual quarterly revenue
- Board-ready pipeline report: generated in < 5 minutes
- Win/loss data available without a BI tool or spreadsheet

---

## Persona 6: Sam Okafor — Customer Success Manager (CSM)

**Age**: 27 | **Experience**: 3 years in customer success | **Company**: 180-person SaaS company

> "By the time I realize an account is at risk, they've already decided not to renew. I need to see it coming."

### Role Overview
Sam manages 35 post-sale accounts in the $10K–$60K ARR range. Her work is check-in calls, QBRs, issue escalation, and identifying upsell opportunities.

### Goals
- Maintain NPS > 50 across her book of business
- Reduce churn to < 5% annually
- Surface upsell opportunities to AEs proactively
- Never miss a renewal date

### Pain Points
| Pain | Impact |
|------|--------|
| Customer history is in email — not the CRM | Context lost between calls; looks unprepared |
| No at-risk flag on accounts | Reactive churn: finds out too late |
| Renewal dates in a spreadsheet | Misses renewals; damaged relationships |
| No handoff from AE on closed deals | No context on why they bought, what was promised |

### Key Jobs to Be Done
1. **View complete account timeline** — all calls, emails, notes, tasks chronologically
2. **Log a QBR call** with outcomes, action items, and sentiment score
3. **Flag an account as at-risk** with a reason and follow-up date
4. **Set a renewal reminder** 90/60/30 days before renewal date
5. **Create an upsell opportunity** linked to the existing account and pass it to an AE

### OpsNext Requirements Driven by Sam
- Account activity timeline (all interactions, all team members)
- Account health field (custom dropdown: Healthy / At Risk / Critical)
- Renewal date field on Account with reminder workflow
- Upsell opportunity creation linked to existing Account + Contact
- Ownership visibility: see AE owner + CSM owner on account record

### Success Criteria
- 100% of customer touchpoints logged within 24 hours
- At-risk accounts reviewed weekly
- Zero renewal surprises — all dates visible 90 days out

---

## Persona Summary Matrix

| Dimension | Alex (SDR) | Jordan (AE) | Morgan (Mgr) | Casey (Admin) | Riley (VP) | Sam (CSM) |
|-----------|:----------:|:-----------:|:------------:|:-------------:|:----------:|:---------:|
| **Primary module** | Leads | Opportunities | Reports | Settings | Dashboards | Accounts |
| **Sessions/day** | 8–12 | 5–8 | 3–5 | 1–2 | 1–2 | 3–5 |
| **Data entry volume** | Very High | High | Low | Low | None | Medium |
| **Report consumption** | Low | Low | High | Medium | Very High | Medium |
| **RBAC level** | Standard Rep | Standard Rep | Manager | Admin | Executive Viewer | Standard Rep |
| **Mobile use** | Medium | High | Medium | Low | High | Medium |
| **Key metric** | Meetings booked | ARR closed | Team quota | System uptime | Forecast accuracy | Churn rate |
| **Biggest fear** | Missing follow-ups | Forecast surprise | Missing quota | Data breach | Bad board meeting | Surprise churn |
