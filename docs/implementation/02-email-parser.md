# PropFlow — Implementation Plan: Email Parser

## PropFlow Context

PropFlow is a SaaS bundle for UK estate and letting agents, consolidating AI lead handling, digital onboarding, AML checks, and document signing into a single £149/month subscription. This document is one of 10 feature implementation plans that together form the complete MVP.

**Tech stack:** AWS Lambda + API Gateway, PostgreSQL (RDS), React + Tailwind, Clerk (auth), Stripe (billing), Twilio (comms), S3 (storage). All AWS eu-west-2.

**Build order:** This is **Feature 2 of 10**. Depends on Auth & Multi-tenancy (Feature 1). AI Lead AutoPilot (Feature 3) depends on this.

---

## Feature Overview

AWS SES receives forwarded Rightmove/Zoopla/OnTheMarket lead emails from agents. A Lambda function parses the structured lead data and creates a lead record in the database. No CRM integration required.

## How It Works (Flow)

1. Agent sets up a forwarding rule in their inbox (Outlook/Gmail) to forward all Rightmove/Zoopla emails to `leads@dis-rupt.com` (or tenant-specific: `{tenant-slug}@leads.dis-rupt.com`)
2. AWS SES receives the email
3. SES stores raw email in S3 (`s3://propflow-emails/{tenant_id}/{timestamp}.eml`)
4. SES triggers Lambda function
5. Lambda parses email → extracts lead data → writes to DB → triggers AI Lead AutoPilot

## Build Sequence

### Step 1: AWS SES Setup
- Verify domain `dis-rupt.com` in SES (eu-west-2)
- Configure MX record: `10 inbound-smtp.eu-west-2.amazonaws.com`
- Create SES Receipt Rule Set:
  - Rule 1: Match `*@leads.dis-rupt.com`
  - Actions: S3 (store raw email) → Lambda (trigger parser)
- Set up S3 bucket `propflow-emails-{env}` with lifecycle policy (90-day retention for raw emails)

### Step 2: Tenant Email Mapping
```sql
CREATE TABLE tenant_email_mappings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  forwarding_address VARCHAR(255) UNIQUE NOT NULL, -- e.g. abc123@leads.dis-rupt.com
  source_email VARCHAR(255), -- agent's email that forwards
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```
- When tenant signs up, generate a unique forwarding address
- Display in dashboard with setup instructions for Outlook/Gmail forwarding rules

### Step 3: Leads Table
```sql
CREATE TABLE leads (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  source VARCHAR(50) NOT NULL, -- rightmove/zoopla/onthemarket/manual
  source_email_s3_key VARCHAR(500),
  
  -- Enquirer details
  enquirer_name VARCHAR(255),
  enquirer_email VARCHAR(255),
  enquirer_phone VARCHAR(50),
  enquirer_message TEXT,
  
  -- Property details
  property_address TEXT,
  property_postcode VARCHAR(20),
  property_price INTEGER,
  property_url VARCHAR(500),
  property_portal_ref VARCHAR(100),
  
  -- Qualification
  score VARCHAR(20) DEFAULT 'unscored', -- hot/warm/cold/unscored
  qualification_status VARCHAR(50) DEFAULT 'pending', -- pending/in_progress/qualified/unresponsive
  qualification_data JSONB DEFAULT '{}',
  
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
```

### Step 4: Email Parser Lambda
```javascript
// handlers/emailParser.js
// Triggered by SES → S3 event

async function handler(event) {
  // 1. Get raw email from S3
  const s3Key = event.Records[0].s3.object.key;
  const rawEmail = await s3.getObject({ Bucket: BUCKET, Key: s3Key });
  
  // 2. Parse MIME email (use 'mailparser' npm package)
  const parsed = await simpleParser(rawEmail.Body);
  
  // 3. Identify source portal from sender/subject
  const source = identifySource(parsed.from, parsed.subject);
  // Rightmove: from noreply@rightmove.co.uk, subject contains property address
  // Zoopla: from noreply@zoopla.co.uk
  // OnTheMarket: from enquiries@onthemarket.com
  
  // 4. Determine tenant from recipient address
  const recipient = parsed.to[0].address; // e.g. abc123@leads.dis-rupt.com
  const tenant = await db.query(
    'SELECT tenant_id FROM tenant_email_mappings WHERE forwarding_address = $1',
    [recipient]
  );
  
  // 5. Extract lead data using GPT-4o-mini
  const extractedData = await extractLeadData(parsed.text || parsed.html, source);
  
  // 6. Insert lead record
  const lead = await db.query(
    `INSERT INTO leads (tenant_id, source, source_email_s3_key, 
     enquirer_name, enquirer_email, enquirer_phone, enquirer_message,
     property_address, property_postcode, property_price, property_url)
     VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9,$10,$11) RETURNING *`,
    [tenant.tenant_id, source, s3Key, ...]
  );
  
  // 7. Trigger AI Lead AutoPilot (invoke next Lambda or publish to SQS)
  await triggerLeadQualification(lead);
}
```

### Step 5: GPT-4o-mini Extraction
```javascript
async function extractLeadData(emailBody, source) {
  const response = await openai.chat.completions.create({
    model: 'gpt-4o-mini',
    temperature: 0,
    response_format: { type: 'json_object' },
    messages: [{
      role: 'system',
      content: `Extract lead data from this ${source} enquiry email. 
Return JSON with: enquirer_name, enquirer_email, enquirer_phone, 
enquirer_message, property_address, property_postcode, property_price, 
property_url. Use null for missing fields.`
    }, {
      role: 'user',
      content: emailBody
    }]
  });
  return JSON.parse(response.choices[0].message.content);
}
```

### Step 6: Fallback/Regex Parser
- Rightmove and Zoopla emails have consistent HTML templates
- Build regex fallback for when GPT extraction fails or is too slow
- Rightmove pattern: name in `<strong>` after "Name:", phone after "Telephone:", etc.
- Use GPT as primary, regex as fallback — log mismatches for monitoring

### Step 7: Deduplication
- Check for duplicate leads: same `enquirer_email` + `property_url` + `tenant_id` within 24 hours
- If duplicate, update existing lead record rather than creating new one
- Log duplicate detection for analytics

## Environment Variables Required
```
AWS_SES_REGION=eu-west-2
S3_EMAIL_BUCKET=propflow-emails-{env}
OPENAI_API_KEY=
```

## Testing Checklist
- [ ] SES receives email and stores in S3
- [ ] Lambda triggers on S3 put event
- [ ] Rightmove email parsed correctly (name, phone, email, property)
- [ ] Zoopla email parsed correctly
- [ ] OnTheMarket email parsed correctly
- [ ] Unknown format logged but doesn't crash
- [ ] Correct tenant identified from forwarding address
- [ ] Duplicate lead within 24hrs updates rather than creates
- [ ] GPT extraction failure falls back to regex
- [ ] Lead record written to DB with correct tenant_id
- [ ] AI Lead AutoPilot triggered after lead creation

## Dependencies
- **Requires:** Auth & Multi-tenancy (tenant records, DB schema)
- **Required by:** AI Lead AutoPilot (needs parsed lead data to start qualification)

## What This Feature Provides to Others
- `leads` table populated with parsed portal enquiries
- S3 archive of raw emails for audit
- Event trigger for downstream lead qualification
- `tenant_email_mappings` for agent onboarding setup
