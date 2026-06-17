# Security Architecture — OpsNext CRM

---

## Security Layers Overview

```
External Traffic
      │
      ▼
┌─────────────────────────────────────────────────────────────────┐
│  Layer 1: Network / Transport                                   │
│  • TLS 1.3 termination at Nginx reverse proxy                   │
│  • HSTS preload; no HTTP allowed                                │
│  • IP-based rate limiting at Nginx level                        │
└─────────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────┐
│  Layer 2: HTTP Security Headers                                 │
│  • Helmet.js: CSP, X-Frame-Options, HSTS, nosniff, etc.         │
│  • CORS: allowlist-only origins                                 │
└─────────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────┐
│  Layer 3: Authentication & Session                              │
│  • JWT RS256 validation (JwtAuthGuard)                          │
│  • Refresh token rotation (Redis-backed)                        │
│  • Rate limiting per IP on auth endpoints (ThrottlerGuard)      │
└─────────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────┐
│  Layer 4: Authorization (RBAC)                                  │
│  • PermissionsGuard: checks required permission on every route  │
│  • TenantMiddleware: binds request to tenantId from JWT         │
│  • PlanGateGuard: enforces subscription plan limits             │
└─────────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────┐
│  Layer 5: Input Validation                                      │
│  • class-validator + class-transformer on all request bodies    │
│  • Zod validation on query parameters                           │
│  • File MIME type + magic byte validation                       │
└─────────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────┐
│  Layer 6: Data Access (Database)                                │
│  • Prisma tenant extension: automatic tenantId filter           │
│  • PostgreSQL RLS: DB-level tenant enforcement                  │
│  • Parameterized queries only (no raw SQL with user input)      │
│  • Soft delete: no permanent data loss on user action           │
└─────────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7: Audit & Monitoring                                    │
│  • Immutable audit log (all writes)                             │
│  • Structured access logs (no PII)                              │
│  • Error monitoring (Sentry)                                    │
│  • Anomaly detection via metrics thresholds                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Authentication Architecture

### JWT Flow (RS256)

```
┌──────────┐   POST /auth/login   ┌───────────────────────────────┐
│  Client  │─────────────────────►│  AuthService.login()          │
│          │                      │  1. Find user by email+tenant  │
│          │                      │  2. bcrypt.compare(password)   │
│          │                      │  3. Generate access token      │
│          │                      │     { userId, tenantId,        │
│          │                      │       roleId, permissions }    │
│          │                      │     Signed: RS256 private key  │
│          │                      │     Expiry: 15 minutes         │
│          │                      │  4. Generate refresh token     │
│          │                      │     Random UUID → SHA-256 hash │
│          │                      │     Stored in Redis + DB       │
│          │                      │     Expiry: 30 days            │
│          │◄────────────────────│                                │
│          │  200: { accessToken }│                                │
│          │  Set-Cookie:         │                                │
│          │    refreshToken=...  │                                │
│          │    HttpOnly;Secure;  │                                │
│          │    SameSite=Strict   │                                │
└──────────┘                      └───────────────────────────────┘
```

### Access Token Structure (JWT Payload)
```json
{
  "sub": "user-uuid",
  "tenantId": "tenant-uuid",
  "roleId": "role-uuid",
  "permissions": ["leads:read", "leads:create", "contacts:read", "..."],
  "iat": 1700000000,
  "exp": 1700000900
}
```

Permissions are embedded in the token to avoid a Redis lookup on every request.

### Refresh Token Rotation
```
Client has expired access token + valid refresh token cookie
      │
      ▼
POST /auth/refresh (cookie sent automatically by browser)
      │
      ▼
AuthService.refreshTokens()
  1. Read refreshToken from cookie
  2. Hash it: SHA-256(rawToken)
  3. Look up hash in DB (RefreshToken table)
  4. Verify: not revoked, not expired, matches userId
  5. REVOKE old refresh token (set revokedAt = now())
  6. Generate new access token (new permissions lookup)
  7. Generate new refresh token → store hash in DB
  8. Return new access token + set new refresh token cookie
```

**Reuse detection**: If a previously revoked refresh token is presented, all sessions for that user are immediately revoked (possible token theft).

```typescript
// auth.service.ts
async refreshTokens(rawRefreshToken: string): Promise<AuthTokens> {
  const hash = sha256(rawRefreshToken);
  const stored = await this.db.refreshToken.findUnique({ where: { tokenHash: hash } });

  if (!stored) throw new UnauthorizedException();

  if (stored.revokedAt !== null) {
    // Reuse detected — revoke ALL sessions for this user
    await this.revokeAllUserSessions(stored.userId);
    throw new UnauthorizedException('Token reuse detected');
  }

  if (stored.expiresAt < new Date()) throw new UnauthorizedException('Token expired');

  // Rotate
  await this.db.refreshToken.update({
    where: { id: stored.id },
    data: { revokedAt: new Date() },
  });

  return this.issueTokenPair(stored.userId, stored.tenantId);
}
```

---

## Authorization Architecture (RBAC)

### PermissionsGuard Implementation

```typescript
// common/guards/permissions.guard.ts
@Injectable()
export class PermissionsGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const required = this.reflector.getAllAndOverride<string[]>('permissions', [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!required?.length) return true;

    const { user } = context.switchToHttp().getRequest();
    // user.permissions is extracted from JWT — no DB call needed
    const hasAll = required.every(p => user.permissions.includes(p));

    if (!hasAll) {
      throw new ForbiddenException({
        type: 'https://opsnext.io/errors/forbidden',
        title: 'Insufficient Permissions',
        requiredPermissions: required,
        userPermissions: user.permissions,
      });
    }
    return true;
  }
}
```

### Route-Level Permission Declaration
```typescript
// lead.controller.ts
@Post()
@UseGuards(JwtAuthGuard, PermissionsGuard)
@Permissions('leads:create')
@HttpCode(201)
async create(
  @Body() dto: CreateLeadDto,
  @CurrentUser() user: AuthUser,
): Promise<LeadResponse> {
  return this.leadService.create(dto, user.tenantId, user.id);
}
```

### Permission Matrix (Default System Roles)

| Permission | super_admin | admin | manager | sales_rep | viewer |
|-----------|:-----------:|:-----:|:-------:|:---------:|:------:|
| contacts:read | ✓ | ✓ | ✓ | ✓ | ✓ |
| contacts:create | ✓ | ✓ | ✓ | ✓ | ✗ |
| contacts:update | ✓ | ✓ | ✓ | ✓ | ✗ |
| contacts:delete | ✓ | ✓ | ✗ | ✗ | ✗ |
| leads:read | ✓ | ✓ | ✓ | ✓ | ✓ |
| leads:create | ✓ | ✓ | ✓ | ✓ | ✗ |
| leads:assign | ✓ | ✓ | ✓ | ✗ | ✗ |
| opportunities:read | ✓ | ✓ | ✓ | ✓ | ✓ |
| opportunities:create | ✓ | ✓ | ✓ | ✓ | ✗ |
| opportunities:delete | ✓ | ✓ | ✗ | ✗ | ✗ |
| reports:read | ✓ | ✓ | ✓ | ✗ | ✗ |
| settings:manage | ✓ | ✓ | ✗ | ✗ | ✗ |
| users:manage | ✓ | ✓ | ✗ | ✗ | ✗ |
| audit_logs:read | ✓ | ✓ | ✗ | ✗ | ✗ |

---

## Input Validation Pipeline

```typescript
// main.ts bootstrap
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,        // Strip unknown properties
    forbidNonWhitelisted: true,  // Throw on unknown properties
    transform: true,        // Auto-transform types (string → number, etc.)
    transformOptions: { enableImplicitConversion: false },
    errorHttpStatusCode: 422,    // 422 Unprocessable Entity for validation errors
  }),
);
```

### DTO Example
```typescript
// create-lead.dto.ts
export class CreateLeadDto {
  @IsString()
  @MinLength(1)
  @MaxLength(100)
  firstName: string;

  @IsString()
  @MinLength(1)
  @MaxLength(100)
  lastName: string;

  @IsEmail()
  @IsOptional()
  @Transform(({ value }) => value?.toLowerCase().trim())
  email?: string;

  @IsEnum(LeadSource)
  source: LeadSource;

  @IsUUID()
  @IsOptional()
  ownerId?: string;
}
```

---

## Secrets Management

| Secret | Storage | Rotation |
|--------|---------|---------|
| JWT RS256 private key | Environment variable (Base64 PEM) | On deployment |
| Database connection string | Environment variable | Manual (with zero-downtime procedure) |
| Redis password | Environment variable | Manual |
| S3 credentials | Environment variable (or IAM role) | Automated via IAM |
| SMTP API key | Environment variable | Manual (annual) |
| Webhook signing secret | Per-tenant in DB (encrypted column) | User-initiated |
| Session secrets | Redis (ephmeral) | With JWT keypair rotation |

**No secrets in source code.** No secrets in Docker images. Environment validated at startup via Zod schema:

```typescript
// config/env.schema.ts
const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  JWT_PRIVATE_KEY: z.string().min(100),
  JWT_PUBLIC_KEY: z.string().min(100),
  S3_BUCKET: z.string(),
  S3_REGION: z.string(),
  // ...
});

export const config = envSchema.parse(process.env);
// Throws on startup if any required env var is missing or invalid
```

---

## Audit Log Architecture

### AuditLogInterceptor

```typescript
// common/interceptors/audit-log.interceptor.ts
@Injectable()
export class AuditLogInterceptor implements NestInterceptor {
  constructor(private auditService: AuditService) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const { method } = request;

    if (!['POST', 'PATCH', 'PUT', 'DELETE'].includes(method)) {
      return next.handle();
    }

    return next.handle().pipe(
      tap(async (responseBody) => {
        // Capture after-state from response
        await this.auditService.log({
          tenantId: request.tenantId,
          userId: request.user.id,
          entityType: resolveEntityType(request.route.path),
          entityId: responseBody?.id ?? request.params.id,
          action: resolveAction(method),
          after: sanitizePII(responseBody),
          ipAddress: request.ip,
          userAgent: request.headers['user-agent'],
        });
      }),
    );
  }
}
```

### PII Sanitization in Audit Log
Before/after state snapshots have PII masked:
```typescript
function sanitizePII(data: Record<string, unknown>): Record<string, unknown> {
  const PII_FIELDS = ['password', 'passwordHash', 'phone', 'email'];
  return Object.fromEntries(
    Object.entries(data).map(([k, v]) =>
      PII_FIELDS.includes(k) ? [k, '[REDACTED]'] : [k, v]
    )
  );
}
```

---

## File Upload Security

```typescript
// Validation pipeline for file uploads
const ALLOWED_MIME_TYPES = [
  'application/pdf',
  'application/vnd.openxmlformats-officedocument.wordprocessingml.document',  // docx
  'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',        // xlsx
  'image/png', 'image/jpeg',
  'text/csv',
];

const MAX_FILE_SIZE = 25 * 1024 * 1024; // 25MB

// Magic byte validation (not just Content-Type header)
import { fileTypeFromBuffer } from 'file-type';

async validateFile(buffer: Buffer, declaredMimeType: string): Promise<void> {
  const detected = await fileTypeFromBuffer(buffer);
  if (!detected || !ALLOWED_MIME_TYPES.includes(detected.mime)) {
    throw new BadRequestException('Unsupported file type');
  }
  if (detected.mime !== declaredMimeType) {
    throw new BadRequestException('MIME type mismatch');
  }
}
```

Files are stored with:
- **Random S3 key** (not the original filename) to prevent path traversal
- **Content-Disposition: attachment** on pre-signed URL to prevent XSS via browser rendering
- **Tenant quota check** before upload to enforce plan limits

---

## Security Testing Checklist

| Test Category | Scenario | Expected Result |
|--------------|----------|-----------------|
| Authentication | Login with invalid password | 401 |
| Authentication | Use expired access token | 401 |
| Authentication | Reuse revoked refresh token | 401 + all sessions revoked |
| Authentication | Tamper JWT signature | 401 |
| Authentication | Modify tenantId in JWT payload | 401 (signature invalid) |
| Authorization | Access route without required permission | 403 |
| Multi-tenancy | Request resource owned by different tenant | 404 |
| Multi-tenancy | Create resource with forged tenantId | Resource created in own tenant |
| Input validation | SQL injection attempt in string field | 422 (rejected by class-validator) |
| Input validation | XSS payload in text field | Stored as plain text, escaped on render |
| Rate limiting | 6 failed logins in 15 min | 429 on 6th attempt |
| File upload | Upload executable (.exe disguised as .pdf) | 422 (magic byte mismatch) |
| File upload | Upload 30MB file (limit 25MB) | 413 |
| CSRF | State-changing request without CSRF protection | SameSite=Strict cookie prevents |
