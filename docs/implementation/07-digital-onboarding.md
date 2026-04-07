# PropFlow — Implementation Plan: Digital Onboarding Flow

## PropFlow Context

PropFlow is a SaaS bundle for UK estate and letting agents, consolidating AI lead handling, digital onboarding, AML checks, and document signing into a single £149/month subscription. This document is one of 10 feature implementation plans that together form the complete MVP.

**Tech stack:** AWS Lambda + API Gateway, PostgreSQL (RDS), React + Tailwind, Clerk (auth), Stripe (billing), Twilio (comms), S3 (storage). All AWS eu-west-2.

**Build order:** This is **Feature 7 of 10**. Depends on Auth (1), Agent Dashboard (6). Feeds into ID & AML Checks (8) and Document Signing (9).

---

## Feature Overview

Branded client-facing digital journey that collects buyer/seller details and Material Information. Agent triggers onboarding from the dashboard; client receives an SMS/email link to a self-serve form. Progress tracked in real-time on the agent's dashboard. Completed onboarding can trigger AML checks and document signing.

## How It Works (Flow)

1. Agent clicks "Send Onboarding" from lead detail page
2. System generates a unique onboarding link
3. Client receives SMS + email with link
4. Client completes multi-step form (no login required — token-authenticated)
5. Agent sees completion progress in real-time
6. On completion → triggers AML check invitation if required

## Build Sequence

### Step 1: Onboarding Records Table
```sql
CREATE TABLE onboardings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  lead_id UUID REFERENCES leads(id),
  
  -- Access
  token VARCHAR(255) UNIQUE NOT NULL, -- URL token for client access
  expires_at TIMESTAMPTZ NOT NULL, -- 30-day expiry
  
  -- Type
  client_type VARCHAR(20) NOT NULL, -- buyer/seller/landlord/tenant
  
  -- Status
  status VARCHAR(30) DEFAULT 'invited', -- invited/in_progress/completed/expired
  current_step INTEGER DEFAULT 0,
  total_steps INTEGER DEFAULT 5,
  completed_at TIMESTAMPTZ,
  last_activity_at TIMESTAMPTZ,
  
  -- Auto-chase
  chase_count INTEGER DEFAULT 0,
  next_chase_at TIMESTAMPTZ,
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE onboarding_data (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  onboarding_id UUID NOT NULL REFERENCES onboardings(id),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  section VARCHAR(50) NOT NULL, -- personal_details/property_info/material_info_a/material_info_b/material_info_c
  data JSONB NOT NULL DEFAULT '{}',
  completed BOOLEAN DEFAULT FALSE,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_onboardings_token ON onboardings(token);
CREATE INDEX idx_onboardings_tenant ON onboardings(tenant_id);
CREATE INDEX idx_onboarding_data_onboarding ON onboarding_data(onboarding_id);
```

### Step 2: Onboarding Invitation API
```javascript
// POST /api/onboardings
async function createOnboarding(tenantId, leadId, clientType) {
  const lead = await getLead(leadId, tenantId);
  const token = generateSecureToken(); // crypto.randomUUID() or similar
  
  const onboarding = await db.query(
    `INSERT INTO onboardings (tenant_id, lead_id, token, expires_at, client_type, total_steps)
     VALUES ($1, $2, $3, NOW() + INTERVAL '30 days', $4, $5) RETURNING *`,
    [tenantId, leadId, token, clientType, clientType === 'seller' ? 6 : 5]
  );
  
  // Pre-create data sections
  const sections = clientType === 'seller' 
    ? ['personal_details', 'property_info', 'material_info_a', 'material_info_b', 'material_info_c']
    : ['personal_details', 'financial_details', 'solicitor_details', 'requirements', 'declarations'];
  
  for (const section of sections) {
    await db.query(
      `INSERT INTO onboarding_data (onboarding_id, tenant_id, section) VALUES ($1, $2, $3)`,
      [onboarding.id, tenantId, section]
    );
  }
  
  // Send invitation
  const link = `https://app.dis-rupt.com/onboard/${token}`;
  await sendOnboardingInvite(lead, link, tenantId);
  
  return onboarding;
}
```

### Step 3: Client-Facing Onboarding Form (React)
- Route: `/onboard/:token` — **public page, no auth** (token-based access)
- Branded: fetch tenant details from token → display agency logo/name/colours
- Multi-step wizard with progress bar

**Buyer Steps:**
1. **Personal Details** — full name, DOB, current address, phone, email, nationality
2. **Financial Details** — purchase method (mortgage/cash), lender name, AIP status, deposit amount, solicitor name + contact
3. **Solicitor Details** — firm name, contact name, email, phone, reference
4. **Requirements** — desired move date, chain position, any conditions
5. **Declarations** — source of funds declaration, consent for AML checks, T&Cs acceptance

**Seller Steps:**
1. **Personal Details** — same as buyer
2. **Property Information** — address, tenure, council tax band, EPC rating
3. **Material Information Part A** — property-specific: utilities, parking, broadband, flooding, boundary disputes, planning permissions
4. **Material Information Part B** — property rights: right of way, shared access, restrictive covenants, conservation area
5. **Material Information Part C** — additional: alterations, guarantees/warranties, complaints, insurance claims
6. **Declarations** — accuracy declaration, consent for checks

### Step 4: Section Save API
```javascript
// PATCH /api/onboard/:token/sections/:section
// No auth — token validation only

async function saveSectionData(token, section, data) {
  const onboarding = await db.query(
    'SELECT * FROM onboardings WHERE token = $1 AND expires_at > NOW()',
    [token]
  );
  if (!onboarding) throw new Error('Invalid or expired link');
  
  await db.query(
    `UPDATE onboarding_data SET data = $1, completed = $2, updated_at = NOW()
     WHERE onboarding_id = $3 AND section = $4`,
    [JSON.stringify(data), true, onboarding.id, section]
  );
  
  // Update progress
  const completed = await db.query(
    'SELECT COUNT(*) FROM onboarding_data WHERE onboarding_id = $1 AND completed = TRUE',
    [onboarding.id]
  );
  
  await db.query(
    `UPDATE onboardings SET current_step = $1, status = 'in_progress', 
     last_activity_at = NOW() WHERE id = $2`,
    [completed.count, onboarding.id]
  );
  
  // Check if all complete
  if (parseInt(completed.count) >= onboarding.total_steps) {
    await db.query(
      `UPDATE onboardings SET status = 'completed', completed_at = NOW() WHERE id = $1`,
      [onboarding.id]
    );
    await triggerOnboardingComplete(onboarding);
  }
}
```

### Step 5: Auto-Save & Resume
- Form auto-saves each section on blur/tab switch
- Client can close browser and return later — token valid for 30 days
- Progress restored from `onboarding_data` records on page load

### Step 6: Agent Dashboard — Onboarding View
- New section in Lead Detail page: "Onboarding" tab
- Shows: status, progress bar, per-section completion
- View submitted data (read-only)
- Actions: "Resend Invite", "Send Reminder"
- List view: `/onboardings` page showing all active onboardings with status

### Step 7: Auto-Chase
- EventBridge schedule: check daily for incomplete onboardings with `last_activity_at` > 48 hours
- Send reminder SMS (max 3 reminders)
- After 3 reminders + 7 days inactivity → mark as stale, notify agent

### Step 8: Completion Trigger
```javascript
async function triggerOnboardingComplete(onboarding) {
  // Notify agent
  await createNotification(onboarding.tenant_id, null, {
    type: 'onboarding_complete',
    leadId: onboarding.lead_id,
    title: 'Onboarding completed',
    body: `${onboarding.client_name} has completed their onboarding`
  });
  
  // If AML check consent given → trigger AML check invitation (Feature 8)
  const declarations = await getOnboardingSection(onboarding.id, 'declarations');
  if (declarations.data.aml_consent) {
    await triggerAMLCheck(onboarding);
  }
}
```

## Testing Checklist
- [ ] Agent can trigger onboarding from lead detail
- [ ] Client receives SMS + email with link
- [ ] Token-based access works without login
- [ ] All form steps render correctly for buyer and seller types
- [ ] Section data saves correctly
- [ ] Auto-save on blur works
- [ ] Client can close and resume from where they left off
- [ ] Progress bar updates in real-time on agent dashboard
- [ ] Auto-chase fires after 48 hours inactivity
- [ ] Max 3 reminders sent
- [ ] Expired token (30 days) returns appropriate error page
- [ ] Completed onboarding triggers AML check invitation if consent given
- [ ] Agent can view submitted data in dashboard
- [ ] Tenant branding (logo/name) shown on client form

## Dependencies
- **Requires:** Auth (1), Agent Dashboard (6)
- **Required by:** ID & AML Checks (8) — AML triggered post-onboarding. Document Signing (9) — client data pre-fills signing templates.

## What This Feature Provides to Others
- `onboardings` and `onboarding_data` tables with structured client data
- Client data (name, address, solicitor) available for pre-filling document templates
- AML consent flag for triggering ID & AML Checks
- Token-based public page pattern reusable for other client-facing flows
