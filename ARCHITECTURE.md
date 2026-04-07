# PropFlow — Master Architecture Document

> **Purpose:** This file is the single source of truth for the entire PropFlow codebase. Every Claude Code session MUST read this file first before building any feature. If a feature implementation doc conflicts with this file, this file wins.

---

## What Is PropFlow

A SaaS bundle for UK estate and letting agents. One subscription replaces 4-6 separate tools (lead handling, onboarding, AML checks, document signing). Three pricing tiers: Starter (£79/mo), Professional (£149/mo), Agency (£249/mo). All hosted in AWS eu-west-2 (London).

---

## Tech Stack

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend | AWS Lambda + API Gateway | Serverless, Node.js 20.x runtime |
| Database | PostgreSQL 16 on AWS RDS | t3.micro to start, eu-west-2 |
| Storage | AWS S3 | Buckets: `propflow-emails-{env}`, `propflow-documents-{env}` |
| Frontend | React 18 + Vite + Tailwind CSS 3 | SPA, deployed to S3 + CloudFront |
| Auth | Clerk | Organisations = tenants, JWT custom claims |
| Payments | Stripe | Subscriptions + metered AML billing |
| SMS/WhatsApp | Twilio | UK numbers, standard WhatsApp (not Business API) |
| Calendar | Nylas API v3 | OAuth per agent |
| AML/ID | ComplyCube API | DIATF-certified, hosted verification flow |
| Doc Signing | Dropbox Sign (HelloSign) API | Templates + signature requests |
| AI | OpenAI GPT-4o-mini | Lead parsing + qualification dialogue |
| Email Ingestion | AWS SES + Lambda | Receives forwarded portal emails |
| IaC | Serverless Framework v4 | `serverless.yml` in `api/` |

---

## Project Structure

```
propflow/
├── ARCHITECTURE.md              ← THIS FILE — read first every session
├── docs/
│   └── implementation/          ← Feature implementation plans (01-10)
├── api/                         ← Backend (Lambda functions)
│   ├── serverless.yml
│   ├── package.json
│   └── src/
│       ├── handlers/            ← Lambda handler functions (one per endpoint/event)
│       │   ├── leads.js
│       │   ├── conversations.js
│       │   ├── viewings.js
│       │   ├── onboardings.js
│       │   ├── amlChecks.js
│       │   ├── signingRequests.js
│       │   ├── notifications.js
│       │   ├── billing.js
│       │   ├── tenants.js
│       │   └── webhooks/
│       │       ├── clerk.js
│       │       ├── twilio.js
│       │       ├── complycube.js
│       │       ├── hellosign.js
│       │       ├── stripe.js
│       │       └── ses.js
│       ├── lib/                 ← SHARED UTILITIES — see section below
│       │   ├── db.js
│       │   ├── queries.js
│       │   ├── tenantContext.js
│       │   ├── notifications.js
│       │   ├── twilio.js
│       │   ├── s3.js
│       │   ├── openai.js
│       │   ├── stripe.js
│       │   ├── complycube.js
│       │   ├── hellosign.js
│       │   ├── nylas.js
│       │   ├── errors.js
│       │   └── response.js
│       ├── middleware/
│       │   ├── auth.js          ← Clerk JWT verification + tenant context
│       │   └── planGating.js    ← Feature access by subscription plan
│       └── migrations/
│           └── 001_initial_schema.sql  ← ALL tables, created in session 1
├── web/                         ← Frontend (React SPA)
│   ├── package.json
│   ├── vite.config.js
│   ├── tailwind.config.js
│   ├── index.html
│   └── src/
│       ├── App.tsx
│       ├── main.tsx
│       ├── api/                 ← API client
│       │   └── client.ts        ← Axios/fetch wrapper with Clerk token
│       ├── components/
│       │   ├── layout/          ← Shell, Sidebar, TopBar
│       │   ├── leads/           ← LeadCard, LeadDetail, ConversationThread
│       │   ├── calendar/        ← CalendarView, ViewingCard
│       │   ├── onboarding/      ← OnboardingWizard, ProgressBar
│       │   ├── aml/             ← AMLStatus, AuditDownload
│       │   ├── documents/       ← SigningStatus, TemplateManager
│       │   ├── notifications/   ← NotificationPanel, NotificationItem
│       │   ├── billing/         ← PlanCard, UsageMeter
│       │   └── ui/              ← ScoreBadge, SourceBadge, ChatBubble, StatusPill
│       ├── pages/
│       │   ├── Dashboard.tsx
│       │   ├── LeadDetail.tsx
│       │   ├── Calendar.tsx
│       │   ├── Onboardings.tsx
│       │   ├── AMLChecks.tsx
│       │   ├── Documents.tsx
│       │   ├── Settings.tsx
│       │   └── public/          ← No-auth pages
│       │       ├── BookViewing.tsx
│       │       ├── OnboardingForm.tsx
│       │       └── AMLComplete.tsx
│       ├── hooks/
│       │   ├── useAuth.ts
│       │   ├── useLeads.ts
│       │   ├── useNotifications.ts
│       │   └── useTenant.ts
│       └── lib/
│           ├── constants.ts
│           └── formatters.ts
└── infra/                       ← CloudFormation / deployment scripts
    ├── s3-buckets.yml
    ├── ses-rules.yml
    └── rds.yml
```

---

## Conventions — Follow These Exactly

### IDs
- All primary keys: `UUID` via `gen_random_uuid()`
- Never use auto-incrementing integers

### Dates
- All timestamps: `TIMESTAMPTZ` stored in UTC
- API responses: ISO 8601 strings (`2026-04-02T14:30:00.000Z`)
- Frontend displays in user's local timezone

### API Response Format
Every API endpoint returns this shape:
```json
{
  "data": {},        // or [] for lists
  "error": null,     // or { "code": "NOT_FOUND", "message": "Lead not found" }
  "meta": {          // only for paginated lists
    "page": 1,
    "limit": 20,
    "total": 145
  }
}
```

### HTTP Status Codes
- 200: Success (GET, PATCH)
- 201: Created (POST)
- 204: No content (DELETE)
- 400: Validation error
- 401: Not authenticated
- 403: Not authorised (wrong tenant, wrong plan)
- 404: Not found
- 500: Internal error

### Error Handling
All handlers wrapped in try/catch. Use error classes from `lib/errors.js`:
```javascript
const { NotFoundError, ForbiddenError, ValidationError } = require('../lib/errors');

// In handler:
if (!lead) throw new NotFoundError('Lead not found');
if (tenant.plan === 'starter' && feature === 'doc_signing') throw new ForbiddenError('Requires Professional plan');
```

### Environment Variables
All loaded from `.env` via `dotenv`. Never hardcoded.
```
# AWS
AWS_REGION=eu-west-2
S3_EMAIL_BUCKET=propflow-emails-dev
S3_DOCUMENTS_BUCKET=propflow-documents-dev

# Database
DATABASE_URL=postgresql://propflow:password@localhost:5432/propflow_dev

# Clerk
CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...
CLERK_WEBHOOK_SECRET=whsec_...

# Stripe
STRIPE_SECRET_KEY=sk_test_...
STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Twilio
TWILIO_ACCOUNT_SID=AC...
TWILIO_AUTH_TOKEN=...
TWILIO_PHONE_NUMBER=+44...

# OpenAI
OPENAI_API_KEY=sk-...

# Nylas
NYLAS_CLIENT_ID=...
NYLAS_API_KEY=...
NYLAS_CALLBACK_URL=http://localhost:3000/auth/nylas/callback

# ComplyCube
COMPLYCUBE_API_KEY=...
COMPLYCUBE_WEBHOOK_SECRET=...

# Dropbox Sign
HELLOSIGN_API_KEY=...
HELLOSIGN_CLIENT_ID=...

# App
APP_URL=http://localhost:3000
API_URL=http://localhost:4000
NODE_ENV=development
```

### Naming
- Database tables: `snake_case` plural (`leads`, `aml_checks`, `signing_requests`)
- Database columns: `snake_case` (`tenant_id`, `created_at`, `enquirer_name`)
- JS variables/functions: `camelCase` (`tenantId`, `createLead`, `getConversationHistory`)
- API routes: `kebab-case` (`/api/aml-checks`, `/api/signing-requests`)
- React components: `PascalCase` (`LeadCard`, `ScoreBadge`)
- Files: `camelCase.js` (backend), `PascalCase.tsx` (React components)

### Tenant Isolation
**Every query that touches tenant data MUST include `WHERE tenant_id = $X`.** No exceptions. The `tenantContext` middleware extracts `tenantId` from the Clerk JWT and passes it to every handler. If you're writing a query and it doesn't have a `tenant_id` filter, it's a bug.

---

## Shared Utilities (`api/src/lib/`)

These are built in Session 1 and imported by every feature. **Never duplicate this logic in handlers — always import from `lib/`.**

### `lib/db.js` — Database Connection Pool
```javascript
const { Pool } = require('pg');

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000
});

async function query(text, params) {
  const result = await pool.query(text, params);
  return result.rows;
}

async function queryOne(text, params) {
  const rows = await query(text, params);
  return rows[0] || null;
}

async function transaction(callback) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const result = await callback(client);
    await client.query('COMMIT');
    return result;
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;
  } finally {
    client.release();
  }
}

module.exports = { query, queryOne, transaction, pool };
```

### `lib/queries.js` — Common Query Helpers
**Every feature needs these. Import from here — never write these queries inline.**
```javascript
const db = require('./db');
const { NotFoundError } = require('./errors');

async function getTenant(tenantId) {
  const tenant = await db.queryOne('SELECT * FROM tenants WHERE id = $1', [tenantId]);
  if (!tenant) throw new NotFoundError('Tenant not found');
  return tenant;
}

async function getLead(leadId, tenantId) {
  const lead = await db.queryOne(
    'SELECT * FROM leads WHERE id = $1 AND tenant_id = $2',
    [leadId, tenantId]
  );
  if (!lead) throw new NotFoundError('Lead not found');
  return lead;
}

async function getLeadWithAgent(leadId, tenantId) {
  const lead = await db.queryOne(
    `SELECT l.*, u.name as agent_name, u.email as agent_email, u.phone as agent_phone,
            u.nylas_grant_id, u.nylas_calendar_id
     FROM leads l
     LEFT JOIN users u ON l.assigned_agent_id = u.id
     WHERE l.id = $1 AND l.tenant_id = $2`,
    [leadId, tenantId]
  );
  if (!lead) throw new NotFoundError('Lead not found');
  return lead;
}

async function getUser(userId, tenantId) {
  const user = await db.queryOne(
    'SELECT * FROM users WHERE id = $1 AND tenant_id = $2',
    [userId, tenantId]
  );
  if (!user) throw new NotFoundError('User not found');
  return user;
}

async function getActiveAgents(tenantId) {
  return db.query(
    `SELECT * FROM users 
     WHERE tenant_id = $1 AND is_active = TRUE AND role IN ('agent', 'admin', 'owner')
     ORDER BY name`,
    [tenantId]
  );
}

async function getDefaultAgent(tenantId) {
  const agent = await db.queryOne(
    `SELECT * FROM users 
     WHERE tenant_id = $1 AND is_active = TRUE AND role = 'owner' 
     LIMIT 1`,
    [tenantId]
  );
  if (!agent) throw new NotFoundError('No active agent found');
  return agent;
}

async function getOnboardingByToken(token) {
  const onboarding = await db.queryOne(
    'SELECT * FROM onboardings WHERE token = $1 AND expires_at > NOW()',
    [token]
  );
  if (!onboarding) throw new NotFoundError('Invalid or expired onboarding link');
  return onboarding;
}

async function getConversationByPhone(phoneNumber) {
  return db.queryOne(
    `SELECT c.*, l.tenant_id, l.enquirer_name, l.property_address
     FROM conversations c
     JOIN leads l ON c.lead_id = l.id
     WHERE c.phone_number = $1 AND c.status = 'active'
     ORDER BY c.created_at DESC LIMIT 1`,
    [phoneNumber]
  );
}

async function getConversationHistory(conversationId) {
  return db.query(
    'SELECT * FROM messages WHERE conversation_id = $1 ORDER BY sent_at ASC',
    [conversationId]
  );
}

module.exports = {
  getTenant, getLead, getLeadWithAgent, getUser,
  getActiveAgents, getDefaultAgent,
  getOnboardingByToken, getConversationByPhone, getConversationHistory
};
```

### `lib/tenantContext.js` — Extract Tenant from JWT
```javascript
const { verifyToken } = require('@clerk/backend');

async function extractTenantContext(event) {
  const authHeader = event.headers?.authorization || event.headers?.Authorization;
  if (!authHeader?.startsWith('Bearer ')) {
    throw new UnauthorizedError('Missing auth token');
  }

  const token = authHeader.substring(7);
  const claims = await verifyToken(token, {
    secretKey: process.env.CLERK_SECRET_KEY
  });

  const tenantId = claims.org_id;
  const userId = claims.sub;
  const role = claims.org_role || 'agent';

  if (!tenantId) throw new UnauthorizedError('No organisation context');

  return { tenantId, userId, role };
}

module.exports = { extractTenantContext };
```

### `lib/notifications.js` — Create Notifications
```javascript
const db = require('./db');

async function createNotification(tenantId, userId, { type, leadId, title, body, priority = 'normal' }) {
  return db.queryOne(
    `INSERT INTO notifications (tenant_id, user_id, type, lead_id, title, body, priority)
     VALUES ($1, $2, $3, $4, $5, $6, $7) RETURNING *`,
    [tenantId, userId, type, leadId || null, title, body, priority]
  );
}

async function getUnreadNotifications(tenantId, userId) {
  return db.query(
    `SELECT * FROM notifications 
     WHERE tenant_id = $1 AND (user_id = $2 OR user_id IS NULL) AND read_at IS NULL
     ORDER BY created_at DESC LIMIT 50`,
    [tenantId, userId]
  );
}

async function markRead(notificationId, tenantId) {
  return db.queryOne(
    'UPDATE notifications SET read_at = NOW() WHERE id = $1 AND tenant_id = $2 RETURNING *',
    [notificationId, tenantId]
  );
}

async function markAllRead(tenantId, userId) {
  return db.query(
    `UPDATE notifications SET read_at = NOW() 
     WHERE tenant_id = $1 AND (user_id = $2 OR user_id IS NULL) AND read_at IS NULL`,
    [tenantId, userId]
  );
}

module.exports = { createNotification, getUnreadNotifications, markRead, markAllRead };
```

### `lib/twilio.js` — Twilio Client
```javascript
const twilio = require('twilio');

let client;

function getTwilioClient() {
  if (!client) {
    client = twilio(process.env.TWILIO_ACCOUNT_SID, process.env.TWILIO_AUTH_TOKEN);
  }
  return client;
}

async function sendSMS(to, body, from = process.env.TWILIO_PHONE_NUMBER) {
  const client = getTwilioClient();
  return client.messages.create({ to, body, from });
}

module.exports = { getTwilioClient, sendSMS };
```

### `lib/s3.js` — S3 Operations
```javascript
const { S3Client, PutObjectCommand, GetObjectCommand } = require('@aws-sdk/client-s3');
const { getSignedUrl } = require('@aws-sdk/s3-request-presigner');

const s3 = new S3Client({ region: process.env.AWS_REGION });

async function uploadFile(bucket, key, body, contentType) {
  await s3.send(new PutObjectCommand({
    Bucket: bucket,
    Key: key,
    Body: body,
    ContentType: contentType
  }));
  return key;
}

async function getFile(bucket, key) {
  const response = await s3.send(new GetObjectCommand({ Bucket: bucket, Key: key }));
  return response.Body;
}

async function getPresignedUrl(bucket, key, expiresIn = 3600) {
  const command = new GetObjectCommand({ Bucket: bucket, Key: key });
  return getSignedUrl(s3, command, { expiresIn });
}

module.exports = { s3, uploadFile, getFile, getPresignedUrl };
```

### `lib/openai.js` — OpenAI Client
```javascript
const OpenAI = require('openai');

let client;

function getOpenAIClient() {
  if (!client) {
    client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
  }
  return client;
}

async function chatCompletion(systemPrompt, userMessage, options = {}) {
  const openai = getOpenAIClient();
  const response = await openai.chat.completions.create({
    model: options.model || 'gpt-4o-mini',
    temperature: options.temperature ?? 0.7,
    max_tokens: options.maxTokens || 500,
    response_format: options.json ? { type: 'json_object' } : undefined,
    messages: [
      { role: 'system', content: systemPrompt },
      ...(Array.isArray(userMessage) ? userMessage : [{ role: 'user', content: userMessage }])
    ]
  });
  const content = response.choices[0].message.content;
  return options.json ? JSON.parse(content) : content;
}

module.exports = { getOpenAIClient, chatCompletion };
```

### `lib/stripe.js` — Stripe Client
```javascript
const Stripe = require('stripe');

let client;

function getStripeClient() {
  if (!client) {
    client = new Stripe(process.env.STRIPE_SECRET_KEY);
  }
  return client;
}

module.exports = { getStripeClient };
```

### `lib/errors.js` — Error Classes
```javascript
class AppError extends Error {
  constructor(message, statusCode, code) {
    super(message);
    this.statusCode = statusCode;
    this.code = code;
  }
}

class NotFoundError extends AppError {
  constructor(message = 'Not found') { super(message, 404, 'NOT_FOUND'); }
}

class ValidationError extends AppError {
  constructor(message = 'Validation failed') { super(message, 400, 'VALIDATION_ERROR'); }
}

class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') { super(message, 401, 'UNAUTHORIZED'); }
}

class ForbiddenError extends AppError {
  constructor(message = 'Forbidden') { super(message, 403, 'FORBIDDEN'); }
}

module.exports = { AppError, NotFoundError, ValidationError, UnauthorizedError, ForbiddenError };
```

### `lib/response.js` — Standard API Response
```javascript
function success(data, statusCode = 200, meta) {
  return {
    statusCode,
    headers: { 'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*' },
    body: JSON.stringify({ data, error: null, ...(meta ? { meta } : {}) })
  };
}

function error(err) {
  const statusCode = err.statusCode || 500;
  const code = err.code || 'INTERNAL_ERROR';
  const message = statusCode === 500 ? 'Internal server error' : err.message;
  
  if (statusCode === 500) console.error('Unhandled error:', err);
  
  return {
    statusCode,
    headers: { 'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*' },
    body: JSON.stringify({ data: null, error: { code, message } })
  };
}

module.exports = { success, error };
```

### `middleware/auth.js` — Auth Middleware Wrapper
```javascript
const { extractTenantContext } = require('../lib/tenantContext');
const { error } = require('../lib/response');
const { UnauthorizedError } = require('../lib/errors');

function withAuth(handler) {
  return async (event) => {
    try {
      const context = await extractTenantContext(event);
      event.tenantContext = context;
      return await handler(event, context);
    } catch (err) {
      if (err instanceof UnauthorizedError) return error(err);
      return error(err);
    }
  };
}

module.exports = { withAuth };
```

### `middleware/planGating.js` — Plan-Based Feature Access
```javascript
const db = require('../lib/db');
const { ForbiddenError } = require('../lib/errors');

const PLAN_FEATURES = {
  starter: ['lead_autopilot', 'basic_onboarding', 'aml_checks', 'billing'],
  professional: ['lead_autopilot', 'onboarding', 'aml_checks', 'doc_signing', 'billing'],
  agency: ['lead_autopilot', 'onboarding', 'aml_checks', 'doc_signing', 'white_label', 'billing']
};

async function requireFeature(tenantId, feature) {
  const tenant = await db.queryOne('SELECT plan FROM tenants WHERE id = $1', [tenantId]);
  if (!tenant) throw new ForbiddenError('Tenant not found');
  
  const allowed = PLAN_FEATURES[tenant.plan] || [];
  if (!allowed.includes(feature)) {
    throw new ForbiddenError(`Feature "${feature}" requires a higher plan`);
  }
}

module.exports = { requireFeature, PLAN_FEATURES };
```

### Handler Pattern — Every Handler Follows This
```javascript
const { withAuth } = require('../middleware/auth');
const { success, error } = require('../lib/response');
const db = require('../lib/db');

module.exports.list = withAuth(async (event, { tenantId }) => {
  try {
    const items = await db.query(
      'SELECT * FROM some_table WHERE tenant_id = $1 ORDER BY created_at DESC',
      [tenantId]
    );
    return success(items);
  } catch (err) {
    return error(err);
  }
});
```

---

## Complete Database Schema

**All tables are created in a single migration file: `api/src/migrations/001_initial_schema.sql`.** This runs once during Session 1. Subsequent sessions do NOT create tables — they use what already exists.

```sql
-- ============================================================
-- PropFlow — Complete Database Schema
-- Run once during initial setup
-- ============================================================

-- Enable UUID generation
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- ============================================================
-- CORE: Tenants & Users
-- ============================================================

CREATE TABLE tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  slug VARCHAR(100) UNIQUE NOT NULL,
  plan VARCHAR(50) NOT NULL DEFAULT 'professional',
  logo_url VARCHAR(500),
  brand_colour VARCHAR(7) DEFAULT '#1F2937',
  
  -- Stripe
  stripe_customer_id VARCHAR(255),
  stripe_subscription_id VARCHAR(255),
  
  -- AML usage tracking
  aml_checks_used INTEGER DEFAULT 0,
  aml_checks_limit INTEGER DEFAULT 10,
  
  -- Twilio
  twilio_phone_number VARCHAR(50),
  
  -- Settings
  settings JSONB DEFAULT '{}',
  is_active BOOLEAN DEFAULT TRUE,
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  clerk_user_id VARCHAR(255) UNIQUE NOT NULL,
  email VARCHAR(255) NOT NULL,
  name VARCHAR(255) NOT NULL,
  phone VARCHAR(50),
  role VARCHAR(50) NOT NULL DEFAULT 'agent',
  
  -- Nylas calendar
  nylas_grant_id VARCHAR(255),
  nylas_calendar_id VARCHAR(255),
  
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_clerk ON users(clerk_user_id);

-- ============================================================
-- EMAIL PARSING: Tenant email mappings
-- ============================================================

CREATE TABLE tenant_email_mappings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  forwarding_address VARCHAR(255) UNIQUE NOT NULL,
  source_email VARCHAR(255),
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_email_mappings_address ON tenant_email_mappings(forwarding_address);

-- ============================================================
-- LEADS
-- ============================================================

CREATE TABLE leads (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  source VARCHAR(50) NOT NULL,
  source_email_s3_key VARCHAR(500),
  
  -- Enquirer
  enquirer_name VARCHAR(255),
  enquirer_email VARCHAR(255),
  enquirer_phone VARCHAR(50),
  enquirer_message TEXT,
  
  -- Property
  property_address TEXT,
  property_postcode VARCHAR(20),
  property_price INTEGER,
  property_url VARCHAR(500),
  property_portal_ref VARCHAR(100),
  
  -- Qualification
  score VARCHAR(20) DEFAULT 'unscored',
  qualification_status VARCHAR(50) DEFAULT 'pending',
  qualification_data JSONB DEFAULT '{}',
  
  -- Cross-sell
  cross_sell_detected BOOLEAN DEFAULT FALSE,
  
  -- Assignment
  assigned_agent_id UUID REFERENCES users(id),
  
  -- Timestamps
  received_at TIMESTAMPTZ DEFAULT NOW(),
  first_response_at TIMESTAMPTZ,
  qualified_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_leads_tenant ON leads(tenant_id);
CREATE INDEX idx_leads_tenant_score ON leads(tenant_id, score);
CREATE INDEX idx_leads_tenant_status ON leads(tenant_id, qualification_status);
CREATE INDEX idx_leads_received ON leads(received_at DESC);
CREATE INDEX idx_leads_dedup ON leads(tenant_id, enquirer_email, property_url);

-- ============================================================
-- CONVERSATIONS & MESSAGES
-- ============================================================

CREATE TABLE conversations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  lead_id UUID NOT NULL REFERENCES leads(id),
  channel VARCHAR(20) NOT NULL,
  twilio_sid VARCHAR(100),
  phone_number VARCHAR(50) NOT NULL,
  status VARCHAR(50) DEFAULT 'active',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_conversations_lead ON conversations(lead_id);
CREATE INDEX idx_conversations_phone ON conversations(phone_number, status);

CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID NOT NULL REFERENCES conversations(id),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  direction VARCHAR(10) NOT NULL,
  channel VARCHAR(20) NOT NULL,
  body TEXT NOT NULL,
  twilio_message_sid VARCHAR(100),
  sent_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_messages_conversation ON messages(conversation_id);

-- ============================================================
-- VIEWINGS
-- ============================================================

CREATE TABLE viewings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  lead_id UUID NOT NULL REFERENCES leads(id),
  agent_id UUID NOT NULL REFERENCES users(id),
  property_address TEXT NOT NULL,
  
  starts_at TIMESTAMPTZ NOT NULL,
  ends_at TIMESTAMPTZ NOT NULL,
  duration_minutes INTEGER DEFAULT 30,
  
  nylas_event_id VARCHAR(255),
  
  status VARCHAR(30) DEFAULT 'confirmed',
  cancelled_by VARCHAR(20),
  cancellation_reason TEXT,
  
  attendee_name VARCHAR(255) NOT NULL,
  attendee_phone VARCHAR(50),
  attendee_email VARCHAR(255),
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_viewings_tenant ON viewings(tenant_id);
CREATE INDEX idx_viewings_agent_date ON viewings(agent_id, starts_at);
CREATE INDEX idx_viewings_lead ON viewings(lead_id);

-- ============================================================
-- NOTIFICATIONS
-- ============================================================

CREATE TABLE notifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  user_id UUID REFERENCES users(id),
  type VARCHAR(50) NOT NULL,
  lead_id UUID REFERENCES leads(id),
  title VARCHAR(255) NOT NULL,
  body TEXT,
  priority VARCHAR(20) DEFAULT 'normal',
  read_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_notifications_user_unread ON notifications(user_id, read_at) WHERE read_at IS NULL;
CREATE INDEX idx_notifications_tenant ON notifications(tenant_id);

-- ============================================================
-- ONBOARDING
-- ============================================================

CREATE TABLE onboardings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  lead_id UUID REFERENCES leads(id),
  
  token VARCHAR(255) UNIQUE NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL,
  
  client_type VARCHAR(20) NOT NULL,
  
  status VARCHAR(30) DEFAULT 'invited',
  current_step INTEGER DEFAULT 0,
  total_steps INTEGER DEFAULT 5,
  completed_at TIMESTAMPTZ,
  last_activity_at TIMESTAMPTZ,
  
  chase_count INTEGER DEFAULT 0,
  next_chase_at TIMESTAMPTZ,
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_onboardings_token ON onboardings(token);
CREATE INDEX idx_onboardings_tenant ON onboardings(tenant_id);
CREATE INDEX idx_onboardings_lead ON onboardings(lead_id);

CREATE TABLE onboarding_data (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  onboarding_id UUID NOT NULL REFERENCES onboardings(id),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  section VARCHAR(50) NOT NULL,
  data JSONB NOT NULL DEFAULT '{}',
  completed BOOLEAN DEFAULT FALSE,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_onboarding_data_onboarding ON onboarding_data(onboarding_id);

-- ============================================================
-- AML CHECKS
-- ============================================================

CREATE TABLE aml_checks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  lead_id UUID REFERENCES leads(id),
  onboarding_id UUID REFERENCES onboardings(id),
  
  client_name VARCHAR(255) NOT NULL,
  client_email VARCHAR(255),
  client_phone VARCHAR(50),
  client_dob DATE,
  
  complycube_client_id VARCHAR(255),
  complycube_check_id VARCHAR(255),
  complycube_session_token VARCHAR(500),
  
  status VARCHAR(30) DEFAULT 'pending',
  id_result VARCHAR(30),
  pep_result VARCHAR(30),
  sanctions_result VARCHAR(30),
  result_summary JSONB DEFAULT '{}',
  
  audit_pdf_s3_key VARCHAR(500),
  completed_at TIMESTAMPTZ,
  expires_at TIMESTAMPTZ,
  
  is_billable BOOLEAN DEFAULT TRUE,
  billed BOOLEAN DEFAULT FALSE,
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_aml_checks_tenant ON aml_checks(tenant_id);
CREATE INDEX idx_aml_checks_lead ON aml_checks(lead_id);
CREATE INDEX idx_aml_checks_status ON aml_checks(tenant_id, status);
CREATE INDEX idx_aml_checks_complycube ON aml_checks(complycube_client_id);

-- ============================================================
-- DOCUMENT SIGNING
-- ============================================================

CREATE TABLE document_templates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  name VARCHAR(255) NOT NULL,
  description TEXT,
  category VARCHAR(50),
  hellosign_template_id VARCHAR(255),
  fields JSONB DEFAULT '[]',
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_doc_templates_tenant ON document_templates(tenant_id);

CREATE TABLE signing_requests (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  lead_id UUID REFERENCES leads(id),
  template_id UUID REFERENCES document_templates(id),
  
  hellosign_signature_request_id VARCHAR(255),
  
  title VARCHAR(255) NOT NULL,
  document_url VARCHAR(500),
  
  status VARCHAR(30) DEFAULT 'sent',
  
  signed_pdf_s3_key VARCHAR(500),
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  completed_at TIMESTAMPTZ,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_signing_requests_tenant ON signing_requests(tenant_id);
CREATE INDEX idx_signing_requests_lead ON signing_requests(lead_id);
CREATE INDEX idx_signing_requests_hs ON signing_requests(hellosign_signature_request_id);

CREATE TABLE signers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  signing_request_id UUID NOT NULL REFERENCES signing_requests(id),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  
  name VARCHAR(255) NOT NULL,
  email VARCHAR(255) NOT NULL,
  phone VARCHAR(50),
  role VARCHAR(50) NOT NULL,
  order_index INTEGER DEFAULT 0,
  
  status VARCHAR(30) DEFAULT 'pending',
  hellosign_signature_id VARCHAR(255),
  signed_at TIMESTAMPTZ,
  
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_signers_request ON signers(signing_request_id);
CREATE INDEX idx_signers_hs ON signers(hellosign_signature_id);

-- ============================================================
-- UPDATED_AT TRIGGER (auto-update on all tables)
-- ============================================================

CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply to all tables with updated_at
CREATE TRIGGER trg_tenants_updated BEFORE UPDATE ON tenants FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER trg_users_updated BEFORE UPDATE ON users FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER trg_leads_updated BEFORE UPDATE ON leads FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER trg_conversations_updated BEFORE UPDATE ON conversations FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER trg_viewings_updated BEFORE UPDATE ON viewings FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER trg_onboardings_updated BEFORE UPDATE ON onboardings FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER trg_aml_checks_updated BEFORE UPDATE ON aml_checks FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER trg_doc_templates_updated BEFORE UPDATE ON document_templates FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER trg_signing_requests_updated BEFORE UPDATE ON signing_requests FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

---

## Dependency Graph (Build Order)

```
Session 1:  [1. Auth + Schema + Shared Utils]
                    │
Session 2:  [2. Email Parser] → [3. AI Lead AutoPilot]
                                        │
Session 3:                      [4. Lead Scoring & Routing]
                                        │
Session 4:                      [5. Viewing Booking]
                                        │
Session 5:                      [6. Agent Dashboard]  (frontend, consumes APIs from 1-5)
                                        │
Session 6:                      [7. Digital Onboarding]
                                        │
Session 7:                      [8. AML Checks] + [10. Stripe Billing]  (build together)
                                        │
Session 8:                      [9. Document Signing]
                                        │
Session 9:                      Integration testing + fixes
```

---

## Session Instructions for Claude Code

Paste this at the start of every Claude Code session:

```
Read ARCHITECTURE.md first — it contains the full database schema, shared utilities, 
project structure, and conventions. All code must follow the patterns defined there.

Then read docs/implementation/0X-feature-name.md for this session's feature.

Rules:
1. Import shared utilities from api/src/lib/ — never duplicate them.
2. Use the database tables exactly as defined in ARCHITECTURE.md — never create new tables 
   or alter existing columns without noting it.
3. Every database query must include tenant_id filtering.
4. Every handler must use the withAuth wrapper and the standard response format.
5. Run the app after each major step to verify nothing is broken.
6. If you need a utility function that doesn't exist in lib/, add it to the appropriate 
   lib/ file — not inline in the handler.
```

---

## API Route Map (Complete)

```
# Auth (handled by Clerk)
POST   /api/webhooks/clerk              → webhooks/clerk.js

# Leads
GET    /api/leads                       → handlers/leads.list
GET    /api/leads/:id                   → handlers/leads.get
PATCH  /api/leads/:id                   → handlers/leads.update
GET    /api/leads/:id/messages          → handlers/conversations.listMessages
POST   /api/leads/:id/messages          → handlers/conversations.sendMessage

# Viewings
GET    /api/viewings                    → handlers/viewings.list
POST   /api/viewings                    → handlers/viewings.create
PATCH  /api/viewings/:id                → handlers/viewings.update
GET    /api/bookings/availability       → handlers/viewings.availability  (public)
POST   /api/bookings                    → handlers/viewings.book          (public)

# Notifications
GET    /api/notifications               → handlers/notifications.list
PATCH  /api/notifications/:id/read      → handlers/notifications.markRead
PATCH  /api/notifications/read-all      → handlers/notifications.markAllRead

# Onboarding
GET    /api/onboardings                 → handlers/onboardings.list
POST   /api/onboardings                 → handlers/onboardings.create
GET    /api/onboardings/:id             → handlers/onboardings.get
GET    /api/onboard/:token              → handlers/onboardings.getPublic   (public)
PATCH  /api/onboard/:token/sections/:s  → handlers/onboardings.saveSection (public)

# AML Checks
GET    /api/aml-checks                  → handlers/amlChecks.list
POST   /api/aml-checks                  → handlers/amlChecks.create
GET    /api/aml-checks/:id              → handlers/amlChecks.get
GET    /api/aml-checks/:id/audit-pdf    → handlers/amlChecks.downloadAudit
POST   /api/webhooks/complycube         → webhooks/complycube.js

# Document Signing
GET    /api/templates                   → handlers/signingRequests.listTemplates
POST   /api/templates                   → handlers/signingRequests.createTemplate
GET    /api/signing-requests            → handlers/signingRequests.list
POST   /api/signing-requests            → handlers/signingRequests.create
GET    /api/signing-requests/:id        → handlers/signingRequests.get
POST   /api/webhooks/hellosign          → webhooks/hellosign.js

# Billing
POST   /api/billing/checkout            → handlers/billing.createCheckout
POST   /api/billing/portal              → handlers/billing.createPortal
GET    /api/billing/usage               → handlers/billing.getUsage
POST   /api/webhooks/stripe             → webhooks/stripe.js

# Tenant / Settings
GET    /api/tenant                      → handlers/tenants.get
PATCH  /api/tenant                      → handlers/tenants.update
GET    /api/tenant/agents               → handlers/tenants.listAgents
POST   /api/tenant/agents/invite        → handlers/tenants.inviteAgent

# Webhooks (no auth — signature verified)
POST   /api/webhooks/twilio/sms         → webhooks/twilio.js
POST   /api/webhooks/ses                → webhooks/ses.js
```
