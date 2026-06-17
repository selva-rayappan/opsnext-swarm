# Functional Requirements — OpsNext CRM

Functional requirements define the specific behaviors the system must exhibit. Each requirement is independently verifiable.

Convention: `FR-[MODULE]-[NNN]`

---

## Module 1: Identity & Access Management

### Authentication

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-IAM-001 | The system shall allow a user to register with email and password. Password policy: minimum 8 characters, at least one uppercase letter, one digit, one special character. | P0 |
| FR-IAM-002 | On successful login with valid credentials, the system shall return a JWT access token (RS256, 15-minute expiry) and set a HttpOnly refresh token cookie (30-day expiry). | P0 |
| FR-IAM-003 | The system shall validate the JWT on every request to a protected endpoint. Expired or invalid tokens shall return 401. | P0 |
| FR-IAM-004 | The system shall support refresh token rotation: exchanging a valid refresh token returns a new access token AND a new refresh token. The old refresh token is immediately invalidated. | P0 |
| FR-IAM-005 | The system shall allow a user to log out. Logout revokes the current refresh token; subsequent use of the revoked token returns 401. | P0 |
| FR-IAM-006 | The system shall provide a password reset flow: request via email → time-limited token (1 hour) → reset form → new password saved. | P0 |
| FR-IAM-007 | The system shall rate-limit login attempts: 5 failures per 15 minutes per IP returns 429 with a `retryAfter` field. | P0 |
| FR-IAM-008 | The system shall allow tenant admins to invite new users via email. Invitation links expire in 48 hours. Accepting the invitation activates the user account. | P0 |
| FR-IAM-009 | The system shall return the authenticated user's profile, tenantId, and effective permissions on `GET /api/v1/auth/me`. | P0 |

### User Management

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-IAM-010 | Admins shall be able to list all users in their tenant (filterable by role, status, team). | P0 |
| FR-IAM-011 | Admins shall be able to update any user's profile fields: firstName, lastName, phone, roleId, teamIds. | P0 |
| FR-IAM-012 | Admins shall be able to deactivate a user. Deactivated users cannot log in. Their data and history are preserved. Active sessions are invalidated within 60 seconds. | P0 |
| FR-IAM-013 | Admins shall be able to reactivate a deactivated user. | P1 |
| FR-IAM-014 | Admins shall be able to bulk-reassign all records owned by a specific user to another user. | P1 |
| FR-IAM-015 | Users shall be able to update their own profile: name, password, timezone, locale, avatar. | P0 |

### RBAC

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-IAM-016 | The system shall ship with 5 system roles: `super_admin`, `admin`, `manager`, `sales_rep`, `viewer`. System roles cannot be deleted or modified by tenants. | P0 |
| FR-IAM-017 | Admins shall be able to create custom roles with a selected set of permissions. | P1 |
| FR-IAM-018 | Permissions shall be expressed as `resource:action` pairs. Resources: contacts, accounts, leads, opportunities, activities, tasks, emails, reports, settings, users, teams, audit_logs, pipelines. Actions: read, create, update, delete, assign, export. | P0 |
| FR-IAM-019 | Every protected API endpoint shall be guarded by a permission check. Requests without the required permission return 403 with `{ error: "forbidden", requiredPermission: "X:Y" }`. | P0 |
| FR-IAM-020 | Admins shall be able to view the resolved effective permissions for any user. | P1 |

### Audit Logs

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-IAM-021 | Every create, update, and delete operation on a business entity shall produce an audit log entry. | P0 |
| FR-IAM-022 | Audit log entries shall contain: id, tenantId, userId, entityType, entityId, action (created/updated/deleted/restored), before (JSON snapshot), after (JSON snapshot), ipAddress, userAgent, createdAt. | P0 |
| FR-IAM-023 | The audit log shall be queryable by tenantId, userId, entityType, entityId, action, and date range. | P0 |
| FR-IAM-024 | No update or delete endpoints shall exist for audit log records. The table is append-only. | P0 |

---

## Module 2: Contact & Account Management

### Accounts

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-CAM-001 | Users with `accounts:create` permission shall be able to create an Account. Required field: name. | P0 |
| FR-CAM-002 | Account fields: name, industry, website, phone, email, address (street/city/state/country/postal), employees, annualRevenue, currency, description, ownerId, status (active/archived), customFields. | P0 |
| FR-CAM-003 | Account list shall support: full-text search (name, website), filter (ownerId, industry, status), sort (name, createdAt, updatedAt), pagination (max 100/page). | P0 |
| FR-CAM-004 | Users shall be able to view an Account's activity timeline: all linked calls, emails, meetings, notes, tasks, and stage changes in reverse-chronological order. | P0 |
| FR-CAM-005 | Admins shall be able to transfer Account ownership to another user. Transfer is logged in the audit log. | P1 |
| FR-CAM-006 | Accounts shall support tenant-defined custom fields (see FR-ADMIN-001). | P1 |

### Contacts

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-CAM-007 | Users with `contacts:create` permission shall be able to create a Contact. Required fields: firstName, lastName. | P0 |
| FR-CAM-008 | Contact fields: firstName, lastName, email, phone, mobilePhone, jobTitle, department, linkedinUrl, primaryAccountId, ownerId, status (active/archived), customFields. | P0 |
| FR-CAM-009 | Contact list shall support: full-text search (name, email, company, jobTitle), filter (ownerId, accountId, status), sort, pagination. | P0 |
| FR-CAM-010 | On Contact creation, if an existing Contact with the same email exists in the tenant, the system shall show a duplicate warning (not block). | P0 |
| FR-CAM-011 | Admins shall be able to merge two Contact records. The surviving record inherits all linked activities, emails, leads, opportunities, and notes from the merged record. The merged record is soft-deleted. | P1 |
| FR-CAM-012 | A Contact shall be associatable with multiple Accounts via a ContactAccount join with `isPrimary` and `role` fields. | P1 |
| FR-CAM-013 | Contacts shall support directional relationship links between each other (e.g., "Reports To", "Decision Maker For") with tenant-defined relationship types. | P2 |

### Notes & Attachments

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-CAM-014 | Users shall be able to add markdown-formatted notes to any entity (Contact, Account, Lead, Opportunity). Notes are timestamped and attributed to the author. | P0 |
| FR-CAM-015 | Note authors and admins shall be able to edit or delete their own notes. Deletions are soft-deleted. | P1 |
| FR-CAM-016 | Users shall be able to upload file attachments (max 25MB) to Contacts, Accounts, Leads, and Opportunities. Supported types: PDF, DOCX, XLSX, PNG, JPG, CSV. | P1 |
| FR-CAM-017 | Attachment downloads shall use pre-signed URLs (1-hour expiry) from S3-compatible storage. Direct public URL access returns 403. | P1 |

---

## Module 3: Lead Management

### Lead Capture

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-LM-001 | Users with `leads:create` permission shall be able to manually create a Lead. Required field: firstName + lastName OR email. | P0 |
| FR-LM-002 | Lead fields: firstName, lastName, email, phone, company, jobTitle, source (web_form, cold_outreach, referral, event, inbound, import, api, other), status, qualificationNotes, ownerId, teamId, customFields. | P0 |
| FR-LM-003 | The system shall expose a public `POST /api/v1/leads/ingest` endpoint (authenticated via API key) for web form and external tool integrations. Rate-limited to 100 req/min per key. | P0 |
| FR-LM-004 | Users shall be able to import Leads via CSV upload. The import UI shall present a field-mapping step, a preview of the first 10 rows, and an error report after import. | P1 |
| FR-LM-005 | Lead list shall support: search (name, email, company), filter (status, ownerId, teamId, source, date range), sort (createdAt, updatedAt, lastActivity), pagination. | P0 |

### Lead Assignment

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-LM-006 | Managers/admins shall be able to manually assign any Lead to any user in the tenant. | P0 |
| FR-LM-007 | The system shall support round-robin lead assignment to a configured team. | P1 |
| FR-LM-008 | Lead assignment changes shall be logged in the Lead's activity timeline with previous owner, new owner, and actor. | P0 |
| FR-LM-009 | A dedicated "Unassigned Leads" view shall be visible to managers and admins. | P1 |

### Lead Lifecycle

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-LM-010 | Lead status transitions: `new` → `contacted` → `qualified` → `converted` or `disqualified`. The system shall allow any forward transition. Backward transitions are allowed (except from `converted`). | P0 |
| FR-LM-011 | Transitioning a Lead to `disqualified` requires selecting a disqualification reason from a tenant-configured list. | P0 |
| FR-LM-012 | Transitioning a Lead to `converted` opens a conversion dialog that allows the user to optionally create: a Contact, an Account, and/or an Opportunity. Pre-populates fields from the Lead. | P0 |
| FR-LM-013 | After conversion, the Lead status is set to `converted` and is read-only. The conversion creates links to the resulting Contact/Account/Opportunity IDs. | P0 |

---

## Module 4: Opportunity & Pipeline Management

### Pipeline Configuration

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-PM-001 | Admins shall be able to create multiple Pipelines per tenant. Each Pipeline has a name and an ordered list of Stages. | P0 |
| FR-PM-002 | Stage fields: name, ordinal (display order), probability (0–100%), isWon (boolean), isLost (boolean), description. | P0 |
| FR-PM-003 | Admins shall be able to reorder Pipeline Stages. Reorder is transactional — all ordinal values update atomically. | P1 |
| FR-PM-004 | Deleting a Stage is blocked if any Opportunities exist in that Stage. Admin must migrate existing Opportunities first. | P0 |
| FR-PM-005 | One Pipeline per tenant shall be designated as the default (used when creating Opportunities without specifying a Pipeline). | P0 |

### Opportunities

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-PM-006 | Users with `opportunities:create` shall be able to create an Opportunity. Required field: name. | P0 |
| FR-PM-007 | Opportunity fields: name, accountId, primaryContactId, pipelineId, stageId, amount, currency, expectedCloseDate, probability (auto-set from Stage, overridable), ownerId, description, lostReason, customFields. | P0 |
| FR-PM-008 | The Kanban view shall display Opportunities as cards in stage columns. Cards shall show: name, amount, account name, owner avatar, days in stage. | P0 |
| FR-PM-009 | Users shall be able to drag an Opportunity card between stage columns. Stage change is immediately persisted and logged in the activity timeline. | P0 |
| FR-PM-010 | Opportunities with `expectedCloseDate` in the past and `status: open` shall be visually flagged as overdue. | P0 |
| FR-PM-011 | Closing an Opportunity as Lost requires selecting a `lostReason` from a tenant-configured dropdown. | P0 |
| FR-PM-012 | Amount changes on an Opportunity shall be recorded in the activity timeline (previous amount → new amount). | P1 |

### Pipeline Analytics

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-PM-013 | The Pipeline view shall display: total pipeline value (sum of all open opportunity amounts), weighted pipeline (sum of amount × probability), count by stage. | P0 |
| FR-PM-014 | Win Rate shall be calculated as: closed-won ÷ (closed-won + closed-lost) for a configurable time window. | P1 |

---

## Module 5: Activity & Task Management

### Tasks

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-ATM-001 | Users shall be able to create Tasks. Fields: title (required), description, dueDate, priority (low/medium/high/urgent), assigneeId, entityType, entityId. | P0 |
| FR-ATM-002 | Task status: `open`, `in_progress`, `completed`, `cancelled`. Users update status manually. | P0 |
| FR-ATM-003 | "My Tasks" view: tasks assigned to the current user, sortable/filterable by status, priority, due date. | P0 |
| FR-ATM-004 | Tasks past their `dueDate` with status not `completed` or `cancelled` shall be flagged as overdue (visual indicator + filtered view). | P0 |
| FR-ATM-005 | Completing a task where the assignee ≠ creator triggers an in-app notification to the creator. | P1 |
| FR-ATM-006 | Recurring tasks: tasks can be configured with a recurrence rule (daily/weekly/monthly). On completion, the next instance is automatically created. | P2 |

### Activity Logging

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-ATM-007 | Users shall be able to log a Call. Fields: subject, direction (inbound/outbound), duration (minutes), outcome (connected/no_answer/left_voicemail/other), notes, callDate, contactId, entityType, entityId. | P0 |
| FR-ATM-008 | Users shall be able to log a Meeting. Fields: subject, meetingDate, duration, attendees (name + email, internal or external), notes, entityType, entityId. | P0 |
| FR-ATM-009 | All logged activities (calls, meetings, notes, emails, stage changes) shall appear in the entity's activity timeline sorted by `occurredAt` descending. | P0 |

### Notifications & Reminders

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-ATM-010 | Users shall be able to set a reminder (date + time) on any Task, Lead, or Opportunity. | P1 |
| FR-ATM-011 | Reminders shall trigger in-app notifications and optionally email notifications at the configured time. | P1 |
| FR-ATM-012 | In-app notification bell shall display the unread notification count persistently in the navigation bar. | P0 |
| FR-ATM-013 | Notification types: task_due, task_overdue, task_completed (notify creator), lead_assigned, opp_stage_changed, deal_closed_won, deal_closed_lost, mention (@user in note). | P1 |
| FR-ATM-014 | Users shall be able to configure per-notification-type delivery preferences: in_app_only, email_only, both, off. | P2 |

---

## Module 6: Email & Communication

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-EC-001 | Users shall be able to manually log an email. Fields: direction, fromAddress, toAddresses, ccAddresses, subject, body, sentAt, contactId, accountId, leadId, opportunityId. | P0 |
| FR-EC-002 | Logged emails shall appear in the activity timeline of all linked entities. | P0 |
| FR-EC-003 | The system shall provide a unique per-user BCC address (format: `{userId}@ingest.{tenantSlug}.opsnext.io`) that auto-logs inbound emails when BCC'd. | P1 |
| FR-EC-004 | Full-text search across email subject and body within a tenant. | P1 |
| FR-EC-005 | Admins shall be able to create, edit, and delete Email Templates. Template fields: name, subject (with merge fields), body (with merge fields), category (outreach/follow_up/proposal/closing/other), isShared. | P1 |
| FR-EC-006 | Merge fields in templates: `{{contact.firstName}}`, `{{contact.lastName}}`, `{{account.name}}`, `{{rep.firstName}}`, `{{opportunity.name}}`. | P1 |
| FR-EC-007 | When logging a manual email, users can select a template to pre-fill subject and body, then personalize before saving. | P2 |

---

## Module 7: Reporting & Dashboards

### Dashboards

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-RPT-001 | The system shall provide 3 pre-built dashboards: Executive Overview, Sales Performance, Lead Analytics. | P0 |
| FR-RPT-002 | Each dashboard supports a date range selector: this week, this month, this quarter, last quarter, custom range. | P0 |
| FR-RPT-003 | Managers and admins shall be able to filter any dashboard by user or team. | P0 |
| FR-RPT-004 | Dashboards shall auto-refresh every 5 minutes. Manual refresh button available. | P1 |

### Reports

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-RPT-005 | Pre-built reports: Pipeline by Stage, Win/Loss Analysis, Lead Source Performance, Rep Activity Summary, Revenue Forecast. | P0 |
| FR-RPT-006 | All reports shall be exportable to CSV. | P0 |
| FR-RPT-007 | Reports shall support parameterization: date range, ownerId filter, teamId filter, pipelineId filter. | P0 |
| FR-RPT-008 | Admins/managers shall be able to schedule any report for weekly or monthly email delivery to a configured recipient list. | P2 |

### KPI Widgets

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-RPT-009 | KPI widgets (Executive): Total Pipeline Value, Weighted Pipeline, Win Rate %, Avg Deal Size, Avg Sales Cycle (days), New Leads, Conversion Rate. | P0 |
| FR-RPT-010 | KPI widgets (Activity): Calls Logged, Emails Logged, Meetings Held, Tasks Completed — per rep/team/period. | P1 |
| FR-RPT-011 | Each KPI widget shall display: current value, period-over-period change (absolute + %), trend indicator (up/down/flat). | P0 |

---

## Module 8: Administration (Cross-Cutting)

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-ADMIN-001 | Admins shall be able to define custom fields on: Contact, Account, Lead, Opportunity. Field types: text (single-line), textarea (multi-line), number, date, dropdown (single-select), multi-select, checkbox, URL. | P1 |
| FR-ADMIN-002 | Admins shall be able to mark a custom field as required at a specific Pipeline Stage (e.g., "Budget" required when moving to "Negotiation"). | P1 |
| FR-ADMIN-003 | Admins shall be able to configure tenant-level settings: name, timezone, locale, base currency, logo. | P0 |
| FR-ADMIN-004 | Admins shall be able to configure the list of Lead disqualification reasons and Opportunity loss reasons. | P0 |
| FR-ADMIN-005 | Admins shall be able to configure webhook endpoints: URL, secret (for HMAC signature), event subscriptions. | P1 |
