# CLAUDE.md — Rules for Building PropFlow

> Claude Code reads this file automatically. These rules are non-negotiable.

## Before Writing Any Code

1. Read `ARCHITECTURE.md` in full — it contains the database schema, shared utilities, project structure, and conventions.
2. Read the relevant feature implementation doc from `docs/implementation/`.
3. Do NOT start coding until you have read both files.

## Shared Utilities — USE THEM, DO NOT REBUILD THEM

The `api/src/lib/` directory contains shared utilities that are used across every feature. **You MUST import from these files. You MUST NOT duplicate their functionality.**

### Mandatory Imports

| When you need to... | Import from | Function |
|---------------------|-------------|----------|
| Run a database query | `lib/db` | `query()`, `queryOne()`, `transaction()` |
| Get a tenant record | `lib/queries` | `getTenant(tenantId)` |
| Get a lead record | `lib/queries` | `getLead(leadId, tenantId)`, `getLeadWithAgent()` |
| Get a user record | `lib/queries` | `getUser(userId, tenantId)` |
| Get active agents | `lib/queries` | `getActiveAgents(tenantId)` |
| Get default agent | `lib/queries` | `getDefaultAgent(tenantId)` |
| Get onboarding by token | `lib/queries` | `getOnboardingByToken(token)` |
| Find conversation by phone | `lib/queries` | `getConversationByPhone(phone)` |
| Get message history | `lib/queries` | `getConversationHistory(conversationId)` |
| Create a notification | `lib/notifications` | `createNotification()` |
| Get unread notifications | `lib/notifications` | `getUnreadNotifications()` |
| Send an SMS | `lib/twilio` | `sendSMS(to, body, from?)` |
| Upload to S3 | `lib/s3` | `uploadFile()`, `getPresignedUrl()` |
| Call OpenAI | `lib/openai` | `chatCompletion(system, user, options?)` |
| Get Stripe client | `lib/stripe` | `getStripeClient()` |
| Throw a typed error | `lib/errors` | `NotFoundError`, `ValidationError`, `ForbiddenError`, `UnauthorizedError` |
| Return an API response | `lib/response` | `success(data)`, `error(err)` |
| Wrap a handler in auth | `middleware/auth` | `withAuth(handler)` |
| Check plan access | `middleware/planGating` | `requireFeature(tenantId, feature)` |

### NEVER Do These Things

- **NEVER** instantiate a new `pg.Pool` or database connection — use `lib/db`
- **NEVER** write `const twilio = require('twilio')` — use `lib/twilio`
- **NEVER** write `const { S3Client } = require(...)` — use `lib/s3`
- **NEVER** write `const OpenAI = require('openai')` — use `lib/openai`
- **NEVER** write `const Stripe = require('stripe')` — use `lib/stripe`
- **NEVER** write a raw `SELECT * FROM tenants WHERE id = $1` inline — use `getTenant()` from `lib/queries`
- **NEVER** write a raw `SELECT * FROM leads WHERE id = $1 AND tenant_id = $2` inline — use `getLead()` from `lib/queries`
- **NEVER** write a raw `INSERT INTO notifications` inline — use `createNotification()` from `lib/notifications`
- **NEVER** write a custom response format — use `success()` and `error()` from `lib/response`
- **NEVER** write custom auth verification logic — use `withAuth()` from `middleware/auth`
- **NEVER** create new database tables or alter existing columns without explicit instruction to do so
- **NEVER** write a handler without the `withAuth` wrapper (except webhook handlers and public endpoints)

### If a Helper Doesn't Exist Yet

If you need a shared function that isn't in `lib/`, **add it to the appropriate existing lib file** — do not create it inline in the handler. For example:
- New query pattern needed? → Add to `lib/queries.js`
- New notification type? → Use existing `createNotification()` with a new `type` string
- New S3 operation? → Add to `lib/s3.js`

## Handler Pattern

Every authenticated handler follows this exact pattern:

```javascript
const { withAuth } = require('../middleware/auth');
const { success, error } = require('../lib/response');
const db = require('../lib/db');
const { getTenant, getLead } = require('../lib/queries');
const { createNotification } = require('../lib/notifications');

module.exports.handlerName = withAuth(async (event, { tenantId, userId, role }) => {
  try {
    // ... feature logic using shared utilities
    return success(result);
  } catch (err) {
    return error(err);
  }
});
```

Webhook handlers (no auth, signature-verified) follow this pattern:

```javascript
const { success, error } = require('../lib/response');
const db = require('../lib/db');

module.exports.handler = async (event) => {
  try {
    // Verify signature first
    // ... process webhook
    return success({ received: true });
  } catch (err) {
    return error(err);
  }
};
```

## Database Rules

- All tables already exist in `migrations/001_initial_schema.sql` — do not create them again
- Every query on tenant-scoped data MUST include `WHERE tenant_id = $X`
- Use parameterised queries (`$1`, `$2`) — never string interpolation
- Use `db.queryOne()` when expecting a single row, `db.query()` for lists
- Use `db.transaction()` when multiple writes must succeed or fail together

## Frontend Rules

- API calls go through `web/src/api/client.ts` which attaches the Clerk session token
- All pages under `/dashboard/*` are wrapped in Clerk's auth provider
- Public pages (`/book/:token`, `/onboard/:token`, `/aml-complete`) have no auth — token-based access only
- Use Tailwind utility classes — no custom CSS files
- Reusable UI components go in `web/src/components/ui/`

## Testing After Each Step

After completing each major step within a feature:
1. Check the code compiles/runs without errors
2. Verify imports resolve correctly (no missing modules)
3. Confirm no duplicate utility code was introduced
4. Check that all database queries include tenant_id filtering
