# Multi-Tenant Architecture — OpsNext CRM

---

## Multi-Tenancy Strategy

OpsNext CRM uses **row-level multi-tenancy**: all tenants share a single database and schema. Every tenant-scoped table carries a `tenantId UUID NOT NULL` column. Two enforcement layers prevent cross-tenant data access:

1. **Application layer**: Prisma client extension that automatically appends `WHERE tenantId = ?` to every query.
2. **Database layer**: PostgreSQL Row-Level Security (RLS) policies as a defence-in-depth backstop.

### Rationale for Row-Level vs. Schema-per-Tenant

| Dimension | Row-Level (chosen) | Schema-per-Tenant |
|-----------|-------------------|-------------------|
| Migration complexity | Single migration applies to all tenants | Fan-out: run migration per tenant schema |
| Connection pool | Single pool shared across all tenants | Pool per schema or connection-string per tenant |
| Query routing | Automatic via tenantId filter | Connection selection per request |
| Tenant count limit | Unlimited (thousands) | Practical limit ~hundreds (connection overhead) |
| Cross-tenant analytics | Simple SQL across tenant dimension | Requires UNION across schemas |
| RLS support | Native PostgreSQL RLS | Works but more complex |
| Risk | Logic bug exposes cross-tenant data | Schema isolation prevents it |
| Mitigation | PostgreSQL RLS as backstop | N/A |

---

## Tenant Isolation Layers

```
HTTP Request
     │
     ▼
┌────────────────────────────────────────────────────────────────┐
│  Layer 1: JWT Authentication                                   │
│  • JWT contains: { userId, tenantId, roleId, permissions }     │
│  • Token signed with RS256 private key                         │
│  • Invalid/expired token → 401, request rejected               │
└────────────────────────────────────────────────────────────────┘
     │
     ▼
┌────────────────────────────────────────────────────────────────┐
│  Layer 2: TenantMiddleware                                     │
│  • Reads tenantId from JWT claims                              │
│  • Attaches req.tenantId                                       │
│  • All downstream services read tenantId from request context  │
└────────────────────────────────────────────────────────────────┘
     │
     ▼
┌────────────────────────────────────────────────────────────────┐
│  Layer 3: Prisma Tenant Extension (Application-level RLS)      │
│  • Prisma client extended with a $extends middleware           │
│  • ALL queries automatically include WHERE tenantId = ?        │
│  • Cannot be bypassed by service code                          │
│  • Raw $queryRaw calls are prohibited in service layer         │
└────────────────────────────────────────────────────────────────┘
     │
     ▼
┌────────────────────────────────────────────────────────────────┐
│  Layer 4: PostgreSQL Row-Level Security (Database-level RLS)   │
│  • RLS policies on all tenant-scoped tables                    │
│  • DB connection uses a role that can only see its tenant rows │
│  • Even a SQL injection that bypasses Prisma cannot read       │
│    another tenant's data                                       │
└────────────────────────────────────────────────────────────────┘
```

---

## Implementation: Prisma Tenant Extension

The `PrismaService` creates a base Prisma client. A tenant-scoped client is created per-request using `$extends`.

```typescript
// prisma/prisma.service.ts
@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  async onModuleInit() {
    await this.$connect();
  }

  forTenant(tenantId: string): TenantPrismaClient {
    return this.$extends({
      query: {
        $allModels: {
          async findMany({ args, query }) {
            args.where = { ...args.where, tenantId };
            return query(args);
          },
          async findFirst({ args, query }) {
            args.where = { ...args.where, tenantId };
            return query(args);
          },
          async findUniqueOrThrow({ args, query }) {
            args.where = { ...args.where, tenantId };
            return query(args);
          },
          async create({ args, query }) {
            args.data = { ...args.data, tenantId };
            return query(args);
          },
          async update({ args, query }) {
            args.where = { ...args.where, tenantId };
            return query(args);
          },
          async delete({ args, query }) {
            args.where = { ...args.where, tenantId };
            return query(args);
          },
          async updateMany({ args, query }) {
            args.where = { ...args.where, tenantId };
            return query(args);
          },
          async deleteMany({ args, query }) {
            args.where = { ...args.where, tenantId };
            return query(args);
          },
        },
      },
    }) as TenantPrismaClient;
  }
}
```

### Usage in Services

```typescript
// lead.service.ts
@Injectable()
export class LeadService {
  constructor(
    private readonly prisma: PrismaService,
    @Inject(REQUEST) private readonly request: Request,
  ) {}

  private get db() {
    return this.prisma.forTenant(this.request.tenantId);
  }

  async findAll(filters: LeadFilterDto) {
    // tenantId automatically injected — cannot query other tenants
    return this.db.lead.findMany({
      where: buildLeadWhere(filters),
      orderBy: { createdAt: 'desc' },
    });
  }
}
```

---

## Implementation: PostgreSQL Row-Level Security

RLS policies are applied as a defence-in-depth layer. The API connects to PostgreSQL as a non-superuser role (`opsnext_app`). The policy enforces that the role can only access rows matching its `app.current_tenant_id` setting.

### Migration: Enable RLS

```sql
-- Applied via Prisma migration (raw SQL block)

-- Enable RLS on all tenant-scoped tables
ALTER TABLE tenants ENABLE ROW LEVEL SECURITY;
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE leads ENABLE ROW LEVEL SECURITY;
ALTER TABLE contacts ENABLE ROW LEVEL SECURITY;
ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;
ALTER TABLE opportunities ENABLE ROW LEVEL SECURITY;
ALTER TABLE activities ENABLE ROW LEVEL SECURITY;
ALTER TABLE tasks ENABLE ROW LEVEL SECURITY;
ALTER TABLE emails ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;
-- ... (all tenant-scoped tables)

-- Policy: rows visible only when tenantId matches session variable
CREATE POLICY tenant_isolation ON leads
  USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- Repeat for each table

-- Force RLS even for table owner (belt-and-suspenders)
ALTER TABLE leads FORCE ROW LEVEL SECURITY;
```

### Setting Tenant Context per Request

```typescript
// prisma.service.ts — within forTenant()
// After creating the extended client, set the PostgreSQL session variable
await this.$executeRaw`SELECT set_config('app.current_tenant_id', ${tenantId}, true)`;
```

The `true` parameter makes this a transaction-scoped setting (resets after transaction ends).

---

## Tenant Provisioning Flow

When a new tenant signs up, the `TenantProvisioningService` executes an atomic transaction:

```
POST /api/v1/auth/register (new tenant signup)
  │
  ▼
TenantProvisioningService.provision(dto)
  │
  ├── 1. Create Tenant record
  │       { name, slug, plan: 'free', status: 'active' }
  │
  ├── 2. Create Admin User
  │       { email, passwordHash, firstName, lastName, status: 'pending_invite', tenantId }
  │
  ├── 3. Create 5 System Roles (scoped to this tenant)
  │       super_admin, admin, manager, sales_rep, viewer
  │       (each with predefined permission sets)
  │
  ├── 4. Assign Admin User → super_admin role
  │
  ├── 5. Create default Pipeline
  │       { name: 'Sales Pipeline', isDefault: true }
  │       Stages: Prospecting (10%) → Qualification (25%) →
  │               Discovery (40%) → Proposal (60%) →
  │               Negotiation (80%) → Closed Won (100%) / Closed Lost (0%)
  │
  ├── 6. (All steps in a single Prisma transaction — atomic)
  │
  ├── 7. Emit TenantProvisioned event
  │
  └── 8. [Event handler] Queue welcome email → Admin User
```

---

## Plan-Based Feature Gating

The tenant's subscription plan gates access to features at the API layer.

```typescript
// common/guards/plan-gate.guard.ts
@Injectable()
export class PlanGateGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private tenantService: TenantService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const requiredPlan = this.reflector.get<Plan>('requiredPlan', context.getHandler());
    if (!requiredPlan) return true;

    const { tenantId } = context.switchToHttp().getRequest();
    const tenant = await this.tenantService.findById(tenantId);

    return PLAN_HIERARCHY[tenant.plan] >= PLAN_HIERARCHY[requiredPlan];
  }
}

// Plan hierarchy: free < starter < growth < enterprise
const PLAN_HIERARCHY: Record<Plan, number> = {
  free: 0, starter: 1, growth: 2, enterprise: 3,
};
```

### Usage on Routes
```typescript
@Get('analytics/advanced')
@RequiresPlan(Plan.GROWTH)          // Blocks free/starter tenants
@Permissions('reports:read')
getAdvancedAnalytics() { ... }
```

### Plan Limits

| Limit | Free | Starter | Growth | Enterprise |
|-------|------|---------|--------|------------|
| Users | 3 | 10 | 50 | Unlimited |
| Contacts | 500 | 5,000 | 50,000 | Unlimited |
| Custom pipelines | 1 | 3 | 10 | Unlimited |
| Custom fields | 5 | 20 | Unlimited | Unlimited |
| API rate limit (req/min) | 100 | 500 | 2,000 | 10,000 |
| Storage (GB) | 1 | 10 | 50 | 250 |
| Webhooks | 0 | 2 | 10 | Unlimited |
| Scheduled reports | No | No | Yes | Yes |

---

## Tenant-Aware Caching Strategy

Redis keys are always prefixed with `tenant:{tenantId}:` to prevent cache key collisions across tenants.

```typescript
// common/services/cache.service.ts
async get<T>(tenantId: string, key: string): Promise<T | null> {
  return this.redis.get(`tenant:${tenantId}:${key}`);
}

async set(tenantId: string, key: string, value: unknown, ttlSeconds: number): Promise<void> {
  await this.redis.setex(`tenant:${tenantId}:${key}`, ttlSeconds, JSON.stringify(value));
}

async invalidate(tenantId: string, keyPattern: string): Promise<void> {
  const keys = await this.redis.keys(`tenant:${tenantId}:${keyPattern}`);
  if (keys.length) await this.redis.del(...keys);
}
```

### Cache Invalidation Rules
| Cache Key | TTL | Invalidated When |
|-----------|-----|-----------------|
| `pipeline:stages:{pipelineId}` | 5 min | Stage created/updated/deleted |
| `roles:{roleId}:permissions` | 5 min | Role permissions updated |
| `user:{userId}:permissions` | 5 min | User role changed |
| `dashboard:executive:{filters}` | 5 min | Any opportunity/lead change |
| `plan:{tenantId}` | 10 min | Plan upgraded/downgraded |

---

## Tenant Data Isolation Verification

Security tests must cover these scenarios:

1. **Cross-tenant read**: User in Tenant A requests `/api/v1/leads/{leadId}` where the lead belongs to Tenant B. Expected: 404 (not 403 — do not reveal existence).
2. **Cross-tenant write**: User in Tenant A attempts to update a contact belonging to Tenant B. Expected: 404.
3. **JWT manipulation**: Attacker modifies the tenantId claim in a JWT (invalidates signature). Expected: 401.
4. **RLS bypass attempt**: Direct DB query without app.current_tenant_id set. Expected: no rows returned (RLS policy blocks).
5. **Prisma bypass attempt**: Service calls `$queryRaw` without tenantId filter. Expected: blocked by code review policy + linting rule.
