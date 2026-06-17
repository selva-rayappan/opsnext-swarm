# Integration Architecture — OpsNext CRM

---

## Integration Overview

OpsNext CRM integrates with external systems through three mechanisms:

1. **Inbound**: REST API (authenticated) + Lead Ingest API (API key) + BCC email capture
2. **Outbound**: Webhooks (event-driven HTTP POST to tenant-configured endpoints)
3. **Internal async**: BullMQ job queues for email delivery, CSV import, data export, webhook dispatch

```
                    INBOUND                         OUTBOUND
                       │                               │
┌──────────────────────▼───────────────────────────────▼──────────────┐
│                   OpsNext CRM API                                    │
│                                                                      │
│  Web Forms ──► /api/v1/leads/ingest (API Key auth)                  │
│  Zapier    ──► /api/v1/* (JWT or API Key)          Webhooks ──► Slack│
│  CSV Upload──► /api/v1/leads/import                         ──► HubSpot│
│  BCC Email ──► /api/v1/email/ingest                         ──► Zapier│
│                                                             ──► Any  │
│                    INTERNAL ASYNC                                    │
│                                                                      │
│  BullMQ Queues: email | export | import | webhook | report | purge  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## REST API Versioning Strategy

All API endpoints are versioned under `/api/v1/`. Version is in the URL path.

### Version Policy
- **v1** is current and stable
- New versions are introduced when breaking changes are required
- Old versions are deprecated with 6-month notice, then sunset
- Version is declared in route prefix: `@Controller({ path: 'leads', version: '1' })`

### API Key Authentication (Machine-to-Machine)

For server-to-server integrations (web forms, Zapier, CI tools):

```
POST /api/v1/auth/api-keys
Authorization: Bearer <admin-JWT>

→ { key: "ops_live_abc123...", name: "Web Form Integration" }
```

API keys are stored as bcrypt hashes in the database. Plaintext is shown once at creation.

```typescript
// Ingest endpoint — API key auth
@Post('ingest')
@UseGuards(ApiKeyGuard)   // Validates api key, resolves tenantId from key
@Throttle({ default: { limit: 100, ttl: 60000 } })
async ingestLead(
  @Body() dto: IngestLeadDto,
  @Tenant() tenantId: string,
): Promise<{ id: string }> {
  return this.leadService.ingest(dto, tenantId);
}
```

---

## Webhook System

### Architecture

```
Domain Event Fired
       │
       ▼
WebhookService.dispatch(event)
  1. Look up tenant's webhook configs matching this event type
  2. For each matching config: enqueue a WebhookJob in BullMQ
       │
       ▼
WebhookWorker (BullMQ worker)
  1. Build payload: { event, tenantId, data, timestamp, deliveryId }
  2. Sign payload: HMAC-SHA256(secret, JSON.stringify(payload))
  3. POST to webhook URL with:
     X-OpsNext-Signature: sha256=<hmac>
     X-OpsNext-Event: lead.created
     X-OpsNext-Delivery: <uuid>
     Content-Type: application/json
  4. If 2XX: mark delivery as success, log
  5. If non-2XX or timeout: retry with backoff
     Attempt 1 → wait 1s → Attempt 2 → wait 5s → Attempt 3 → wait 30s
  6. After 3 failures: mark delivery as failed, log error, notify admin
```

### Webhook Configuration Schema

```typescript
// Stored per tenant in webhooks table
interface WebhookConfig {
  id: string;
  tenantId: string;
  url: string;                    // HTTPS only
  secret: string;                 // Encrypted at rest (AES-256)
  events: string[];               // e.g., ['lead.created', 'opportunity.stage_changed']
  isActive: boolean;
  createdAt: Date;
}
```

### Supported Webhook Events

| Event | Trigger |
|-------|---------|
| `lead.created` | Lead created |
| `lead.assigned` | Lead ownership changed |
| `lead.converted` | Lead converted to opportunity |
| `lead.disqualified` | Lead disqualified |
| `opportunity.created` | Opportunity created |
| `opportunity.stage_changed` | Stage updated |
| `opportunity.closed_won` | Deal won |
| `opportunity.closed_lost` | Deal lost |
| `contact.created` | Contact created |
| `account.created` | Account created |
| `task.completed` | Task marked complete |
| `user.invited` | User invited to tenant |

### Payload Envelope

```json
{
  "deliveryId": "uuid",
  "event": "opportunity.stage_changed",
  "tenantId": "uuid",
  "occurredAt": "2026-06-17T10:30:00Z",
  "data": {
    "opportunityId": "uuid",
    "name": "Acme Corp Renewal",
    "previousStage": "Proposal",
    "newStage": "Negotiation",
    "amount": 45000,
    "currency": "USD",
    "ownerId": "uuid"
  }
}
```

### HMAC Signature Verification (Consumer)

```typescript
// How a webhook consumer verifies the payload
const signature = req.headers['x-opsnext-signature'];       // "sha256=<hex>"
const expectedSig = 'sha256=' + createHmac('sha256', secret)
  .update(JSON.stringify(req.body))
  .digest('hex');

if (!timingSafeEqual(Buffer.from(signature), Buffer.from(expectedSig))) {
  return res.status(401).send('Invalid signature');
}
```

---

## CSV Import Pipeline

### Lead Import Flow

```
POST /api/v1/leads/import
Content-Type: multipart/form-data
(file: CSV, fieldMappings: JSON)

  ▼
1. ValidateCSVFile: size check, encoding, max 10,000 rows
  ▼
2. ParseCSV: stream-parse using papaparse
  ▼
3. Store raw CSV in S3 with import job ID
  ▼
4. Create ImportJob record { status: 'processing', totalRows, tenantId }
  ▼
5. Return 202 Accepted { importJobId }
  ▼
[BullMQ worker: import queue]
  ▼
6. For each row (batch of 50):
   a. Apply field mappings
   b. Validate row (email format, required fields)
   c. Check duplicate email → skip or merge per policy
   d. Upsert Lead record
   e. Increment successCount / errorCount
  ▼
7. Update ImportJob { status: 'completed', successCount, errorCount, errorLogKey }
  ▼
8. Notify user: in-app notification + email with download link for error report
```

### Field Mapping

```typescript
// Sent by client in the import request
interface FieldMapping {
  csvColumn: string;       // e.g., "First Name"
  crmField: string;        // e.g., "firstName"
  transform?: 'lowercase' | 'uppercase' | 'trim' | 'phone_normalize';
}
```

---

## BCC Email Capture

Each user gets a unique BCC address: `{userId-hash}@ingest.{tenantSlug}.opsnext.io`

```
Email sent (CC: above BCC address)
       │
       ▼
Email provider webhook (e.g., SendGrid Inbound Parse)
  POST /api/v1/email/ingest (internal endpoint, IP allowlisted)
       │
       ▼
BccIngestService.process(rawEmail)
  1. Parse From, To, CC, Subject, Body from raw MIME
  2. Resolve userId from BCC address token
  3. Match contact by From/To email against CRM contacts
  4. Create Email record linked to matched contact/account/lead
  5. Create Activity entry (type: EMAIL) on linked entities
  6. Return 200
```

---

## Queue Architecture (BullMQ)

```typescript
// Queue names and their jobs
const QUEUES = {
  EMAIL:   'opsnext:email',     // Send transactional emails
  IMPORT:  'opsnext:import',    // Process CSV imports
  EXPORT:  'opsnext:export',    // Generate data exports
  WEBHOOK: 'opsnext:webhook',   // Deliver outbound webhooks
  REPORT:  'opsnext:report',    // Generate scheduled reports
  PURGE:   'opsnext:purge',     // Soft-delete purge jobs (daily)
};
```

### Queue Configuration

```typescript
// Shared BullMQ connection (Redis)
const connection = { host: config.REDIS_HOST, port: config.REDIS_PORT };

// Email queue — fast, high priority
const emailQueue = new Queue(QUEUES.EMAIL, {
  connection,
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 2000 },
    removeOnComplete: { count: 100 },
    removeOnFail: { count: 1000 },
  },
});

// Webhook queue — medium priority, aggressive retry
const webhookQueue = new Queue(QUEUES.WEBHOOK, {
  connection,
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'fixed', delay: [1000, 5000, 30000] },
  },
});

// Import queue — low priority, large jobs
const importQueue = new Queue(QUEUES.IMPORT, {
  connection,
  defaultJobOptions: {
    attempts: 1,    // No retry — user must re-upload on failure
    timeout: 300000, // 5 minute timeout
  },
});
```

---

## External Service Integrations (v1 Stubs + v2 Full)

| Service | v1 | v2 |
|---------|----|----|
| **SMTP / SendGrid** | Configured via env var (`SMTP_*`). Nodemailer-based. | SendGrid API with event tracking |
| **S3-compatible storage** | AWS S3 or MinIO (Docker). Presigned URL generation. | Multi-region S3 with CDN in front |
| **Stripe** | Tenant model has `stripeCustomerId` field. No billing logic. | Full subscription management, webhooks |
| **Google Calendar** | Stub interface defined (`ICalendarProvider`). No implementation. | OAuth + Google Calendar sync |
| **Slack** | Webhook delivery handles Slack naturally (standard webhook). | Native Slack app with slash commands |

### Provider Interface Pattern

```typescript
// External services implement provider interfaces for easy swap
interface IEmailProvider {
  sendTransactional(to: string, subject: string, html: string, text: string): Promise<void>;
}

interface IStorageProvider {
  upload(key: string, buffer: Buffer, mimeType: string): Promise<string>;
  getPresignedUrl(key: string, expirySeconds: number): Promise<string>;
  delete(key: string): Promise<void>;
}

// Registered as NestJS providers — swappable via config
// STORAGE_PROVIDER=s3 → AwsS3StorageProvider
// STORAGE_PROVIDER=minio → MinioStorageProvider
// STORAGE_PROVIDER=local → LocalStorageProvider (dev only)
```

---

## API Observability

All API responses include:

```
X-Request-ID: {correlationId}
X-Response-Time: {ms}
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 997
X-RateLimit-Reset: 1700001000
```

Errors follow RFC 7807:
```json
{
  "type": "https://opsnext.io/errors/validation",
  "title": "Validation Failed",
  "status": 422,
  "detail": "Request body failed validation",
  "correlationId": "req-uuid",
  "errors": [
    { "field": "email", "message": "must be a valid email" }
  ]
}
```
