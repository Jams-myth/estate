# PropFlow — Implementation Plan: Agent Dashboard

## PropFlow Context

PropFlow is a SaaS bundle for UK estate and letting agents, consolidating AI lead handling, digital onboarding, AML checks, and document signing into a single £149/month subscription. This document is one of 10 feature implementation plans that together form the complete MVP.

**Tech stack:** AWS Lambda + API Gateway, PostgreSQL (RDS), React + Tailwind, Clerk (auth), Stripe (billing), Twilio (comms), S3 (storage). All AWS eu-west-2.

**Build order:** This is **Feature 6 of 10**. Depends on Auth (1), Email Parser (2), AI Lead AutoPilot (3), Lead Scoring (4), Viewing Booking (5). This is primarily a frontend feature consuming APIs built in previous features.

---

## Feature Overview

Central web dashboard for agents. Shows all leads in a filterable feed, conversation threads with enquirers, Hot/Warm/Cold status tags, notifications, and a calendar view of upcoming viewings. This is where agents spend their day.

## Pages & Components

### Page 1: Lead Feed (Home)
- Default landing page after login
- Shows all leads for the tenant, newest first
- Each lead card shows:
  - Enquirer name, phone
  - Property address (truncated)
  - Source badge (Rightmove/Zoopla/OTM)
  - Score badge: 🔴 Hot / 🟡 Warm / 🔵 Cold
  - Qualification status
  - Assigned agent name
  - Time since received (relative: "3 min ago")
- Filters: score (hot/warm/cold/all), status (pending/in_progress/qualified/unresponsive), assigned agent, date range
- Search: by enquirer name, phone, property address

### API Endpoint
```javascript
// GET /api/leads?score=hot&status=qualified&page=1&limit=20
async function listLeads(tenantId, filters) {
  let query = `SELECT l.*, u.name as agent_name 
    FROM leads l 
    LEFT JOIN users u ON l.assigned_agent_id = u.id
    WHERE l.tenant_id = $1`;
  const params = [tenantId];
  
  if (filters.score) { query += ` AND l.score = $${params.push(filters.score)}`; }
  if (filters.status) { query += ` AND l.qualification_status = $${params.push(filters.status)}`; }
  if (filters.agentId) { query += ` AND l.assigned_agent_id = $${params.push(filters.agentId)}`; }
  if (filters.search) {
    query += ` AND (l.enquirer_name ILIKE $${params.push('%'+filters.search+'%')} 
      OR l.property_address ILIKE $${params.push('%'+filters.search+'%')}
      OR l.enquirer_phone ILIKE $${params.push('%'+filters.search+'%')})`;
  }
  
  query += ` ORDER BY l.received_at DESC LIMIT $${params.push(filters.limit)} OFFSET $${params.push(filters.offset)}`;
  return db.query(query, params);
}
```

### Page 2: Lead Detail / Conversation Thread
- Route: `/leads/:leadId`
- Top section: lead info (name, phone, email, property, score badge, qualification data)
- Action buttons: "Call", "Reassign", "Change Score", "Send Onboarding" (links to Digital Onboarding feature)
- Main section: full conversation thread (SMS messages in/out, displayed as chat bubbles)
- Agent can type a manual SMS reply from this page
- Timeline sidebar: key events (received, first response, qualified, viewing booked, etc.)

### API Endpoints
```javascript
// GET /api/leads/:id — full lead detail
// GET /api/leads/:id/messages — conversation thread
// POST /api/leads/:id/messages — agent sends manual SMS
// PATCH /api/leads/:id — update score, assignment, status
```

### Page 3: Calendar View
- Route: `/calendar`
- Weekly/daily view showing all viewings for the tenant
- Each viewing card: property address, enquirer name, time, assigned agent
- Colour-coded by agent
- Click viewing → opens lead detail
- Data from `viewings` table (Feature 5)

### Page 4: Notifications Panel
- Slide-out panel (not a full page)
- Shows unread notifications with badge count in nav
- Click notification → navigates to relevant lead
- Mark as read on click
- "Mark all read" button

### API Endpoints
```javascript
// GET /api/notifications?unread=true
// PATCH /api/notifications/:id/read
// PATCH /api/notifications/read-all
```

### Page 5: Settings
- Route: `/settings`
- Tabs:
  - **Agency Profile:** name, logo upload, contact details
  - **Team:** list agents, invite new agent, remove agent, change roles
  - **Calendar:** Nylas connection status, connect/disconnect button
  - **Lead Handling:** forwarding email address display + setup instructions
  - **Billing:** link to Stripe Customer Portal (Feature 9)

## Build Sequence

### Step 1: Layout Shell
- Sidebar navigation: Leads, Calendar, Settings
- Top bar: notification bell (with unread count), agent name/avatar, tenant name
- Responsive: sidebar collapses to hamburger on mobile
- Use Tailwind + a minimal component library (e.g. Headless UI for dropdowns/modals)

### Step 2: Lead Feed Page
- Build lead list component with filter bar
- Implement `/api/leads` endpoint with pagination
- Score badges as coloured pills
- Infinite scroll or pagination controls

### Step 3: Lead Detail / Conversation
- Chat-style message display (inbound left-aligned, outbound right-aligned)
- Manual message compose box at bottom
- Lead info card at top
- Action buttons wired to PATCH endpoints

### Step 4: Calendar View
- Use a React calendar library (e.g. `react-big-calendar` or custom)
- Fetch viewings from `/api/viewings?start=X&end=Y`
- Agent colour coding
- Click-to-navigate to lead detail

### Step 5: Notifications
- Poll `/api/notifications?unread=true` every 30 seconds (WebSockets are Phase 2)
- Badge count in nav
- Toast notification on new hot lead (using browser Notification API if permission granted)

### Step 6: Settings Pages
- Agency profile form (PATCH `/api/tenants/:id`)
- Team management (list, invite, remove)
- Calendar connection status
- Forwarding address display with copy-to-clipboard

## Key UI Components
- `LeadCard` — reusable lead summary card
- `ScoreBadge` — hot/warm/cold pill
- `SourceBadge` — rightmove/zoopla/otm
- `ChatBubble` — message in conversation thread
- `NotificationItem` — notification list item
- `FilterBar` — score/status/agent/search filters
- `ViewingCard` — calendar event card

## Testing Checklist
- [ ] Lead feed loads with correct tenant scoping
- [ ] Filters work correctly (score, status, agent, search)
- [ ] Lead detail shows full conversation thread
- [ ] Agent can send manual SMS from lead detail
- [ ] Calendar shows viewings for correct date range
- [ ] Notifications show unread count
- [ ] Click notification navigates to lead
- [ ] Settings pages load and save correctly
- [ ] Mobile responsive layout works
- [ ] No cross-tenant data visible
- [ ] Empty states handled (no leads yet, no viewings, etc.)

## Dependencies
- **Requires:** Auth (1), Email Parser (2), AI Lead AutoPilot (3), Lead Scoring (4), Viewing Booking (5)
- **Required by:** None directly — but all subsequent features (Onboarding, AML, Doc Signing) add pages/sections to this dashboard

## What This Feature Provides to Others
- Reusable layout shell and navigation for all future pages
- Component library (badges, cards, chat bubbles)
- Pattern for adding new pages/sections as features are built
- Settings infrastructure for tenant configuration
