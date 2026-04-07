# PropFlow — Pre-Build Setup Checklist

Everything you need to create/configure manually before Claude Code can start building. Ordered by when you need them (Session 1 first, then by feature).

---

## Session 1: Foundation (Needed Before Any Code)

### AWS Account
- [ ] Create AWS account (or use existing)
- [ ] Create IAM user with programmatic access for deployment
- [ ] Attach policies: `AmazonRDSFullAccess`, `AmazonS3FullAccess`, `AmazonSESFullAccess`, `AWSLambda_FullAccess`, `AmazonAPIGatewayAdministrator`, `CloudWatchLogsFullAccess`
- [ ] Save `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`
- [ ] Set default region: `eu-west-2` (London)

### AWS RDS (PostgreSQL)
- [ ] Create PostgreSQL 16 instance in eu-west-2
  - Instance class: `db.t3.micro` (free tier eligible)
  - Storage: 20GB gp3
  - DB name: `propflow`
  - Master username: `propflow`
  - Set a strong master password
  - VPC: default VPC, publicly accessible (for dev — lock down before production)
  - Security group: allow inbound PostgreSQL (port 5432) from your IP and Lambda security group
- [ ] Save `DATABASE_URL=postgresql://propflow:{password}@{endpoint}:5432/propflow`

### AWS S3 Buckets
- [ ] Create bucket: `propflow-emails-dev` (eu-west-2, private, versioning on)
- [ ] Create bucket: `propflow-documents-dev` (eu-west-2, private, versioning on)
- [ ] On `propflow-documents-dev`: enable S3 Object Lock (for HMRC 5-year AML audit retention)

### Clerk
- [ ] Sign up at clerk.com
- [ ] Create application named "PropFlow"
- [ ] Enable sign-in methods: Email + Password
- [ ] Enable Organisations feature (Settings → Organisations → Enable)
- [ ] Create development instance
- [ ] Save `CLERK_PUBLISHABLE_KEY` (starts with `pk_test_`)
- [ ] Save `CLERK_SECRET_KEY` (starts with `sk_test_`)
- [ ] Go to Webhooks → Add endpoint: `https://{your-api-domain}/api/webhooks/clerk`
  - Subscribe to: `user.created`, `organization.created`, `organization.membership.created`
  - Save `CLERK_WEBHOOK_SECRET` (starts with `whsec_`)
- [ ] Note: webhook URL can be a placeholder until API is deployed — update it after Session 1

### Node.js & Tooling (Local Dev)
- [ ] Node.js 20.x installed
- [ ] npm or yarn installed
- [ ] AWS CLI v2 installed and configured (`aws configure` with your IAM credentials)
- [ ] Serverless Framework v4 installed: `npm install -g serverless`
- [ ] PostgreSQL client installed (for running migrations): `psql` or a GUI like TablePlus/pgAdmin

### Domain & DNS
- [ ] Register `propflow.co.uk` (or your chosen domain)
- [ ] Set up DNS with a provider you can manage (Cloudflare recommended, or Route 53)
- [ ] You'll configure specific DNS records later (SES, API Gateway, CloudFront)

---

## Session 2: Email Parser + AI Lead AutoPilot

### AWS SES (Email Receiving)
- [ ] In SES console (eu-west-2), go to Verified Identities → Add domain: `propflow.co.uk`
- [ ] Add the DKIM CNAME records SES provides to your DNS
- [ ] Add MX record to DNS: `leads.propflow.co.uk` → `10 inbound-smtp.eu-west-2.amazonaws.com`
- [ ] Wait for domain verification (can take up to 72 hours — start this early)
- [ ] Create SES Receipt Rule Set (Claude Code can do this via CloudFormation, but domain must be verified first)

### OpenAI
- [ ] Sign up at platform.openai.com
- [ ] Add payment method (pay-as-you-go)
- [ ] Generate API key
- [ ] Save `OPENAI_API_KEY` (starts with `sk-`)
- [ ] Estimated cost: ~£5-15/month at MVP volume

### Twilio
- [ ] Sign up at twilio.com
- [ ] Upgrade from trial account (trial numbers can't send to unverified numbers)
- [ ] Buy a UK phone number (local number, SMS-capable): ~£1/month
- [ ] Save `TWILIO_ACCOUNT_SID` (starts with `AC`)
- [ ] Save `TWILIO_AUTH_TOKEN`
- [ ] Save `TWILIO_PHONE_NUMBER` (the UK number you bought, format: `+44...`)
- [ ] Configure SMS webhook URL: `https://{your-api-domain}/api/webhooks/twilio/sms` (set after API deployed)
- [ ] Optional: buy additional numbers for multi-tenant (one per agency) — can start with one shared number

---

## Session 4: Viewing Booking

### Nylas
- [ ] Sign up at nylas.com (developer account — free tier available)
- [ ] Create application named "PropFlow"
- [ ] Save `NYLAS_CLIENT_ID`
- [ ] Save `NYLAS_API_KEY`
- [ ] Set callback URL: `https://{your-app-domain}/auth/nylas/callback`
- [ ] Save `NYLAS_CALLBACK_URL`
- [ ] Note: individual agents will connect their Google/Outlook calendars via OAuth within the app — you don't need to do this yourself

---

## Session 7: AML Checks + Stripe Billing

### ComplyCube
- [ ] Sign up at complycube.com
- [ ] Apply for startup programme (up to $50k credits) — do this early, approval takes time
- [ ] Access developer sandbox (immediate, no sales call needed)
- [ ] Generate API key from sandbox
- [ ] Save `COMPLYCUBE_API_KEY`
- [ ] Configure webhook URL: `https://{your-api-domain}/api/webhooks/complycube`
  - Subscribe to: `check.completed`
- [ ] Save `COMPLYCUBE_WEBHOOK_SECRET`
- [ ] Test with sandbox credentials before going live

### Stripe
- [ ] Sign up at stripe.com (or use existing account)
- [ ] Complete business verification (UK business details)
- [ ] Save `STRIPE_SECRET_KEY` (starts with `sk_test_` for dev)
- [ ] Save `STRIPE_PUBLISHABLE_KEY` (starts with `pk_test_`)
- [ ] Create webhook endpoint: `https://{your-api-domain}/api/webhooks/stripe`
  - Subscribe to: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
- [ ] Save `STRIPE_WEBHOOK_SECRET` (starts with `whsec_`)
- [ ] Enable Customer Portal (Settings → Billing → Customer Portal)
  - Enable: update payment method, view invoices, cancel subscription, switch plans
- [ ] Enable Tax settings if charging VAT (Settings → Tax)
- [ ] Note: the actual Products and Prices (Starter/Pro/Agency) will be created by Claude Code via the Stripe API during the build

---

## Session 8: Document Signing

### Dropbox Sign (HelloSign)
- [ ] Sign up at sign.dropbox.com/developers
- [ ] Create API app
- [ ] Save `HELLOSIGN_API_KEY`
- [ ] Save `HELLOSIGN_CLIENT_ID`
- [ ] Configure webhook callback URL: `https://{your-api-domain}/api/webhooks/hellosign`
- [ ] Enable test mode for development
- [ ] Note: there is no separate webhook secret — HelloSign uses an event hash for verification

---

## Post-Build: Before Go-Live

### DNS Records (Final)
- [ ] A/CNAME record for `app.propflow.co.uk` → CloudFront distribution (frontend)
- [ ] A/CNAME record for `api.propflow.co.uk` → API Gateway custom domain
- [ ] MX record for `leads.propflow.co.uk` → SES inbound (should already be done)
- [ ] DKIM/SPF/DMARC records for `propflow.co.uk` (for SES sending)

### SSL Certificates
- [ ] Request SSL cert in AWS Certificate Manager for `*.propflow.co.uk`
- [ ] Attach to CloudFront distribution and API Gateway custom domain

### Switch API Keys to Production
- [ ] Clerk: create production instance, swap keys
- [ ] Stripe: swap `sk_test_` → `sk_live_` keys
- [ ] ComplyCube: swap sandbox → production API key
- [ ] Dropbox Sign: disable test mode
- [ ] Twilio: already production if upgraded from trial

### Security
- [ ] Lock down RDS: remove public access, restrict to Lambda security group only
- [ ] Enable AWS CloudTrail for audit logging
- [ ] Enable RDS automated backups (7-day retention minimum)
- [ ] Review S3 bucket policies — ensure no public access
- [ ] Set up AWS billing alerts

### Legal
- [ ] Privacy Policy published on `propflow.co.uk/privacy`
- [ ] Terms of Service published on `propflow.co.uk/terms`
- [ ] Data Processing Agreement (DPA) template ready for agencies
- [ ] ICO registration (UK data protection registration)
- [ ] HMRC AML supervisory registration (if acting as AML service provider)

---

## Summary: All Environment Variables

Create a `.env` file with all of these before Session 1. Fill in values as you complete each section above.

```
# AWS
AWS_REGION=eu-west-2
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
S3_EMAIL_BUCKET=propflow-emails-dev
S3_DOCUMENTS_BUCKET=propflow-documents-dev

# Database
DATABASE_URL=postgresql://propflow:PASSWORD@ENDPOINT:5432/propflow

# Clerk
CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...
CLERK_WEBHOOK_SECRET=whsec_...

# Stripe
STRIPE_SECRET_KEY=sk_test_...
STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Twilio
TWILIO_ACCOUNT_SID=AC...
TWILIO_AUTH_TOKEN=
TWILIO_PHONE_NUMBER=+44...

# OpenAI
OPENAI_API_KEY=sk-...

# Nylas
NYLAS_CLIENT_ID=
NYLAS_API_KEY=
NYLAS_CALLBACK_URL=http://localhost:3000/auth/nylas/callback

# ComplyCube
COMPLYCUBE_API_KEY=
COMPLYCUBE_WEBHOOK_SECRET=

# Dropbox Sign
HELLOSIGN_API_KEY=
HELLOSIGN_CLIENT_ID=

# App
APP_URL=http://localhost:3000
API_URL=http://localhost:4000
NODE_ENV=development
```

---

## What You DON'T Need to Set Up Manually

These are handled by Claude Code during the build:
- Serverless Framework configuration (`serverless.yml`)
- Lambda functions and API Gateway routes
- Database tables (migration file)
- SES receipt rules (via CloudFormation)
- Stripe Products and Prices (via Stripe API)
- React app scaffolding
- All application code
