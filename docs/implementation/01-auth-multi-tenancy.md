# PropFlow — Implementation Plan: Auth & Multi-tenancy

## PropFlow Context

PropFlow is a SaaS bundle for UK estate and letting agents, consolidating AI lead handling, digital onboarding, AML checks, and document signing into a single £149/month subscription. This document is one of 10 feature implementation plans that together form the complete MVP. Each feature is designed to be built incrementally using Claude Code, with clear boundaries to avoid overloading any single build session.

**Tech stack:** AWS Lambda + API Gateway (serverless), PostgreSQL (RDS t3.micro), React + Tailwind (frontend), Clerk (auth), Stripe (billing), Twilio (comms), S3 (storage). All hosted in AWS eu-west-2 (London).

**Build order:** Auth & Multi-tenancy is **Feature 1 of 10** — the foundation layer. Everything else depends on this being in place first.

**Other features in the system:** Email Parser, AI Lead AutoPilot, Lead Scoring & Routing, Viewing Booking, Agent Dashboard, Digital Onboarding, ID & AML Checks, Document Signing, Stripe Billing.

---

## Feature Overview

Isolated multi-tenant architecture where each estate agency is a separate tenant. Agents within an agency log in via Clerk. All data is scoped to a tenant — no cross-tenant data leakage.

## Build Sequence

### Step 1: Project Scaffolding
- Initialise monorepo structure:
  ```
  propflow/
  ├── api/              # Lambda functions
  │   ├── src/
  │   │   ├── handlers/
  │   │   ├── middleware/
  │   │   ├── models/
  │   │   └── utils/
  │   ├── serverless.yml
  │   └── package.json
  ├── web/              # React frontend
  │   ├── src/
  │   │   ├── components/
  │   │   ├── pages/
  │   │   ├── hooks/
  │   │   ├── lib/
  │   │   └── App.tsx
  │   ├── tailwind.config.js
  │   └── package.json
  ├── shared/           # Shared types, constants
  └── infra/            # CloudFormation / SST
  ```
- Set up Serverless Framework or SST for Lambda deployment
- Configure `serverless.yml` with eu-west-2 region, environment variables

### Step 2: Database Schema (Core Tables)
```sql
-- Tenants (agencies)
CREATE TABLE tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  slug VARCHAR(100) UNIQUE NOT NULL,
  plan VARCHAR(50) NOT NULL DEFAULT 'professional', -- starter/professional/agency
  stripe_customer_id VARCHAR(255),
  stripe_subscription_id VARCHAR(255),
  aml_checks_used INTEGER DEFAULT 0,
  aml_checks_limit INTEGER DEFAULT 10,
  settings JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Users (agents within a tenant)
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  clerk_user_id VARCHAR(255) UNIQUE NOT NULL,
  email VARCHAR(255) NOT NULL,
  name VARCHAR(255) NOT NULL,
  role VARCHAR(50) NOT NULL DEFAULT 'agent', -- owner/admin/agent
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_clerk ON users(clerk_user_id);
```

### Step 3: Clerk Integration
- Create Clerk application (production + development instances)
- Configure:
  - Email + password sign-in
  - Organisation feature enabled (maps to tenant)
  - Custom claims in JWT: `tenant_id`, `role`
- Clerk webhook endpoint (`/api/webhooks/clerk`):
  - `user.created` → create user record, link to tenant
  - `organization.created` → create tenant record
  - `organization.membership.created` → link user to tenant

### Step 4: Tenant Middleware
```javascript
// middleware/tenantContext.js
// Extracts tenant_id from Clerk JWT on every API request
// Attaches to request context
// All downstream queries MUST filter by tenant_id

async function tenantMiddleware(event) {
  const token = extractBearerToken(event);
  const claims = await clerk.verifyToken(token);
  const tenantId = claims.org_id; // Clerk org = PropFlow tenant
  
  // Fetch tenant record, check active subscription
  const tenant = await db.query('SELECT * FROM tenants WHERE id = $1', [tenantId]);
  if (!tenant || !tenant.is_active) throw new UnauthorizedError();
  
  return { tenantId, userId: claims.sub, role: claims.role };
}
```

### Step 5: Row-Level Security
- All tables that hold tenant data MUST include `tenant_id UUID NOT NULL REFERENCES tenants(id)`
- Every query MUST include `WHERE tenant_id = $1`
- Consider PostgreSQL RLS policies as a safety net:
```sql
ALTER TABLE leads ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON leads
  USING (tenant_id = current_setting('app.tenant_id')::uuid);
```

### Step 6: Frontend Auth Shell
- Install `@clerk/clerk-react`
- Wrap app in `<ClerkProvider>`
- Routes:
  - `/sign-in` → Clerk sign-in component
  - `/sign-up` → Clerk sign-up + org creation flow
  - `/dashboard/*` → protected routes (redirect if unauthenticated)
- Sidebar layout component with tenant name, agent name, navigation
- API client wrapper that auto-attaches Clerk session token to all requests

### Step 7: Invite Flow
- Owner can invite agents by email
- Clerk handles invite email + account creation
- On accept → webhook fires → user record created in DB linked to tenant
- Roles: `owner` (full access), `admin` (manage agents, settings), `agent` (lead handling only)

## Environment Variables Required
```
CLERK_PUBLISHABLE_KEY=
CLERK_SECRET_KEY=
CLERK_WEBHOOK_SECRET=
DATABASE_URL=postgresql://...
AWS_REGION=eu-west-2
```

## Testing Checklist
- [ ] Sign up creates tenant + user
- [ ] Sign in returns valid JWT with tenant_id
- [ ] API requests without token return 401
- [ ] API requests with token from Tenant A cannot access Tenant B data
- [ ] Invite flow creates user linked to correct tenant
- [ ] Clerk webhook idempotency (duplicate events don't create duplicate records)
- [ ] Expired/revoked tokens are rejected

## Dependencies on Other Features
- **None** — this is the foundation. All other features depend on this.

## What This Feature Provides to Others
- `tenantId` available on every authenticated API request
- `userId` and `role` for permission checks
- Shared database connection and query patterns
- Frontend auth wrapper and protected route structure
- API client with automatic token attachment
