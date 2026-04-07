# PropFlow — Implementation Plan: AI Lead AutoPilot

## PropFlow Context

PropFlow is a SaaS bundle for UK estate and letting agents, consolidating AI lead handling, digital onboarding, AML checks, and document signing into a single £149/month subscription. This document is one of 10 feature implementation plans that together form the complete MVP.

**Tech stack:** AWS Lambda + API Gateway, PostgreSQL (RDS), React + Tailwind, Clerk (auth), Stripe (billing), Twilio (comms), S3 (storage). All AWS eu-west-2.

**Build order:** This is **Feature 3 of 10**. Depends on Auth & Multi-tenancy (1) and Email Parser (2). Feeds into Lead Scoring & Routing (4).

---

## Feature Overview

When a lead is created by the Email Parser, the AI Lead AutoPilot sends an initial SMS/WhatsApp to the enquirer within 30 seconds. It then conducts a 3-question AI qualification dialogue to determine buyer readiness. Conversation history is stored against the lead record.

## How It Works (Flow)

1. Email Parser creates lead → triggers this feature
2. Lambda sends initial outreach SMS via Twilio (within 30 seconds of email receipt)
3. Enquirer replies via SMS
4. Twilio webhook hits Lambda → GPT-4o-mini generates contextual follow-up
5. After 3 key questions answered → lead scored → Lead Scoring & Routing takes over
6. If no reply within 2 hours → automated follow-up. After 24 hours → mark as unresponsive.

## The 3 Qualification Questions

1. **"Have you sold your current property, or are you a first-time buyer?"** — establishes chain position
2. **"Do you have a mortgage Agreement in Principle (AIP)?"** — establishes financial readiness
3. **"When are you looking to move?"** — establishes urgency

These are conversational, not robotic. GPT adapts the phrasing based on context.

## Build Sequence

### Step 1: Conversations Table
```sql
CREATE TABLE conversations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  lead_id UUID NOT NULL REFERENCES leads(id),
  channel VARCHAR(20) NOT NULL, -- sms/whatsapp
  twilio_sid VARCHAR(100),
  phone_number VARCHAR(50) NOT NULL,
  status VARCHAR(50) DEFAULT 'active', -- active/completed/unresponsive
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID NOT NULL REFERENCES conversations(id),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  direction VARCHAR(10) NOT NULL, -- inbound/outbound
  channel VARCHAR(20) NOT NULL,
  body TEXT NOT NULL,
  twilio_message_sid VARCHAR(100),
  sent_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_conversations_lead ON conversations(lead_id);
CREATE INDEX idx_messages_conversation ON messages(conversation_id);
```

### Step 2: Initial Outreach Lambda
```javascript
// handlers/leadAutoReply.js
// Triggered by Email Parser (SQS or direct invoke)

async function handler(event) {
  const { leadId, tenantId } = event;
  
  const lead = await db.query('SELECT * FROM leads WHERE id = $1 AND tenant_id = $2', [leadId, tenantId]);
  const tenant = await db.query('SELECT * FROM tenants WHERE id = $1', [tenantId]);
  
  if (!lead.enquirer_phone) {
    // No phone → send email reply instead (future enhancement)
    await updateLeadStatus(leadId, 'no_phone');
    return;
  }
  
  // Generate personalised opening message
  const openingMessage = await generateOpeningMessage(lead, tenant);
  
  // Send via Twilio SMS
  const twilioMessage = await twilioClient.messages.create({
    body: openingMessage,
    from: tenant.settings.twilio_phone_number, // tenant's Twilio number
    to: lead.enquirer_phone
  });
  
  // Create conversation + message records
  const conversation = await createConversation(tenantId, leadId, 'sms', lead.enquirer_phone);
  await createMessage(conversation.id, tenantId, 'outbound', 'sms', openingMessage, twilioMessage.sid);
  
  // Update lead
  await db.query('UPDATE leads SET first_response_at = NOW(), qualification_status = $1 WHERE id = $2',
    ['in_progress', leadId]);
  
  // Schedule follow-up check (2 hours)
  await scheduleFollowUp(leadId, tenantId, 2 * 60 * 60);
}
```

### Step 3: GPT Opening Message Generation
```javascript
async function generateOpeningMessage(lead, tenant) {
  const response = await openai.chat.completions.create({
    model: 'gpt-4o-mini',
    temperature: 0.7,
    max_tokens: 200,
    messages: [{
      role: 'system',
      content: `You are a friendly, professional assistant for ${tenant.name}, a UK estate agency.
A potential buyer has just enquired about a property on ${lead.source}.
Write a SHORT opening SMS (max 160 chars if possible, max 300 chars).
- Thank them for their enquiry about the property at ${lead.property_address}
- Ask the first qualification question naturally: whether they've sold their property or are a first-time buyer
- Sign off with the agency name
- British English. Professional but warm. No emojis.
- Do NOT say you are an AI or chatbot.`
    }]
  });
  return response.choices[0].message.content;
}
```

### Step 4: Twilio Webhook Handler (Inbound Replies)
```javascript
// handlers/twilioWebhook.js
// POST /api/webhooks/twilio/sms

async function handler(event) {
  const { From, Body, MessageSid } = parseTwilioWebhook(event);
  
  // Find active conversation by phone number
  const conversation = await db.query(
    `SELECT c.*, l.* FROM conversations c 
     JOIN leads l ON c.lead_id = l.id 
     WHERE c.phone_number = $1 AND c.status = 'active' 
     ORDER BY c.created_at DESC LIMIT 1`,
    [From]
  );
  
  if (!conversation) return; // Unknown number — ignore
  
  // Store inbound message
  await createMessage(conversation.id, conversation.tenant_id, 'inbound', 'sms', Body, MessageSid);
  
  // Get conversation history
  const history = await getConversationHistory(conversation.id);
  
  // Generate AI response
  const aiResponse = await generateQualificationResponse(conversation, history, Body);
  
  // Send reply
  const reply = await twilioClient.messages.create({
    body: aiResponse.message,
    from: conversation.twilio_from_number,
    to: From
  });
  
  await createMessage(conversation.id, conversation.tenant_id, 'outbound', 'sms', aiResponse.message, reply.sid);
  
  // Check if qualification is complete
  if (aiResponse.qualificationComplete) {
    await completeQualification(conversation.lead_id, conversation.tenant_id, aiResponse.qualificationData);
  }
}
```

### Step 5: AI Qualification Logic
```javascript
async function generateQualificationResponse(conversation, history, latestMessage) {
  const messagesForGPT = history.map(m => ({
    role: m.direction === 'outbound' ? 'assistant' : 'user',
    content: m.body
  }));
  
  const response = await openai.chat.completions.create({
    model: 'gpt-4o-mini',
    temperature: 0.7,
    response_format: { type: 'json_object' },
    messages: [{
      role: 'system',
      content: `You are a qualification assistant for a UK estate agency.
You need to find out 3 things from this buyer:
1. Chain position: Have they sold? First-time buyer? Renting?
2. Financial readiness: Do they have an AIP (Agreement in Principle)?
3. Timeline: When do they need to move?

You have gathered some answers already from the conversation history.
Analyse the latest reply and determine:
- What new info was provided
- What questions remain
- Whether qualification is complete

Respond with JSON:
{
  "message": "Your next SMS reply (max 300 chars, professional, British English, no emojis, don't say you're AI)",
  "qualificationComplete": true/false,
  "qualificationData": {
    "chain_position": "first_time_buyer|sold_stc|under_offer|not_yet_listed|renting|cash_buyer|null",
    "has_aip": true/false/null,
    "timeline": "asap|1_3_months|3_6_months|6_plus_months|just_browsing|null"
  },
  "detectedCrossSell": true/false  // buyer mentions needing to sell their property
}

If the buyer seems disinterested or asks to stop, be polite and offer to have an agent call them instead.
If the buyer asks a property-specific question you can't answer, offer to arrange a viewing or have the agent call.`
    }, ...messagesForGPT]
  });
  
  return JSON.parse(response.choices[0].message.content);
}
```

### Step 6: Follow-Up Scheduling
- Use AWS EventBridge Scheduler or SQS delay queues
- 2-hour follow-up: "Hi {name}, just checking if you got my message about {property_address}. Happy to help with any questions!"
- 24-hour mark: Set status to `unresponsive`, stop automated messages
- Max 2 follow-ups total (avoid spam complaints)

### Step 7: Qualification Completion
```javascript
async function completeQualification(leadId, tenantId, qualificationData) {
  await db.query(
    `UPDATE leads SET 
       qualification_status = 'qualified',
       qualification_data = $1,
       qualified_at = NOW()
     WHERE id = $2 AND tenant_id = $3`,
    [JSON.stringify(qualificationData), leadId, tenantId]
  );
  
  // Trigger Lead Scoring & Routing (Feature 4)
  await triggerLeadScoring(leadId, tenantId, qualificationData);
}
```

## Twilio Setup Requirements
- Buy a UK phone number per tenant (or use a shared number with tenant routing)
- Configure SMS webhook URL: `https://api.dis-rupt.com/webhooks/twilio/sms`
- WhatsApp: use Twilio standard WhatsApp initially (not WhatsApp Business API — that's Phase 2)

## Environment Variables Required
```
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_PHONE_NUMBER=  # default, can be overridden per tenant
OPENAI_API_KEY=
```

## Testing Checklist
- [ ] Lead creation triggers outreach within 30 seconds
- [ ] Opening message is personalised with property address and agency name
- [ ] Inbound SMS matched to correct conversation
- [ ] AI generates appropriate follow-up questions
- [ ] Qualification data extracted correctly after 3 questions answered
- [ ] Lead marked as qualified with correct data
- [ ] 2-hour follow-up fires if no reply
- [ ] 24-hour timeout marks lead as unresponsive
- [ ] No more than 2 automated follow-ups sent
- [ ] Cross-sell detection flags correctly when buyer mentions selling
- [ ] Lead without phone number handled gracefully (no crash)

## Dependencies
- **Requires:** Auth & Multi-tenancy (1), Email Parser (2)
- **Required by:** Lead Scoring & Routing (4) — receives qualification data

## What This Feature Provides to Others
- `conversations` and `messages` tables for the Agent Dashboard
- `qualification_data` on the lead record for scoring
- `first_response_at` timestamp for speed-to-lead metrics
- Cross-sell flag for future valuation lead workflows
