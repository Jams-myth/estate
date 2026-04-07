# PropFlow — Implementation Plan: Viewing Booking

## PropFlow Context

PropFlow is a SaaS bundle for UK estate and letting agents, consolidating AI lead handling, digital onboarding, AML checks, and document signing into a single £149/month subscription. This document is one of 10 feature implementation plans that together form the complete MVP.

**Tech stack:** AWS Lambda + API Gateway, PostgreSQL (RDS), React + Tailwind, Clerk (auth), Stripe (billing), Twilio (comms), Nylas (calendar), S3 (storage). All AWS eu-west-2.

**Build order:** This is **Feature 5 of 10**. Depends on Auth (1), Lead Scoring & Routing (4). Used by Agent Dashboard (6).

---

## Feature Overview

Self-serve viewing booking via Nylas Calendar Sync API. Enquirers receive a booking link (from Lead Scoring for cold leads, or from AI AutoPilot during qualification). They select an available slot from the agent's real calendar. No agent involvement required for the booking itself.

## How It Works (Flow)

1. Enquirer receives booking link via SMS: `https://app.dis-rupt.com/book/{tenant_id}/{lead_id}`
2. Public booking page shows available slots from the assigned agent's calendar (via Nylas)
3. Enquirer selects a slot → booking created in agent's calendar → confirmation SMS sent to both parties
4. Agent sees booking in their dashboard calendar view

## Build Sequence

### Step 1: Nylas Integration Setup
- Create Nylas developer account
- OAuth flow for agents to connect their Google/Outlook calendar:
  - Agent goes to Settings → Calendar → "Connect Calendar"
  - Nylas hosted auth flow → callback stores `nylas_grant_id` on user record
```sql
ALTER TABLE users ADD COLUMN nylas_grant_id VARCHAR(255);
ALTER TABLE users ADD COLUMN nylas_calendar_id VARCHAR(255);
```

### Step 2: Viewings Table
```sql
CREATE TABLE viewings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  lead_id UUID NOT NULL REFERENCES leads(id),
  agent_id UUID NOT NULL REFERENCES users(id),
  property_address TEXT NOT NULL,
  
  -- Timing
  starts_at TIMESTAMPTZ NOT NULL,
  ends_at TIMESTAMPTZ NOT NULL,
  duration_minutes INTEGER DEFAULT 30,
  
  -- External refs
  nylas_event_id VARCHAR(255),
  
  -- Status
  status VARCHAR(30) DEFAULT 'confirmed', -- confirmed/cancelled/completed/no_show
  cancelled_by VARCHAR(20), -- enquirer/agent
  cancellation_reason TEXT,
  
  -- Attendee
  attendee_name VARCHAR(255) NOT NULL,
  attendee_phone VARCHAR(50),
  attendee_email VARCHAR(255),
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_viewings_tenant ON viewings(tenant_id);
CREATE INDEX idx_viewings_agent_date ON viewings(agent_id, starts_at);
CREATE INDEX idx_viewings_lead ON viewings(lead_id);
```

### Step 3: Availability Endpoint
```javascript
// GET /api/bookings/availability?tenant_id=X&lead_id=Y&date=2026-04-15

async function getAvailability(tenantId, leadId, date) {
  const lead = await getLeadWithAgent(leadId, tenantId);
  const agent = await getAgent(lead.assigned_agent_id || await getDefaultAgent(tenantId));
  
  if (!agent.nylas_grant_id) {
    // Agent hasn't connected calendar — return default business hours
    return generateDefaultSlots(date);
  }
  
  // Get agent's existing events from Nylas
  const startOfDay = new Date(date);
  startOfDay.setHours(9, 0, 0); // Business hours: 9am
  const endOfDay = new Date(date);
  endOfDay.setHours(18, 0, 0); // to 6pm
  
  const events = await nylas.events.list({
    identifier: agent.nylas_grant_id,
    calendarId: agent.nylas_calendar_id,
    queryParams: {
      start: Math.floor(startOfDay.getTime() / 1000),
      end: Math.floor(endOfDay.getTime() / 1000)
    }
  });
  
  // Generate 30-min slots, excluding busy times
  const slots = generateAvailableSlots(startOfDay, endOfDay, events.data, 30);
  return slots;
}
```

### Step 4: Public Booking Page (React)
- Route: `/book/:tenantId/:leadId` — **no auth required** (public page)
- Branded with tenant's name/logo
- Shows:
  - Property address and photo (if available)
  - Date picker (next 14 days)
  - Available time slots for selected date
  - "Book Viewing" button
- On submit → calls booking API → shows confirmation

### Step 5: Create Booking Endpoint
```javascript
// POST /api/bookings

async function createBooking(tenantId, leadId, slotStart) {
  const lead = await getLeadWithAgent(leadId, tenantId);
  const agent = await getAgent(lead.assigned_agent_id);
  
  const slotEnd = new Date(slotStart.getTime() + 30 * 60 * 1000);
  
  // Create event in agent's calendar via Nylas
  let nylasEventId = null;
  if (agent.nylas_grant_id) {
    const event = await nylas.events.create({
      identifier: agent.nylas_grant_id,
      requestBody: {
        calendarId: agent.nylas_calendar_id,
        title: `Viewing: ${lead.property_address} — ${lead.enquirer_name}`,
        when: {
          startTime: Math.floor(slotStart.getTime() / 1000),
          endTime: Math.floor(slotEnd.getTime() / 1000)
        },
        description: `PropFlow viewing booking\nEnquirer: ${lead.enquirer_name}\nPhone: ${lead.enquirer_phone}\nProperty: ${lead.property_address}`,
        location: lead.property_address
      }
    });
    nylasEventId = event.data.id;
  }
  
  // Create viewing record
  const viewing = await db.query(
    `INSERT INTO viewings (tenant_id, lead_id, agent_id, property_address,
     starts_at, ends_at, nylas_event_id, attendee_name, attendee_phone, attendee_email)
     VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9,$10) RETURNING *`,
    [tenantId, leadId, agent.id, lead.property_address,
     slotStart, slotEnd, nylasEventId, lead.enquirer_name, lead.enquirer_phone, lead.enquirer_email]
  );
  
  // Send confirmation SMS to enquirer
  await twilioClient.messages.create({
    body: `Your viewing at ${lead.property_address} is confirmed for ${formatDateTime(slotStart)}. See you there!`,
    from: TENANT_PHONE,
    to: lead.enquirer_phone
  });
  
  // Notify agent
  await createNotification(tenantId, agent.id, {
    type: 'viewing_booked',
    leadId,
    title: `Viewing booked: ${lead.enquirer_name}`,
    body: `${lead.property_address} — ${formatDateTime(slotStart)}`
  });
  
  return viewing;
}
```

### Step 6: Cancellation & Rescheduling
- Enquirer can cancel via a link in the confirmation SMS
- Agent can cancel from the dashboard
- On cancel → Nylas event deleted → SMS notification to other party
- Reschedule = cancel + new booking

### Step 7: Reminder SMS
- Schedule via EventBridge: 24 hours before and 1 hour before viewing
- `"Reminder: Your viewing at {property_address} is tomorrow at {time}. Reply CANCEL to cancel."`

## Environment Variables Required
```
NYLAS_CLIENT_ID=
NYLAS_API_KEY=
NYLAS_CALLBACK_URL=https://app.dis-rupt.com/auth/nylas/callback
```

## Testing Checklist
- [ ] Nylas OAuth flow connects agent's calendar
- [ ] Availability endpoint returns correct free slots
- [ ] Busy calendar times are excluded from available slots
- [ ] Booking creates event in agent's Google/Outlook calendar
- [ ] Confirmation SMS sent to enquirer
- [ ] Agent receives dashboard notification
- [ ] Cancellation removes calendar event + notifies parties
- [ ] Reminder SMS fires 24hr and 1hr before viewing
- [ ] Double-booking prevented (slot rechecked at booking time)
- [ ] Public booking page loads without authentication
- [ ] Agent without connected calendar gets default business-hours slots

## Dependencies
- **Requires:** Auth (1), Lead Scoring & Routing (4) — booking links generated there
- **Required by:** Agent Dashboard (6) — calendar view of viewings

## What This Feature Provides to Others
- `viewings` table for the Agent Dashboard calendar view
- Public booking page pattern reusable for other self-serve flows
- Nylas integration available for future calendar features
