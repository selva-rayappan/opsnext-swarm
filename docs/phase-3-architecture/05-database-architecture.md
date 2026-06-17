# Database Architecture — OpsNext CRM

---

## Overview

OpsNext CRM uses **PostgreSQL 16** as the primary database. All schema management is via **Prisma Migrate**. No ad-hoc schema changes are permitted outside of migrations.

**Key decisions**:
- Single database instance per deployment (shared by all tenants — row-level isolation)
- Read replica for analytics queries (optional in v1, required at scale)
- PgBouncer for connection pooling
- Soft delete on all business entities (`deletedAt` timestamp)
- Prisma 5 with client extensions for tenant scoping

---

## Prisma Schema

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ─────────────────────────────────────────────
// TENANT MANAGEMENT
// ─────────────────────────────────────────────

model Tenant {
  id                String       @id @default(uuid())
  name              String
  slug              String       @unique
  plan              Plan         @default(FREE)
  status            TenantStatus @default(ACTIVE)
  settings          Json         @default("{}")
  stripeCustomerId  String?
  stripePriceId     String?
  createdAt         DateTime     @default(now())
  updatedAt         DateTime     @updatedAt

  users             User[]
  roles             Role[]
  teams             Team[]
  pipelines         Pipeline[]
  leads             Lead[]
  contacts          Contact[]
  accounts          Account[]
  opportunities     Opportunity[]
  activities        Activity[]
  tasks             Task[]
  emails            Email[]
  notifications     Notification[]
  auditLogs         AuditLog[]

  @@map("tenants")
}

enum Plan {
  FREE
  STARTER
  GROWTH
  ENTERPRISE
}

enum TenantStatus {
  ACTIVE
  SUSPENDED
  CANCELLED
}

// ─────────────────────────────────────────────
// IAM
// ─────────────────────────────────────────────

model User {
  id            String     @id @default(uuid())
  tenantId      String
  email         String
  passwordHash  String
  firstName     String
  lastName      String
  phone         String?
  avatarUrl     String?
  locale        String     @default("en-US")
  timezone      String     @default("UTC")
  status        UserStatus @default(PENDING_INVITE)
  roleId        String
  lastLoginAt   DateTime?
  deletedAt     DateTime?
  createdAt     DateTime   @default(now())
  updatedAt     DateTime   @updatedAt

  tenant        Tenant         @relation(fields: [tenantId], references: [id])
  role          Role           @relation(fields: [roleId], references: [id])
  teamMembers   TeamMember[]
  refreshTokens RefreshToken[]
  ownedLeads    Lead[]         @relation("LeadOwner")
  ownedContacts Contact[]      @relation("ContactOwner")
  ownedAccounts Account[]      @relation("AccountOwner")
  ownedOpps     Opportunity[]  @relation("OpportunityOwner")
  activities    Activity[]
  tasks         Task[]

  @@unique([tenantId, email])
  @@index([tenantId, status])
  @@map("users")
}

enum UserStatus {
  PENDING_INVITE
  ACTIVE
  INACTIVE
}

model Role {
  id          String   @id @default(uuid())
  tenantId    String
  name        String
  description String?
  isSystem    Boolean  @default(false)
  permissions String[] // Array of "resource:action" strings

  tenant      Tenant   @relation(fields: [tenantId], references: [id])
  users       User[]

  @@unique([tenantId, name])
  @@map("roles")
}

model RefreshToken {
  id         String    @id @default(uuid())
  userId     String
  tenantId   String
  tokenHash  String    @unique
  expiresAt  DateTime
  revokedAt  DateTime?
  userAgent  String?
  ipAddress  String?
  createdAt  DateTime  @default(now())

  user       User      @relation(fields: [userId], references: [id])

  @@index([userId])
  @@index([tokenHash])
  @@map("refresh_tokens")
}

model Team {
  id          String       @id @default(uuid())
  tenantId    String
  name        String
  leadId      String?
  description String?
  createdAt   DateTime     @default(now())
  updatedAt   DateTime     @updatedAt

  tenant      Tenant       @relation(fields: [tenantId], references: [id])
  members     TeamMember[]

  @@unique([tenantId, name])
  @@map("teams")
}

model TeamMember {
  teamId    String
  userId    String
  joinedAt  DateTime @default(now())

  team      Team     @relation(fields: [teamId], references: [id])
  user      User     @relation(fields: [userId], references: [id])

  @@id([teamId, userId])
  @@map("team_members")
}

// ─────────────────────────────────────────────
// CONTACT & ACCOUNT
// ─────────────────────────────────────────────

model Account {
  id            String        @id @default(uuid())
  tenantId      String
  name          String
  industry      String?
  website       String?
  phone         String?
  email         String?
  address       Json?
  employees     Int?
  annualRevenue Decimal?      @db.Decimal(18, 2)
  currency      String?       @default("USD")
  description   String?
  ownerId       String
  status        EntityStatus  @default(ACTIVE)
  customFields  Json          @default("{}")
  deletedAt     DateTime?
  createdAt     DateTime      @default(now())
  updatedAt     DateTime      @updatedAt

  tenant            Tenant           @relation(fields: [tenantId], references: [id])
  owner             User             @relation("AccountOwner", fields: [ownerId], references: [id])
  contactAccounts   ContactAccount[]
  leads             Lead[]
  opportunities     Opportunity[]
  notes             Note[]
  attachments       Attachment[]
  activities        Activity[]

  @@index([tenantId, status])
  @@index([tenantId, ownerId])
  @@map("accounts")
}

model Contact {
  id               String        @id @default(uuid())
  tenantId         String
  firstName        String
  lastName         String
  email            String?
  phone            String?
  mobilePhone      String?
  jobTitle         String?
  department       String?
  linkedinUrl      String?
  ownerId          String
  status           EntityStatus  @default(ACTIVE)
  customFields     Json          @default("{}")
  deletedAt        DateTime?
  createdAt        DateTime      @default(now())
  updatedAt        DateTime      @updatedAt

  tenant               Tenant               @relation(fields: [tenantId], references: [id])
  owner                User                 @relation("ContactOwner", fields: [ownerId], references: [id])
  contactAccounts      ContactAccount[]
  fromRelationships    ContactRelationship[] @relation("FromContact")
  toRelationships      ContactRelationship[] @relation("ToContact")
  leads                Lead[]
  opportunities        Opportunity[]
  notes                Note[]
  attachments          Attachment[]
  activities           Activity[]
  emails               Email[]

  @@index([tenantId, email])
  @@index([tenantId, ownerId])
  @@map("contacts")
}

model ContactAccount {
  contactId  String
  accountId  String
  isPrimary  Boolean  @default(false)
  role       String?
  createdAt  DateTime @default(now())

  contact    Contact  @relation(fields: [contactId], references: [id])
  account    Account  @relation(fields: [accountId], references: [id])

  @@id([contactId, accountId])
  @@map("contact_accounts")
}

model ContactRelationship {
  id               String   @id @default(uuid())
  tenantId         String
  fromContactId    String
  toContactId      String
  relationshipType String
  description      String?
  createdAt        DateTime @default(now())

  fromContact      Contact  @relation("FromContact", fields: [fromContactId], references: [id])
  toContact        Contact  @relation("ToContact", fields: [toContactId], references: [id])

  @@map("contact_relationships")
}

enum EntityStatus {
  ACTIVE
  ARCHIVED
}

// ─────────────────────────────────────────────
// LEAD MANAGEMENT
// ─────────────────────────────────────────────

model Lead {
  id                      String      @id @default(uuid())
  tenantId                String
  firstName               String
  lastName                String
  email                   String?
  phone                   String?
  company                 String?
  jobTitle                String?
  source                  LeadSource  @default(OTHER)
  status                  LeadStatus  @default(NEW)
  qualificationNotes      String?
  disqualificationReason  String?
  ownerId                 String?
  teamId                  String?
  accountId               String?     // Optional link to Account
  contactId               String?     // Optional link to Contact
  convertedAt             DateTime?
  convertedContactId      String?
  convertedAccountId      String?
  convertedOpportunityId  String?
  customFields            Json        @default("{}")
  deletedAt               DateTime?
  createdAt               DateTime    @default(now())
  updatedAt               DateTime    @updatedAt

  tenant      Tenant        @relation(fields: [tenantId], references: [id])
  owner       User?         @relation("LeadOwner", fields: [ownerId], references: [id])
  account     Account?      @relation(fields: [accountId], references: [id])
  contact     Contact?      @relation(fields: [contactId], references: [id])
  notes       Note[]
  attachments Attachment[]
  activities  Activity[]

  @@index([tenantId, status, ownerId])
  @@index([tenantId, createdAt(sort: Desc)])
  @@map("leads")
}

enum LeadStatus {
  NEW
  CONTACTED
  QUALIFIED
  CONVERTED
  DISQUALIFIED
}

enum LeadSource {
  WEB_FORM
  COLD_OUTREACH
  REFERRAL
  EVENT
  INBOUND
  IMPORT
  API
  OTHER
}

// ─────────────────────────────────────────────
// SALES PIPELINE
// ─────────────────────────────────────────────

model Pipeline {
  id          String     @id @default(uuid())
  tenantId    String
  name        String
  isDefault   Boolean    @default(false)
  description String?
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt

  tenant        Tenant        @relation(fields: [tenantId], references: [id])
  stages        Stage[]
  opportunities Opportunity[]

  @@unique([tenantId, name])
  @@map("pipelines")
}

model Stage {
  id          String   @id @default(uuid())
  tenantId    String
  pipelineId  String
  name        String
  ordinal     Int
  probability Int      @default(0)
  isWon       Boolean  @default(false)
  isLost      Boolean  @default(false)
  description String?

  pipeline      Pipeline      @relation(fields: [pipelineId], references: [id])
  opportunities Opportunity[]

  @@unique([pipelineId, ordinal])
  @@map("stages")
}

model Opportunity {
  id                String          @id @default(uuid())
  tenantId          String
  name              String
  accountId         String?
  primaryContactId  String?
  pipelineId        String
  stageId           String
  amount            Decimal?        @db.Decimal(18, 2)
  currency          String          @default("USD")
  expectedCloseDate DateTime?       @db.Date
  probability       Int?
  status            OpportunityStatus @default(OPEN)
  lostReason        String?
  ownerId           String
  description       String?
  customFields      Json            @default("{}")
  closedAt          DateTime?
  deletedAt         DateTime?
  createdAt         DateTime        @default(now())
  updatedAt         DateTime        @updatedAt

  tenant        Tenant      @relation(fields: [tenantId], references: [id])
  account       Account?    @relation(fields: [accountId], references: [id])
  contact       Contact?    @relation(fields: [primaryContactId], references: [id])
  pipeline      Pipeline    @relation(fields: [pipelineId], references: [id])
  stage         Stage       @relation(fields: [stageId], references: [id])
  owner         User        @relation("OpportunityOwner", fields: [ownerId], references: [id])
  notes         Note[]
  attachments   Attachment[]
  activities    Activity[]

  @@index([tenantId, pipelineId, stageId])
  @@index([tenantId, ownerId, status])
  @@index([tenantId, expectedCloseDate])
  @@map("opportunities")
}

enum OpportunityStatus {
  OPEN
  WON
  LOST
}

// ─────────────────────────────────────────────
// ACTIVITY & TASK
// ─────────────────────────────────────────────

model Activity {
  id          String       @id @default(uuid())
  tenantId    String
  type        ActivityType
  entityType  String
  entityId    String
  userId      String
  subject     String?
  body        String?
  occurredAt  DateTime     @default(now())
  metadata    Json         @default("{}")

  tenant      Tenant       @relation(fields: [tenantId], references: [id])
  user        User         @relation(fields: [userId], references: [id])

  // Polymorphic relations (not Prisma FK — enforced at app layer)
  leadEntity      Lead?        @relation(fields: [entityId], references: [id], map: "activity_lead_fk")
  contactEntity   Contact?     @relation(fields: [entityId], references: [id], map: "activity_contact_fk")
  accountEntity   Account?     @relation(fields: [entityId], references: [id], map: "activity_account_fk")
  opportunityEntity Opportunity? @relation(fields: [entityId], references: [id], map: "activity_opportunity_fk")

  @@index([tenantId, entityType, entityId, occurredAt(sort: Desc)])
  @@index([tenantId, userId, occurredAt(sort: Desc)])
  @@map("activities")
}

enum ActivityType {
  NOTE
  CALL
  MEETING
  EMAIL
  TASK
  STAGE_CHANGE
  ASSIGNMENT
  SYSTEM
}

model Task {
  id             String      @id @default(uuid())
  tenantId       String
  title          String
  description    String?
  dueDate        DateTime?
  priority       TaskPriority @default(MEDIUM)
  status         TaskStatus   @default(OPEN)
  assigneeId     String
  creatorId      String
  entityType     String?
  entityId       String?
  completedAt    DateTime?
  isRecurring    Boolean      @default(false)
  recurrenceRule String?
  deletedAt      DateTime?
  createdAt      DateTime     @default(now())
  updatedAt      DateTime     @updatedAt

  tenant    Tenant   @relation(fields: [tenantId], references: [id])
  assignee  User     @relation(fields: [assigneeId], references: [id])

  @@index([tenantId, assigneeId, status, dueDate])
  @@map("tasks")
}

enum TaskPriority { LOW MEDIUM HIGH URGENT }
enum TaskStatus   { OPEN IN_PROGRESS COMPLETED CANCELLED }

// ─────────────────────────────────────────────
// EMAIL & COMMUNICATION
// ─────────────────────────────────────────────

model Email {
  id              String   @id @default(uuid())
  tenantId        String
  userId          String
  direction       EmailDirection
  fromAddress     String
  toAddresses     String[]
  ccAddresses     String[] @default([])
  subject         String
  body            String
  bodyHtml        String?
  sentAt          DateTime
  contactId       String?
  accountId       String?
  leadId          String?
  opportunityId   String?
  createdAt       DateTime @default(now())

  tenant          Tenant   @relation(fields: [tenantId], references: [id])
  contact         Contact? @relation(fields: [contactId], references: [id])

  @@index([tenantId, contactId])
  @@index([tenantId, sentAt(sort: Desc)])
  @@map("emails")
}

enum EmailDirection { INBOUND OUTBOUND }

model EmailTemplate {
  id         String   @id @default(uuid())
  tenantId   String
  name       String
  subject    String
  body       String
  category   String   @default("other")
  isShared   Boolean  @default(true)
  ownerId    String
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt

  @@unique([tenantId, name])
  @@map("email_templates")
}

// ─────────────────────────────────────────────
// SUPPORTING
// ─────────────────────────────────────────────

model Note {
  id            String    @id @default(uuid())
  tenantId      String
  entityType    String
  entityId      String
  body          String
  authorId      String
  deletedAt     DateTime?
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  @@index([tenantId, entityType, entityId])
  @@map("notes")
}

model Attachment {
  id          String   @id @default(uuid())
  tenantId    String
  entityType  String
  entityId    String
  fileName    String
  fileSize    Int
  mimeType    String
  storageKey  String
  uploadedBy  String
  createdAt   DateTime @default(now())

  @@index([tenantId, entityType, entityId])
  @@map("attachments")
}

model Notification {
  id          String    @id @default(uuid())
  tenantId    String
  userId      String
  type        String
  title       String
  body        String
  entityType  String?
  entityId    String?
  isRead      Boolean   @default(false)
  readAt      DateTime?
  createdAt   DateTime  @default(now())

  tenant      Tenant    @relation(fields: [tenantId], references: [id])

  @@index([tenantId, userId, isRead])
  @@map("notifications")
}

model AuditLog {
  id          String      @id @default(uuid())
  tenantId    String
  userId      String
  entityType  String
  entityId    String
  action      AuditAction
  before      Json?
  after       Json?
  ipAddress   String?
  userAgent   String?
  createdAt   DateTime    @default(now())

  tenant      Tenant      @relation(fields: [tenantId], references: [id])

  @@index([tenantId, entityType, entityId])
  @@index([tenantId, userId, createdAt(sort: Desc)])
  @@map("audit_logs")
}

enum AuditAction { CREATED UPDATED DELETED RESTORED }
```

---

## Index Strategy

| Table | Index | Purpose |
|-------|-------|---------|
| users | `(tenantId, email) UNIQUE` | Login lookup, duplicate detection |
| users | `(tenantId, status)` | Active user queries |
| leads | `(tenantId, status, ownerId)` | Filtered lead list |
| leads | `(tenantId, createdAt DESC)` | Recent leads |
| contacts | `(tenantId, email)` | Duplicate detection |
| contacts | `(tenantId, ownerId)` | My contacts view |
| opportunities | `(tenantId, pipelineId, stageId)` | Kanban board |
| opportunities | `(tenantId, ownerId, status)` | Rep pipeline view |
| opportunities | `(tenantId, expectedCloseDate)` | Forecast queries |
| activities | `(tenantId, entityType, entityId, occurredAt DESC)` | Timeline queries |
| audit_logs | `(tenantId, entityType, entityId)` | Record audit trail |
| audit_logs | `(tenantId, userId, createdAt DESC)` | User audit trail |
| tasks | `(tenantId, assigneeId, status, dueDate)` | My tasks view |
| notifications | `(tenantId, userId, isRead)` | Unread notification count |
| refresh_tokens | `(tokenHash) UNIQUE` | Token lookup on refresh |

---

## Migration Strategy

```bash
# Development
npx prisma migrate dev --name "add_opportunity_probability"

# Production (CI/CD pipeline)
npx prisma migrate deploy   # Applies pending migrations; idempotent
```

### Migration Rules
1. All schema changes via Prisma migration files — no ad-hoc `ALTER TABLE`
2. Migrations committed to git alongside application code
3. Migration files are immutable once merged to main
4. Breaking changes (column removal, rename) use a 3-step migration: add new → dual-write → remove old
5. Large table alterations use `CREATE INDEX CONCURRENTLY` to avoid table locks

---

## Soft Delete Pattern

```typescript
// All repositories apply this filter by default
const DEFAULT_FILTER = { deletedAt: null };

// Prisma extension: auto-apply soft-delete filter
$extends({
  query: {
    $allModels: {
      async findMany({ args, query }) {
        args.where = { ...args.where, deletedAt: null, tenantId };
        return query(args);
      },
      async delete({ args, query }) {
        // Intercept delete → convert to soft delete
        return (this as any).update({
          ...args,
          data: { deletedAt: new Date() },
        });
      },
    },
  },
})
```

---

## Connection Pooling (PgBouncer)

```ini
# pgbouncer.ini
[databases]
opsnext = host=postgres port=5432 dbname=opsnext_prod

[pgbouncer]
pool_mode = transaction          # Transaction-mode pooling
max_client_conn = 1000           # Max connections from app servers
default_pool_size = 100          # Connections to PostgreSQL
min_pool_size = 10
reserve_pool_size = 5
server_idle_timeout = 600
```

**Why transaction-mode**: Prisma creates/releases connections per query. Transaction-mode pooling is compatible and most efficient for this pattern.

---

## Data Retention & Purge Policy

| Data Type | Soft-Delete Retention | Hard-Delete Policy |
|-----------|----------------------|--------------------|
| Leads, Contacts, Accounts, Opportunities | 30 days | Purge job after 30 days |
| Audit logs | 12 months (active), then archive | Archive to cold storage after 12 months |
| Notification records | 90 days | Purge after 90 days |
| Refresh tokens (revoked/expired) | 7 days | Purge after 7 days |
| Email logs | 2 years | Archive after 2 years |
| Attachments (S3) | 30 days post soft-delete | Purge S3 object after 30 days |

Purge jobs run as scheduled BullMQ tasks (daily, off-peak hours).
