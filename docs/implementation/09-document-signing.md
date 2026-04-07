# PropFlow — Implementation Plan: Document Signing

## PropFlow Context

PropFlow is a SaaS bundle for UK estate and letting agents, consolidating AI lead handling, digital onboarding, AML checks, and document signing into a single £149/month subscription. This document is one of 10 feature implementation plans that together form the complete MVP.

**Tech stack:** AWS Lambda + API Gateway, PostgreSQL (RDS), React + Tailwind, Clerk (auth), Stripe (billing), Twilio (comms), Dropbox Sign/HelloSign (signing), S3 (storage). All AWS eu-west-2.

**Build order:** This is **Feature 9 of 10**. Depends on Auth (1), Digital Onboarding (7) for client data pre-fill, Agent Dashboard (6) for UI.

---

## Feature Overview

E-signature via Dropbox Sign (HelloSign) API. Agents create document templates, send for signing to one or multiple parties (buyer, seller, agent), track signing status, and store completed documents. Signing links delivered via SMS — no account creation required for signers. Unlimited signings included in the bundle.

## How It Works (Flow)

1. Agent selects a template (or uploads a document) from the dashboard
2. System pre-fills fields with client data from onboarding
3. Agent specifies signers (email + phone for each party)
4. Dropbox Sign creates signature request → signing links generated
5. SMS sent to each signer with their unique link
6. Signers sign in browser (no account needed)
7. Dropbox Sign webhook fires on completion → signed PDF stored in S3
8. Agent sees status in dashboard

## Build Sequence

### Step 1: Documents & Templates Tables
```sql
-- Reusable templates created by agents
CREATE TABLE document_templates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  name VARCHAR(255) NOT NULL,
  description TEXT,
  category VARCHAR(50), -- agency_agreement/offer/tenancy/ta6/custom
  hellosign_template_id VARCHAR(255), -- Dropbox Sign template reference
  fields JSONB DEFAULT '[]', -- template merge fields definition
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Individual signing requests
CREATE TABLE signing_requests (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  lead_id UUID REFERENCES leads(id),
  template_id UUID REFERENCES document_templates(id),
  
  -- Dropbox Sign refs
  hellosign_signature_request_id VARCHAR(255),
  
  -- Document
  title VARCHAR(255) NOT NULL,
  document_url VARCHAR(500), -- uploaded doc URL if not from template
  
  -- Status
  status VARCHAR(30) DEFAULT 'sent', -- draft/sent/partially_signed/completed/declined/expired
  
  -- Signed document
  signed_pdf_s3_key VARCHAR(500),
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  completed_at TIMESTAMPTZ,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Individual signers on a request
CREATE TABLE signers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  signing_request_id UUID NOT NULL REFERENCES signing_requests(id),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  
  name VARCHAR(255) NOT NULL,
  email VARCHAR(255) NOT NULL,
  phone VARCHAR(50),
  role VARCHAR(50) NOT NULL, -- buyer/seller/agent/landlord/tenant/witness
  order_index INTEGER DEFAULT 0, -- for sequential signing
  
  -- Status
  status VARCHAR(30) DEFAULT 'pending', -- pending/signed/declined
  hellosign_signature_id VARCHAR(255),
  signed_at TIMESTAMPTZ,
  
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_signing_requests_tenant ON signing_requests(tenant_id);
CREATE INDEX idx_signing_requests_lead ON signing_requests(lead_id);
CREATE INDEX idx_signers_request ON signers(signing_request_id);
```

### Step 2: Template Management
```javascript
// POST /api/templates — create template
// Agent uploads a PDF/DOCX → Dropbox Sign creates a template with field placeholders

async function createTemplate(tenantId, file, templateConfig) {
  // Upload to Dropbox Sign as template
  const hsTemplate = await hellosign.template.createEmbeddedDraft({
    clientId: HELLOSIGN_CLIENT_ID,
    files: [file],
    signerRoles: templateConfig.signerRoles, // e.g. ['Buyer', 'Seller', 'Agent']
    mergeFields: templateConfig.mergeFields, // e.g. [{name: 'client_name', type: 'text'}]
    title: templateConfig.name,
    subject: templateConfig.name
  });
  
  // Store template reference
  await db.query(
    `INSERT INTO document_templates (tenant_id, name, description, category, hellosign_template_id, fields)
     VALUES ($1,$2,$3,$4,$5,$6)`,
    [tenantId, templateConfig.name, templateConfig.description, templateConfig.category,
     hsTemplate.template_id, JSON.stringify(templateConfig.mergeFields)]
  );
}
```

### Step 3: Send for Signing
```javascript
// POST /api/signing-requests
async function createSigningRequest(tenantId, templateId, leadId, signers, mergeData) {
  const template = await getTemplate(templateId, tenantId);
  
  // Create signature request via Dropbox Sign
  const signersList = signers.map((s, i) => ({
    email_address: s.email,
    name: s.name,
    role: s.role,
    order: i
  }));
  
  const request = await hellosign.signatureRequest.sendWithTemplate({
    template_ids: [template.hellosign_template_id],
    title: mergeData.title || template.name,
    subject: `Please sign: ${mergeData.title || template.name}`,
    signers: signersList,
    custom_fields: mergeData.fields || [], // pre-filled from onboarding data
    test_mode: process.env.NODE_ENV !== 'production'
  });
  
  // Store signing request
  const signingReq = await db.query(
    `INSERT INTO signing_requests (tenant_id, lead_id, template_id, hellosign_signature_request_id, title)
     VALUES ($1,$2,$3,$4,$5) RETURNING *`,
    [tenantId, leadId, templateId, request.signature_request_id, mergeData.title || template.name]
  );
  
  // Store signers and send SMS links
  for (const sig of request.signatures) {
    const signerData = signers.find(s => s.email === sig.signer_email_address);
    
    await db.query(
      `INSERT INTO signers (signing_request_id, tenant_id, name, email, phone, role, hellosign_signature_id)
       VALUES ($1,$2,$3,$4,$5,$6,$7)`,
      [signingReq.id, tenantId, sig.signer_name, sig.signer_email_address,
       signerData?.phone, signerData?.role, sig.signature_id]
    );
    
    // Send signing link via SMS (if phone provided)
    if (signerData?.phone) {
      const signingUrl = await hellosign.signatureRequest.getSigningUrl(sig.signature_id);
      await twilioClient.messages.create({
        body: `Please sign "${mergeData.title}": ${signingUrl.sign_url}`,
        from: TENANT_PHONE,
        to: signerData.phone
      });
    }
  }
  
  return signingReq;
}
```

### Step 4: Dropbox Sign Webhook Handler
```javascript
// POST /api/webhooks/hellosign
async function handleHelloSignWebhook(event) {
  const { event_type, signature_request } = parseHelloSignWebhook(event);
  
  // Verify webhook — HelloSign requires returning "Hello API Event Received"
  
  switch (event_type) {
    case 'signature_request_signed':
      // Individual signer completed
      const signer = signature_request.signatures.find(s => s.status_code === 'signed');
      await db.query(
        `UPDATE signers SET status = 'signed', signed_at = NOW() 
         WHERE hellosign_signature_id = $1`,
        [signer.signature_id]
      );
      
      // Check if all signers done
      const allSigned = signature_request.signatures.every(s => s.status_code === 'signed');
      if (allSigned) {
        await completeSigningRequest(signature_request);
      } else {
        await db.query(
          `UPDATE signing_requests SET status = 'partially_signed' 
           WHERE hellosign_signature_request_id = $1`,
          [signature_request.signature_request_id]
        );
      }
      break;
      
    case 'signature_request_all_signed':
      await completeSigningRequest(signature_request);
      break;
      
    case 'signature_request_declined':
      await db.query(
        `UPDATE signing_requests SET status = 'declined' 
         WHERE hellosign_signature_request_id = $1`,
        [signature_request.signature_request_id]
      );
      // Notify agent
      break;
  }
  
  return 'Hello API Event Received'; // Required response
}

async function completeSigningRequest(signatureRequest) {
  // Download signed PDF from Dropbox Sign
  const pdf = await hellosign.signatureRequest.download(
    signatureRequest.signature_request_id,
    { file_type: 'pdf' }
  );
  
  // Store in S3
  const signingReq = await db.query(
    'SELECT * FROM signing_requests WHERE hellosign_signature_request_id = $1',
    [signatureRequest.signature_request_id]
  );
  
  const s3Key = `signed-docs/${signingReq.tenant_id}/${signingReq.id}.pdf`;
  await s3.putObject({
    Bucket: 'propflow-documents',
    Key: s3Key,
    Body: pdf,
    ContentType: 'application/pdf'
  });
  
  await db.query(
    `UPDATE signing_requests SET status = 'completed', signed_pdf_s3_key = $1, completed_at = NOW()
     WHERE id = $2`,
    [s3Key, signingReq.id]
  );
  
  // Notify agent
  await createNotification(signingReq.tenant_id, null, {
    type: 'document_signed',
    leadId: signingReq.lead_id,
    title: 'Document fully signed',
    body: signingReq.title
  });
}
```

### Step 5: Dashboard — Documents Section
- New tab on Lead Detail page: "Documents"
- Shows: all signing requests for this lead, status per signer
- Actions: "Send for Signing" (select template), "Download Signed PDF", "Send Reminder"
- Template management page: `/settings/templates` — list, create, edit templates

### Step 6: Pre-Fill from Onboarding Data
```javascript
// When creating a signing request for a lead with completed onboarding:
async function getMergeFields(leadId, tenantId) {
  const onboarding = await db.query(
    `SELECT od.section, od.data FROM onboarding_data od
     JOIN onboardings o ON od.onboarding_id = o.id
     WHERE o.lead_id = $1 AND o.tenant_id = $2 AND o.status = 'completed'`,
    [leadId, tenantId]
  );
  
  // Flatten into merge field format
  return {
    client_name: onboarding.personal_details.fullName,
    client_address: onboarding.personal_details.address,
    property_address: onboarding.property_info?.address,
    solicitor_name: onboarding.solicitor_details?.firmName,
    // ... etc
  };
}
```

## Dropbox Sign Setup Requirements
- Create Dropbox Sign API account (developer plan for testing)
- Generate API key
- Configure webhook callback URL: `https://api.dis-rupt.com/webhooks/hellosign`
- Create initial templates for common documents (agency agreement, offer form)

## Environment Variables Required
```
HELLOSIGN_API_KEY=
HELLOSIGN_CLIENT_ID=
HELLOSIGN_WEBHOOK_SECRET=
```

## Testing Checklist
- [ ] Template created in Dropbox Sign from uploaded document
- [ ] Signing request sent to multiple parties
- [ ] SMS with signing link delivered to each signer
- [ ] Individual signer status updated on webhook
- [ ] All-signed status triggers PDF download and S3 storage
- [ ] Declined request updates status and notifies agent
- [ ] Signed PDF downloadable from dashboard
- [ ] Pre-fill works with onboarding data
- [ ] Template management (list, create) works in dashboard
- [ ] Signing request without template (custom upload) works
- [ ] Webhook response returns required "Hello API Event Received" string

## Dependencies
- **Requires:** Auth (1), Agent Dashboard (6), Digital Onboarding (7) — for data pre-fill
- **Required by:** None at MVP — standalone feature

## What This Feature Provides to Others
- `signing_requests` and `signers` tables for document status tracking
- Signed PDF storage in S3 for audit/compliance
- Template system reusable for future document types
