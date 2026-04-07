# PropFlow — Implementation Plan: Stripe Billing

## PropFlow Context

PropFlow is a SaaS bundle for UK estate and letting agents, consolidating AI lead handling, digital onboarding, AML checks, and document signing into a single £149/month subscription. This document is one of 10 feature implementation plans that together form the complete MVP.

**Tech stack:** AWS Lambda + API Gateway, PostgreSQL (RDS), React + Tailwind, Clerk (auth), Stripe (billing), S3 (storage). All AWS eu-west-2.

**Build order:** This is **Feature 10 of 10**. Depends on Auth (1). Connected to ID & AML Checks (8) for usage metering. Can be built in parallel with other features from Step 1-3.

---

## Feature Overview

Stripe-powered subscription billing with three tiers (Starter/Professional/Agency) plus metered per-check billing for AML overages. Agents sign up, select a plan, manage their subscription via Stripe Customer Portal. Usage data (AML checks) flows from the AML feature into Stripe metered billing.

## Pricing Model

| Plan | Price | AML Allowance | Key Features |
|------|-------|---------------|--------------|
| Starter | £79/mo | 5 checks | AI Lead AutoPilot + basic onboarding |
| Professional | £149/mo | 10 checks | All features + unlimited doc signing |
| Agency | £249/mo | 25 checks | All features + white-label + priority support |
| AML Top-up | £2.50/check | — | Over monthly allowance |

## Build Sequence

### Step 1: Stripe Product & Price Setup
Create in Stripe Dashboard (or via API during deployment):

```javascript
// One-time setup script
async function setupStripeProducts() {
  // Create product
  const product = await stripe.products.create({
    name: 'PropFlow',
    description: 'AI-powered estate agency toolkit'
  });
  
  // Create subscription prices
  const starterPrice = await stripe.prices.create({
    product: product.id,
    unit_amount: 7900, // £79 in pence
    currency: 'gbp',
    recurring: { interval: 'month' },
    lookup_key: 'propflow_starter_monthly'
  });
  
  const proPrice = await stripe.prices.create({
    product: product.id,
    unit_amount: 14900,
    currency: 'gbp',
    recurring: { interval: 'month' },
    lookup_key: 'propflow_professional_monthly'
  });
  
  const agencyPrice = await stripe.prices.create({
    product: product.id,
    unit_amount: 24900,
    currency: 'gbp',
    recurring: { interval: 'month' },
    lookup_key: 'propflow_agency_monthly'
  });
  
  // AML top-up metered price
  const amlTopUp = await stripe.prices.create({
    product: product.id,
    unit_amount: 250, // £2.50 per check
    currency: 'gbp',
    recurring: {
      interval: 'month',
      usage_type: 'metered'
    },
    lookup_key: 'propflow_aml_topup'
  });
}
```

### Step 2: Checkout Flow
```javascript
// POST /api/billing/checkout
async function createCheckoutSession(tenantId, planKey) {
  const tenant = await getTenant(tenantId);
  
  // Create or retrieve Stripe customer
  let customerId = tenant.stripe_customer_id;
  if (!customerId) {
    const customer = await stripe.customers.create({
      email: tenant.owner_email,
      name: tenant.name,
      metadata: { tenant_id: tenantId }
    });
    customerId = customer.id;
    await db.query('UPDATE tenants SET stripe_customer_id = $1 WHERE id = $2', [customerId, tenantId]);
  }
  
  // Look up prices
  const prices = await stripe.prices.list({ lookup_keys: [planKey, 'propflow_aml_topup'] });
  const planPrice = prices.data.find(p => p.lookup_key === planKey);
  const amlPrice = prices.data.find(p => p.lookup_key === 'propflow_aml_topup');
  
  // Create checkout session with subscription + metered AML
  const session = await stripe.checkout.sessions.create({
    customer: customerId,
    mode: 'subscription',
    line_items: [
      { price: planPrice.id, quantity: 1 },
      { price: amlPrice.id } // metered — no quantity
    ],
    success_url: `https://app.dis-rupt.com/billing/success?session_id={CHECKOUT_SESSION_ID}`,
    cancel_url: 'https://app.dis-rupt.com/billing/cancel',
    metadata: { tenant_id: tenantId },
    tax_id_collection: { enabled: true }, // collect VAT number
    automatic_tax: { enabled: true }
  });
  
  return { url: session.url };
}
```

### Step 3: Stripe Webhook Handler
```javascript
// POST /api/webhooks/stripe
async function handleStripeWebhook(event) {
  const sig = event.headers['stripe-signature'];
  const stripeEvent = stripe.webhooks.constructEvent(event.body, sig, STRIPE_WEBHOOK_SECRET);
  
  switch (stripeEvent.type) {
    case 'checkout.session.completed': {
      const session = stripeEvent.data.object;
      const tenantId = session.metadata.tenant_id;
      
      // Update tenant with subscription info
      const subscription = await stripe.subscriptions.retrieve(session.subscription);
      const planItem = subscription.items.data.find(i => i.price.recurring.usage_type !== 'metered');
      const plan = mapPriceToPlan(planItem.price.lookup_key);
      
      await db.query(
        `UPDATE tenants SET 
          stripe_subscription_id = $1, plan = $2, 
          aml_checks_limit = $3, aml_checks_used = 0
         WHERE id = $4`,
        [subscription.id, plan, planToAMLLimit(plan), tenantId]
      );
      break;
    }
    
    case 'customer.subscription.updated': {
      const subscription = stripeEvent.data.object;
      const tenant = await db.query(
        'SELECT * FROM tenants WHERE stripe_subscription_id = $1',
        [subscription.id]
      );
      
      if (subscription.status === 'active') {
        const planItem = subscription.items.data.find(i => i.price.recurring.usage_type !== 'metered');
        const plan = mapPriceToPlan(planItem.price.lookup_key);
        await db.query(
          'UPDATE tenants SET plan = $1, aml_checks_limit = $2 WHERE id = $3',
          [plan, planToAMLLimit(plan), tenant.id]
        );
      }
      break;
    }
    
    case 'customer.subscription.deleted': {
      const subscription = stripeEvent.data.object;
      await db.query(
        `UPDATE tenants SET plan = 'cancelled', stripe_subscription_id = NULL WHERE stripe_subscription_id = $1`,
        [subscription.id]
      );
      // TODO: handle grace period, data retention policy
      break;
    }
    
    case 'invoice.payment_failed': {
      const invoice = stripeEvent.data.object;
      // Notify tenant owner about failed payment
      const tenant = await db.query(
        'SELECT * FROM tenants WHERE stripe_customer_id = $1',
        [invoice.customer]
      );
      await createNotification(tenant.id, null, {
        type: 'payment_failed',
        title: 'Payment failed',
        body: 'Please update your payment method to avoid service interruption.'
      });
      break;
    }
  }
}

function planToAMLLimit(plan) {
  switch (plan) {
    case 'starter': return 5;
    case 'professional': return 10;
    case 'agency': return 25;
    default: return 0;
  }
}
```

### Step 4: AML Usage Metering
```javascript
// Called by the AML Checks feature when a check exceeds monthly allowance
async function recordAMLOverage(tenantId, checkId) {
  const tenant = await getTenant(tenantId);
  const subscription = await stripe.subscriptions.retrieve(tenant.stripe_subscription_id);
  
  // Find the metered subscription item
  const meteredItem = subscription.items.data.find(
    i => i.price.recurring.usage_type === 'metered'
  );
  
  // Report usage to Stripe
  await stripe.subscriptionItems.createUsageRecord(meteredItem.id, {
    quantity: 1,
    timestamp: Math.floor(Date.now() / 1000),
    action: 'increment'
  });
  
  // Mark check as billed
  await db.query('UPDATE aml_checks SET billed = TRUE WHERE id = $1', [checkId]);
}
```

### Step 5: Customer Portal
```javascript
// POST /api/billing/portal
async function createPortalSession(tenantId) {
  const tenant = await getTenant(tenantId);
  
  const session = await stripe.billingPortal.sessions.create({
    customer: tenant.stripe_customer_id,
    return_url: 'https://app.dis-rupt.com/settings/billing'
  });
  
  return { url: session.url };
}
```
- Agents can: update payment method, view invoices, change plan, cancel subscription
- Configure Stripe Customer Portal in Dashboard: enable plan switching, invoice history, payment method update

### Step 6: Dashboard — Billing Page
- Route: `/settings/billing`
- Shows:
  - Current plan name + price
  - Subscription status (active/past_due/cancelled)
  - AML usage: "X of Y checks used this month" + overage count
  - Next billing date
  - "Manage Subscription" button → Stripe Customer Portal
  - "Upgrade Plan" / "Downgrade Plan" buttons → Stripe Checkout with new price
  - Invoice history (via Stripe API)

### Step 7: Plan Gating
```javascript
// Middleware to check feature access based on plan
function requirePlan(requiredPlans) {
  return async (tenantId) => {
    const tenant = await getTenant(tenantId);
    if (!requiredPlans.includes(tenant.plan)) {
      throw new ForbiddenError(`This feature requires ${requiredPlans.join(' or ')} plan`);
    }
  };
}

// Usage:
// await requirePlan(['professional', 'agency'])(tenantId); // before doc signing
// await requirePlan(['agency'])(tenantId); // before white-label features
```

Feature access by plan:
| Feature | Starter | Professional | Agency |
|---------|---------|-------------|--------|
| AI Lead AutoPilot | ✅ | ✅ | ✅ |
| Digital Onboarding | Basic | Full | Full |
| AML Checks | 5/mo | 10/mo | 25/mo |
| Document Signing | ❌ | Unlimited | Unlimited |
| White-label | ❌ | ❌ | ✅ |
| Priority support | ❌ | ❌ | ✅ |

## Environment Variables Required
```
STRIPE_SECRET_KEY=
STRIPE_PUBLISHABLE_KEY=
STRIPE_WEBHOOK_SECRET=
```

## Testing Checklist
- [ ] Checkout session creates subscription in Stripe
- [ ] Webhook updates tenant plan and AML limit correctly
- [ ] Plan change (upgrade/downgrade) reflected in tenant record
- [ ] Subscription cancellation handled gracefully
- [ ] AML overage usage reported to Stripe correctly
- [ ] Customer Portal accessible and functional
- [ ] Failed payment triggers notification to tenant
- [ ] Plan gating restricts features correctly per tier
- [ ] Billing page shows correct usage data
- [ ] VAT/tax collection works for UK business customers
- [ ] Test mode works in development (Stripe test keys)

## Dependencies
- **Requires:** Auth (1) — tenant records
- **Required by:** ID & AML Checks (8) — overage metering

## What This Feature Provides to Others
- `plan` field on tenant for feature gating across all features
- `aml_checks_limit` for the AML feature to check against
- `stripe_customer_id` and `stripe_subscription_id` on tenant record
- `requirePlan()` middleware for feature access control
- Customer Portal for self-serve billing management
