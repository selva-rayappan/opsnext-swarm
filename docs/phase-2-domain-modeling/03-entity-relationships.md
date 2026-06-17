# Entity Relationships — OpsNext CRM

---

## Relationship Summary Table

| Entity A | Relationship | Entity B | Cardinality | Ownership |
|----------|-------------|----------|-------------|-----------|
| Tenant | has many | Users | 1:N | Tenant owns Users |
| Tenant | has many | Roles | 1:N | Tenant owns Roles |
| Tenant | has many | Teams | 1:N | Tenant owns Teams |
| Tenant | has many | Pipelines | 1:N | Tenant owns Pipelines |
| Tenant | has many | Accounts | 1:N | Tenant owns Accounts |
| Tenant | has many | Contacts | 1:N | Tenant owns Contacts |
| Tenant | has many | Leads | 1:N | Tenant owns Leads |
| User | belongs to | Role | N:1 | User references Role |
| User | belongs to many | Teams | M:N via TeamMember | |
| Team | has one | Lead (team lead) | N:1 | Team references User |
| Account | has many | Contacts | 1:N via ContactAccount | Account can have many Contacts |
| Contact | belongs to many | Accounts | M:N via ContactAccount | Contact has a primary Account |
| Contact | has many | ContactRelationships | 1:N | Directional |
| Account | has many | Leads | 1:N | Lead has one Account (optional) |
| Account | has many | Opportunities | 1:N | Opportunity belongs to Account (optional) |
| Contact | has many | Leads | 1:N | Lead has one Contact (optional) |
| Contact | has many | Opportunities | 1:N | Opportunity has one primary Contact |
| Lead | converts to | Contact | 1:0..1 | On conversion |
| Lead | converts to | Account | 1:0..1 | On conversion |
| Lead | converts to | Opportunity | 1:0..1 | On conversion |
| Pipeline | has many | Stages | 1:N | Pipeline owns Stages (ordered) |
| Opportunity | belongs to | Pipeline | N:1 | |
| Opportunity | belongs to | Stage | N:1 | Stage must be in Opportunity's Pipeline |
| Activity | belongs to | Contact OR Account OR Lead OR Opportunity | N:1 (polymorphic) | |
| Task | belongs to | Contact OR Account OR Lead OR Opportunity | N:1 (polymorphic, optional) | |
| Email | belongs to | Contact OR Account OR Lead OR Opportunity | N:1..N (one primary, multi-link) | |
| Note | belongs to | Contact OR Account OR Lead OR Opportunity | N:1 (polymorphic) | |
| Attachment | belongs to | Contact OR Account OR Lead OR Opportunity | N:1 (polymorphic) | |
| AuditLog | references | any entity | N:1 (string key polymorphic) | |
| Notification | belongs to | User | N:1 | |

---

## Aggregate Roots

In DDD, aggregate roots define transaction boundaries. Changes to a cluster of entities go through the root.

| Aggregate Root | Owned Entities | Rationale |
|----------------|---------------|-----------|
| **Tenant** | Tenant settings, Plan | Tenant is the ultimate consistency boundary |
| **User** | RefreshTokens | Token lifecycle is internal to User |
| **Role** | Permission list (embedded array) | Permissions are value objects on Role |
| **Account** | ContactAccount links | Account manages its own contact associations |
| **Contact** | ContactRelationships | Contact manages its own relationship links |
| **Lead** | Conversion references | Lead controls conversion — one-way, irreversible |
| **Pipeline** | Stages | Stage ordinals must be consistent within a Pipeline |
| **Opportunity** | (none — references other aggregates by ID) | Opportunity has no owned children; links by ID |
| **Task** | (none) | Simple entity |
| **Email** | Attachment references (by ID) | Email does not own Attachments |

---

## Detailed Relationship Diagrams by Context

### Tenant Management Context

```
Tenant (1)
  ├── (N) Users
  ├── (N) Roles
  ├── (N) Teams
  ├── (N) Pipelines
  └── (N) [all tenant-scoped entities]
```

---

### IAM Context

```
User (N) ──── belongs to ──── (1) Role
  │
  └── (M:N) via TeamMember ──── Team
              └── teamLead (N:1) ──── User
```

Constraints:
- A User has exactly 1 Role at any time. Role changes are not versioned in v1.
- A User can be in 0..N Teams.
- A Team has 0..1 Team Lead (must be a member of the team).

---

### Contact & Account Context

```
Account (1) ──── ContactAccount (M:N) ──── Contact
    │                    └── isPrimary: boolean
    │                    └── role: string?
    │
    ├── (N) Leads (via Lead.accountId)
    ├── (N) Opportunities (via Opportunity.accountId)
    ├── (N) Activities (polymorphic)
    ├── (N) Notes (polymorphic)
    └── (N) Attachments (polymorphic)

Contact (1)
    ├── (N) ContactRelationships (from)
    ├── (N) ContactRelationships (to)
    ├── (N) Leads (via Lead.contactId — optional)
    ├── (N) Opportunities (via Opportunity.primaryContactId)
    ├── (N) Activities (polymorphic)
    ├── (N) Notes (polymorphic)
    ├── (N) Attachments (polymorphic)
    └── (N) Emails (polymorphic)
```

Constraints:
- A Contact must have at most one primary Account (`isPrimary: true`) via ContactAccount.
- Deleting an Account soft-deletes the Account; Contacts are NOT deleted (they may be independent).
- Deleting a Contact soft-deletes the Contact; ContactAccount entries are removed.

---

### Lead Context

```
Lead (1)
    ├── (0..1) Contact [convertedContactId] — created on conversion
    ├── (0..1) Account [convertedAccountId] — created on conversion
    ├── (0..1) Opportunity [convertedOpportunityId] — created on conversion
    ├── (N) Activities (polymorphic)
    ├── (N) Notes (polymorphic)
    └── (N) Attachments (polymorphic)
```

Constraints:
- `convertedContactId`, `convertedAccountId`, `convertedOpportunityId` are all nullable; conversion is partial (user may choose to create only some of these).
- Once `convertedAt` is set, the Lead is permanently in `converted` status — no further state transitions.

---

### Sales Pipeline Context

```
Pipeline (1)
    └── (N, ordered) Stages

Opportunity (N) ──── belongs to ──── (1) Pipeline
    └── belongs to ──── (1) Stage [must be in same Pipeline]
    └── (0..1) Account
    └── (0..1) Contact (primaryContact)
    └── (N) Activities (polymorphic)
    └── (N) Notes (polymorphic)
    └── (N) Attachments (polymorphic)
```

Constraints:
- Deleting a Pipeline is blocked if any open Opportunities exist in it.
- Moving an Opportunity to a different Pipeline requires selecting a new Stage in the target Pipeline.
- Stage `ordinal` must be unique per Pipeline. Reorder is transactional.

---

### Activity Context (Polymorphic)

Activities, Notes, Attachments, Emails, and Tasks use a polymorphic association pattern:

```
entityType: 'contact' | 'account' | 'lead' | 'opportunity'
entityId:   UUID (references the relevant table)
```

This means there are no foreign key constraints at the DB level for the polymorphic association — referential integrity is enforced at the application layer.

For query efficiency, a composite index on `(tenantId, entityType, entityId, occurredAt DESC)` is placed on the Activity table.

---

### Cross-Context Reference Rules

| When | Rule |
|------|------|
| Opportunity references Account | Only ID is stored. Account data fetched via its own aggregate root. |
| Activity references any entity | Polymorphic by entityType + entityId. No FK enforced at DB level. |
| Lead references converted entities | IDs stored as nullable FKs; no cascade on the referenced entities. |
| AuditLog references any entity | String entityType + UUID entityId. Logs are immutable; no FK cascade. |

---

## Database Index Strategy

| Table | Index Columns | Purpose |
|-------|--------------|---------|
| users | (tenantId, email) UNIQUE | Login lookup |
| users | (tenantId, status) | Active user queries |
| leads | (tenantId, status, ownerId) | Lead list with filters |
| leads | (tenantId, createdAt DESC) | Recent leads |
| opportunities | (tenantId, pipelineId, stageId) | Kanban board query |
| opportunities | (tenantId, ownerId, status) | Rep pipeline view |
| opportunities | (tenantId, expectedCloseDate) | Forecast queries |
| contacts | (tenantId, email) | Duplicate detection |
| contacts | (tenantId, primaryAccountId) | Contacts by account |
| activity | (tenantId, entityType, entityId, occurredAt DESC) | Timeline queries |
| audit_log | (tenantId, entityType, entityId) | Entity audit trail |
| audit_log | (tenantId, userId, createdAt DESC) | User audit trail |
| tasks | (tenantId, assigneeId, status, dueDate) | Task list queries |
| notifications | (tenantId, userId, isRead) | Unread notification count |

---

## Cascade Behavior Reference

| Entity Deleted | Cascade Effect |
|----------------|---------------|
| Tenant (hard delete — admin only) | Cascade soft-delete all tenant data; schedule purge after 30 days |
| User (soft delete) | Reassign owned records to admin; preserve history |
| Account (soft delete) | Retain all Contacts, Leads, Opportunities; soft-delete Account only |
| Contact (soft delete) | Remove ContactAccount entries; retain Activities, Emails, Notes |
| Lead (soft delete) | Retain Activities, Notes; converted entities unaffected |
| Opportunity (soft delete) | Retain Activities, Notes; Stage/Pipeline unaffected |
| Pipeline (blocked if open opps exist) | Cascade delete Stages only when no Opportunities in pipeline |
| Stage (blocked if opps in stage) | Admin must migrate opportunities first |
