# PropFlow — Implementation Plan: ID & AML Checks

## PropFlow Context

PropFlow is a SaaS bundle for UK estate and letting agents, consolidating AI lead handling, digital onboarding, AML checks, and document signing into a single £149/month subscription. This document is one of 10 feature implementation plans that together form the complete MVP.

**Tech stack:** AWS Lambda + API Gateway, PostgreSQL (RDS), React + Tailwind, Clerk (auth), Stripe (billing), Twilio (comms), ComplyCube (AML), S3 (storage). All AWS eu-west-2.

**Build order:** This is **Feature 8 of 10**. Depends on Auth (1), Digital Onboarding (7), Stripe Billing (10 — for usage metering). Feeds into Document Signing (9).

---

## Feature Overview

ComplyCube-powered biometric ID verification, PEP/sanctions screening, and HMRC-compliant audit trail generation. Triggered within (or after) the onboarding flow. Checks are DIATF-certified, giving agents Statutory Excuse protection. Usage metered against monthly allowance; overages billed via Stripe.

## How It Works (Flow)

1. Onboarding completion (or agent manual trigger) → AML check initiated
2. Client receives SMS/email with ComplyCube hosted verification link
3. Client completes: selfie liveness check → document upload (passport/driving licence) → PEP/sanctions screening runs automatically
4. ComplyCube webhook fires on completion → PropFlow stores result
5. Audit trail PDF generated and stored in S3 (5-year retention)
6. Agent sees pass/fail/refer in dashboard
7. Usage counter incremented; if over allowance, overage recorded for billing

## Build Sequence

### Step 1: AML Checks Table
```sql
CREATE TABLE aml_checks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  lead_id UUID REFERENCES leads(id),
  onboarding_id UUID REFERENCES onboardings(id),
  
  -- Client info
  client_name VARCHAR(255) NOT NULL,
  client_email VARCHAR(255),
  client_phone VARCHAR(50),
  client_dob DATE,
  
  -- ComplyCube refs
  complycube_client_id VARCHAR(255),
  complycube_check_id VARCHAR(255),
  complycube_session_token VARCHAR(500),
  
  -- Results
  status VARCHAR(30) DEFAULT 'pending', -- pending/in_progress/passed/failed/referred/expired
  id_result VARCHAR(30), -- passed/failed/referred
  pep_result VARCHAR(30),
  sanctions_result VARCHAR(30),
  result_summary JSONB DEFAULT '{}',
  
  -- Audit
  audit_pdf_s3_key VARCHAR(500),
  completed_at TIMESTAMPTZ,
  expires_at TIMESTAMPTZ, -- results valid for 2 years per HMRC guidance
  
  -- Billing
  is_billable BOOLEAN DEFAULT TRUE, -- false for test/retry
  billed BOOLEAN DEFAULT FALSE,
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_aml_checks_tenant ON aml_checks(tenant_id);
CREATE INDEX idx_aml_checks_lead ON aml_checks(lead_id);
CREATE INDEX idx_aml_checks_status ON aml_checks(tenant_id, status);
```

### Step 2: ComplyCube Client Creation
```javascript
// POST /api/aml-checks
async function initiateAMLCheck(tenantId, leadId, onboardingId) {
  const tenant = await getTenant(tenantId);
  
  // Check usage allowance
  const monthlyUsed = await db.query(
    `SELECT COUNT(*) FROM aml_checks 
     WHERE tenant_id = $1 AND is_billable = TRUE
     AND created_at >= date_trunc('month', NOW())`,
    [tenantId]
  );
  
  const isOverage = parseInt(monthlyUsed.count) >= tenant.aml_checks_limit;
  
  // Get client data from onboarding
  const clientData = await getOnboardingClientData(onboardingId);
  
  // Create ComplyCube client
  const ccClient = await complycube.clients.create({
    type: 'person',
    email: clientData.email,
    personDetails: {
      firstName: clientData.firstName,
      lastName: clientData.lastName,
      dob: clientData.dob,
      nationality: clientData.nationality
    },
    address: {
      line1: clientData.addressLine1,
      city: clientData.city,
      postcode: clientData.postcode,
      country: 'GB'
    }
  });
  
  // Create verification session (hosted flow)
  const session = await complycube.sessions.create({
    clientId: ccClient.id,
    checkTypes: ['identity_check', 'document_check', 'screening_check'],
    successUrl: `https://app.dis-rupt.com/aml-complete/${tenantId}`,
    cancelUrl: `https://app.dis-rupt.com/aml-cancelled/${tenantId}`
  });
  
  // Store check record
  const check = await db.query(
    `INSERT INTO aml_checks (tenant_id, lead_id, onboarding_id, client_name, client_email,
     client_phone, client_dob, complycube_client_id, complycube_session_token, is_billable)
     VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9,$10) RETURNING *`,
    [tenantId, leadId, onboardingId, `${clientData.firstName} ${clientData.lastName}`,
     clientData.email, clientData.phone, clientData.dob, ccClient.id, session.token, true]
  );
  
  // Send verification link to client
  await twilioClient.messages.create({
    body: `${tenant.name} needs to verify your identity. Please complete this secure check: ${session.url}`,
    from: TENANT_PHONE,
    to: clientData.phone
  });
  
  // If overage, record for billing
  if (isOverage) {
    await recordAMLOverage(tenantId, check.id);
  }
  
  // Increment tenant counter
  await db.query(
    'UPDATE tenants SET aml_checks_used = aml_checks_used + 1 WHERE id = $1',
    [tenantId]
  );
  
  return check;
}
```

### Step 3: ComplyCube Webhook Handler
```javascript
// POST /api/webhooks/complycube
async function handleComplyCubeWebhook(event) {
  const { type, payload } = parseComplyCubeWebhook(event);
  
  // Verify webhook signature
  if (!verifyComplyCubeSignature(event)) throw new Error('Invalid signature');
  
  if (type === 'check.completed') {
    const check = await db.query(
      'SELECT * FROM aml_checks WHERE complycube_client_id = $1',
      [payload.clientId]
    );
    
    // Map ComplyCube result to our status
    const overallResult = mapResult(payload);
    
    await db.query(
      `UPDATE aml_checks SET 
        status = $1, id_result = $2, pep_result = $3, sanctions_result = $4,
        result_summary = $5, completed_at = NOW(),
        expires_at = NOW() + INTERVAL '2 years'
       WHERE id = $6`,
      [overallResult, payload.idResult, payload.pepResult, 
       payload.sanctionsResult, JSON.stringify(payload), check.id]
    );
    
    // Generate and store audit PDF
    const pdfKey = await generateAuditPDF(check, payload);
    await db.query('UPDATE aml_checks SET audit_pdf_s3_key = $1 WHERE id = $2', [pdfKey, check.id]);
    
    // Notify agent
    await createNotification(check.tenant_id, null, {
      type: overallResult === 'passed' ? 'aml_passed' : 'aml_attention',
      leadId: check.lead_id,
      title: `AML Check: ${overallResult.toUpperCase()}`,
      body: `${check.client_name} — ${overallResult}`
    });
  }
}
```

### Step 4: Audit Trail PDF Generation
```javascript
async function generateAuditPDF(check, complycubeResult) {
  // Use a PDF library (e.g. PDFKit) to generate HMRC-compliant audit trail
  // Must include:
  // - Date and time of check
  // - Client full name, DOB, address
  // - Document type and number verified
  // - Liveness check result
  // - PEP/Sanctions screening result
  // - DIATF certification reference
  // - PropFlow reference number
  // - Agency name and reference
  
  const pdf = generatePDF(check, complycubeResult);
  const s3Key = `aml-audits/${check.tenant_id}/${check.id}.pdf`;
  
  await s3.putObject({
    Bucket: 'propflow-documents',
    Key: s3Key,
    Body: pdf,
    ContentType: 'application/pdf',
    // 5-year retention per HMRC requirement
    ObjectLockRetainUntilDate: new Date(Date.now() + 5 * 365 * 24 * 60 * 60 * 1000)
  });
  
  return s3Key;
}
```

### Step 5: Dashboard — AML Section
- New tab on Lead Detail page: "AML Checks"
- Shows: check status (with colour), client name, date, result breakdown
- Actions: "Initiate Check" (if none exists), "Download Audit PDF", "View Details"
- List view: `/aml-checks` page showing all checks across the tenant with filters

### Step 6: Usage Dashboard
- In Settings → Billing section
- Shows: "X of Y AML checks used this month" with progress bar
- Overage count and estimated additional charge
- History table of past months' usage

### Step 7: Monthly Usage Reset
- EventBridge cron: first of each month at midnight
- Reset `aml_checks_used` on all tenants to 0
- Generate Stripe usage record for overages (ties into Stripe Billing feature)

## ComplyCube Setup Requirements
- Sign up at complycube.com (developer sandbox)
- Apply for startup programme (up to $50k credits)
- Generate API key
- Configure webhook URL: `https://api.dis-rupt.com/webhooks/complycube`
- Set up webhook event subscriptions: `check.completed`

## Environment Variables Required
```
COMPLYCUBE_API_KEY=
COMPLYCUBE_WEBHOOK_SECRET=
S3_DOCUMENTS_BUCKET=propflow-documents
```

## Testing Checklist
- [ ] AML check initiated from onboarding completion
- [ ] AML check initiated manually from dashboard
- [ ] ComplyCube client created with correct data
- [ ] Verification link sent to client via SMS
- [ ] ComplyCube webhook processed correctly on completion
- [ ] Pass/fail/refer status stored correctly
- [ ] Audit PDF generated with all required HMRC fields
- [ ] PDF stored in S3 with 5-year retention
- [ ] Agent receives notification on check completion
- [ ] Usage counter increments correctly
- [ ] Overage detection works when over monthly allowance
- [ ] Monthly reset fires correctly
- [ ] Dashboard shows check status and allows PDF download
- [ ] Failed/referred checks display appropriate guidance to agent

## Dependencies
- **Requires:** Auth (1), Digital Onboarding (7) — client data. Stripe Billing (10) — overage metering.
- **Required by:** Document Signing (9) — AML status shown alongside signing status.

## What This Feature Provides to Others
- `aml_checks` table with verification status per client
- Audit PDF available for download/attachment in document flows
- Usage data for Stripe Billing metered billing
- AML pass status as a prerequisite check for transaction progression
