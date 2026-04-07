# PropFlow — Implementation Plan: Lead Scoring & Routing

## PropFlow Context

PropFlow is a SaaS bundle for UK estate and letting agents, consolidating AI lead handling, digital onboarding, AML checks, and document signing into a single £149/month subscription. This document is one of 10 feature implementation plans that together form the complete MVP.

**Tech stack:** AWS Lambda + API Gateway, PostgreSQL (RDS), React + Tailwind, Clerk (auth), Stripe (billing), Twilio (comms), S3 (storage). All AWS eu-west-2.

**Build order:** This is **Feature 4 of 10**. Depends on AI Lead AutoPilot (3) for qualification data. Feeds into Viewing Booking (5) for cold lead auto-booking.

---

## Feature Overview

Takes qualification data from AI Lead AutoPilot and scores leads as Hot/Warm/Cold. Hot leads trigger an instant push notification to the assigned agent. Cold leads are auto-booked into the next available viewing slot. Warm leads are queued for agent follow-up.

## Scoring Logic

### Hot Lead (score immediately, notify agent)
- Has sold (or is a first-time buyer/cash buyer) AND has AIP AND timeline is ASAP or 1-3 months
- OR: cash buyer with any timeline

### Warm Lead (queue for agent follow-up within 4 hours)
- Has AIP but hasn't sold yet (STC or under offer)
- OR: has sold but no AIP
- OR: timeline is 3-6 months regardless of other factors

### Cold Lead (auto-book viewing, minimal agent involvement)
- No AIP and not yet listed/sold
- OR: timeline is 6+ months or "just browsing"
- OR: unresponsive after qualification attempt

## Build Sequence

### Step 1: Scoring Function
```javascript
function scoreLead(qualificationData) {
  const { chain_position, has_aip, timeline } = qualificationData;
  
  // Cash buyer is always hot
  if (chain_position === 'cash_buyer') return 'hot';
  
  // Hot: ready to go
  const chainReady = ['first_time_buyer', 'sold_stc'].includes(chain_position);
  const timelineUrgent = ['asap', '1_3_months'].includes(timeline);
  if (chainReady && has_aip && timelineUrgent) return 'hot';
  
  // Cold: not ready
  const chainNotReady = ['not_yet_listed', 'renting'].includes(chain_position) && !has_aip;
  const timelineSlow = ['6_plus_months', 'just_browsing'].includes(timeline);
  if (chainNotReady || timelineSlow) return 'cold';
  
  // Everything else is warm
  return 'warm';
}
```

### Step 2: Routing Handler
```javascript
// handlers/leadScoring.js
// Triggered after AI qualification completes

async function handler(event) {
  const { leadId, tenantId, qualificationData } = event;
  
  const score = scoreLead(qualificationData);
  
  // Update lead score
  await db.query(
    'UPDATE leads SET score = $1, updated_at = NOW() WHERE id = $2 AND tenant_id = $3',
    [score, leadId, tenantId]
  );
  
  // Route based on score
  switch (score) {
    case 'hot':
      await routeHotLead(leadId, tenantId);
      break;
    case 'warm':
      await routeWarmLead(leadId, tenantId);
      break;
    case 'cold':
      await routeColdLead(leadId, tenantId);
      break;
  }
}
```

### Step 3: Hot Lead — Instant Agent Notification
```javascript
async function routeHotLead(leadId, tenantId) {
  const lead = await getLeadWithProperty(leadId);
  const agents = await getActiveAgents(tenantId);
  
  // Assign to agent (round-robin or first available)
  const assignedAgent = await assignAgent(tenantId, agents);
  await db.query('UPDATE leads SET assigned_agent_id = $1 WHERE id = $2', [assignedAgent.id, leadId]);
  
  // Send push notification via multiple channels:
  
  // 1. SMS to agent
  await twilioClient.messages.create({
    body: `🔥 HOT LEAD: ${lead.enquirer_name} for ${lead.property_address}. ` +
          `${lead.chain_position}, has AIP, wants to move ${lead.timeline}. ` +
          `Call them NOW: ${lead.enquirer_phone}`,
    from: PROPFLOW_NOTIFICATION_NUMBER,
    to: assignedAgent.phone
  });
  
  // 2. Dashboard notification (via WebSocket or polling)
  await createNotification(tenantId, assignedAgent.id, {
    type: 'hot_lead',
    leadId,
    title: `Hot lead: ${lead.enquirer_name}`,
    body: `Enquiry for ${lead.property_address}`,
    priority: 'urgent'
  });
}
```

### Step 4: Notifications Table
```sql
CREATE TABLE notifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  user_id UUID REFERENCES users(id), -- null = all agents in tenant
  type VARCHAR(50) NOT NULL,
  lead_id UUID REFERENCES leads(id),
  title VARCHAR(255) NOT NULL,
  body TEXT,
  priority VARCHAR(20) DEFAULT 'normal', -- normal/urgent
  read_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_notifications_user_unread ON notifications(user_id, read_at) WHERE read_at IS NULL;
```

### Step 5: Cold Lead — Auto-Book Viewing
```javascript
async function routeColdLead(leadId, tenantId) {
  const lead = await getLeadWithProperty(leadId);
  
  // Send SMS with self-serve booking link
  const bookingLink = `https://app.dis-rupt.com/book/${tenantId}/${leadId}`;
  
  await twilioClient.messages.create({
    body: `Hi ${lead.enquirer_name}, thanks for your interest in ${lead.property_address}. ` +
          `You can book a viewing at a time that suits you here: ${bookingLink}`,
    from: TENANT_PHONE,
    to: lead.enquirer_phone
  });
  
  // Also store as a message in the conversation
  await createMessage(lead.conversation_id, tenantId, 'outbound', 'sms', `Booking link sent`);
}
```

### Step 6: Warm Lead — Queued Follow-Up
```javascript
async function routeWarmLead(leadId, tenantId) {
  const agents = await getActiveAgents(tenantId);
  const assignedAgent = await assignAgent(tenantId, agents);
  
  await db.query('UPDATE leads SET assigned_agent_id = $1 WHERE id = $2', [assignedAgent.id, leadId]);
  
  // Create task for agent follow-up
  await createNotification(tenantId, assignedAgent.id, {
    type: 'warm_lead_followup',
    leadId,
    title: `Follow up: ${lead.enquirer_name}`,
    body: `Warm lead for ${lead.property_address}. Follow up within 4 hours.`,
    priority: 'normal'
  });
}
```

### Step 7: Agent Assignment Logic
```javascript
async function assignAgent(tenantId, agents) {
  if (agents.length === 1) return agents[0];
  
  // Round-robin: find agent with fewest active leads this week
  const result = await db.query(
    `SELECT u.id, u.name, COUNT(l.id) as lead_count
     FROM users u
     LEFT JOIN leads l ON l.assigned_agent_id = u.id 
       AND l.created_at > NOW() - INTERVAL '7 days'
     WHERE u.tenant_id = $1 AND u.is_active = TRUE AND u.role IN ('agent', 'admin', 'owner')
     GROUP BY u.id, u.name
     ORDER BY lead_count ASC
     LIMIT 1`,
    [tenantId]
  );
  return result;
}
```

## Testing Checklist
- [ ] Cash buyer scores as hot
- [ ] First-time buyer with AIP + ASAP timeline scores hot
- [ ] Buyer with no AIP and 6+ months scores cold
- [ ] All edge cases produce warm (the catch-all)
- [ ] Hot lead triggers SMS to assigned agent within 60 seconds
- [ ] Hot lead creates urgent dashboard notification
- [ ] Cold lead sends booking link SMS to enquirer
- [ ] Warm lead assigns agent and creates follow-up notification
- [ ] Round-robin assignment distributes evenly
- [ ] Single-agent tenant always assigns to that agent
- [ ] Unresponsive leads (from AutoPilot timeout) score as cold

## Dependencies
- **Requires:** Auth (1), Email Parser (2), AI Lead AutoPilot (3)
- **Required by:** Viewing Booking (5) — cold lead auto-booking links. Agent Dashboard (6) — notifications feed.

## What This Feature Provides to Others
- `score` field on lead records (hot/warm/cold)
- `assigned_agent_id` on lead records
- `notifications` table for the Agent Dashboard
- Booking link generation pattern for Viewing Booking feature
