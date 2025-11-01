# Signature Verification Debugging Guide

## Quick Diagnosis

Most common signature verification errors and their causes:

### Error: "No signatures found matching expected signature"

**Root causes (in order of frequency):**

1. **Wrong webhook secret (80%)**
   - Using test secret with live webhooks
   - Using live secret with test webhooks
   - Using old secret from deleted endpoint
   - Using API key (sk_...) instead of webhook secret (whsec_...)

2. **Parsed body before verification (15%)**
   - Framework parsed JSON before you could verify
   - Not using raw request body bytes
   - Converting body to string after parsing

3. **Clock skew (5%)**
   - Server time off by >5 minutes from UTC

### Error: "Unable to extract timestamp and signatures"

**Root causes:**
- Missing `Stripe-Signature` header
- Malformed signature header
- Reverse proxy stripping headers

### Error: "Timestamp outside tolerance zone"

**Root cause:**
- Server clock is off by more than 5 minutes from UTC time

## Systematic Debugging Process

### Step 1: Verify Secret Format

```bash
# Webhook secrets start with:
whsec_...

# API keys start with:
sk_test_...  # Test mode
sk_live_...  # Live mode

# If you're using an sk_ key, that's your problem!
```

**How to get the correct secret:**
1. Go to https://dashboard.stripe.com/webhooks
2. Click on your webhook endpoint
3. Click "Reveal" next to "Signing secret"
4. Copy that exact value (it starts with `whsec_`)

### Step 2: Check Test vs Live Mode

**Verify mode consistency:**
```
Where are webhooks coming from?
├─ Stripe Dashboard (test mode) → Use test webhook secret
├─ Stripe Dashboard (live mode) → Use live webhook secret
└─ Stripe CLI → Use secret from CLI output
```

**In Stripe Dashboard:**
- Look for "Viewing test data" toggle in top right
- Test webhooks = test webhook secret
- Live webhooks = live webhook secret

**Environment variable naming tip:**
```bash
# Good practice: separate test and live secrets
STRIPE_WEBHOOK_SECRET_TEST=whsec_test_...
STRIPE_WEBHOOK_SECRET_LIVE=whsec_live_...

# Check which one you're using in production
```

### Step 3: Verify Raw Body Usage

**The problem:**
Most frameworks parse JSON bodies by default. Stripe signature verification requires the raw bytes.

**Framework-specific guidance:**

#### Next.js App Router
```typescript
// ✅ CORRECT - use req.text() for raw body
export async function POST(req: NextRequest) {
  const body = await req.text();  // Raw string
  const sig = headers().get('stripe-signature');
  
  const event = stripe.webhooks.constructEvent(body, sig, secret);
}

// ❌ WRONG - req.json() parses the body
const body = await req.json();  // Already parsed!
```

#### Next.js Pages Router
```typescript
// Disable body parser for webhook route
export const config = {
  api: {
    bodyParser: false,  // CRITICAL
  },
};

// Use raw-body package
import getRawBody from 'raw-body';

export default async function handler(req, res) {
  const body = await getRawBody(req);  // Buffer
  const sig = req.headers['stripe-signature'];
  
  const event = stripe.webhooks.constructEvent(body, sig, secret);
}
```

#### Express
```typescript
// ✅ CORRECT - use express.raw() for webhook route
app.post('/webhook', 
  express.raw({ type: 'application/json' }),  // Raw body
  (req, res) => {
    const sig = req.headers['stripe-signature'];
    const event = stripe.webhooks.constructEvent(req.body, sig, secret);
  }
);

// ❌ WRONG - using express.json() parses the body
app.use(express.json());  // Parses all routes
app.post('/webhook', (req, res) => {
  // req.body is already parsed object!
});

// ✅ CORRECT ordering
app.post('/webhook', express.raw({ type: 'application/json' }), handler);
app.use(express.json());  // For other routes
```

#### FastAPI
```python
# ✅ CORRECT - use Request.body()
@app.post("/webhook")
async def webhook(request: Request):
    payload = await request.body()  # Raw bytes
    sig = request.headers.get('stripe-signature')
    
    event = stripe.Webhook.construct_event(payload, sig, secret)

# ❌ WRONG - Pydantic model parses body
@app.post("/webhook")
async def webhook(data: dict):  # Already parsed!
    # Can't verify signature with parsed data
```

#### Django
```python
# ✅ CORRECT - use request.body
def stripe_webhook(request):
    payload = request.body  # Raw bytes
    sig = request.META.get('HTTP_STRIPE_SIGNATURE')
    
    event = stripe.Webhook.construct_event(payload, sig, secret)

# ❌ WRONG - request.POST parses the body
def stripe_webhook(request):
    data = request.POST  # Already parsed!
```

### Step 4: Check Server Time

**Verify your server's time is correct:**
```bash
# Check current UTC time on server
date -u

# Compare with actual UTC time
# Visit: https://time.is/UTC

# If difference is >5 minutes, Stripe will reject webhooks
```

**How to fix clock skew:**
```bash
# On Ubuntu/Debian
sudo ntpdate -s time.nist.gov

# On modern systemd systems
sudo timedatectl set-ntp true

# Verify it worked
date -u
```

### Step 5: Verify Header Presence

**Check that Stripe-Signature header exists:**
```typescript
// Next.js
const sig = headers().get('stripe-signature');
if (!sig) {
  return NextResponse.json({ error: 'Missing signature' }, { status: 400 });
}

// Express
const sig = req.headers['stripe-signature'];
if (!sig) {
  return res.status(400).send('Missing signature');
}

// FastAPI
sig = request.headers.get('stripe-signature')
if not sig:
    raise HTTPException(status_code=400, detail="Missing signature")
```

**Common issues:**
- Reverse proxy (nginx, etc.) stripping headers
- CORS preflight removing headers
- Middleware modifying headers

## Testing Your Fix

### Local Testing with Stripe CLI

Most reliable way to test signature verification:

```bash
# 1. Install Stripe CLI
brew install stripe/stripe-cli/stripe

# 2. Login to your Stripe account
stripe login

# 3. Forward webhooks to your local server
stripe listen --forward-to localhost:3000/webhook

# This will output a webhook signing secret like:
# > Ready! Your webhook signing secret is whsec_abc123...
# Use this secret in your local .env file

# 4. In another terminal, trigger test events
stripe trigger payment_intent.succeeded

# 5. Check your server logs
# You should see the webhook processed successfully
```

If Stripe CLI works but production doesn't:
→ Definitely a test/live mode secret mismatch

### Production Testing

```bash
# 1. Go to Dashboard → Webhooks → Your endpoint
# 2. Click "Send test webhook"
# 3. Select an event type
# 4. Click "Send test webhook"
# 5. Check your application logs
```

If this fails:
→ Problem with your production configuration

## Common Gotchas

### Gotcha 1: Environment Variable Not Loaded

```typescript
// ❌ WRONG - might be undefined
const secret = process.env.STRIPE_WEBHOOK_SECRET;

// ✅ BETTER - assert it exists
const secret = process.env.STRIPE_WEBHOOK_SECRET!;
if (!secret) {
  throw new Error('STRIPE_WEBHOOK_SECRET not set');
}

// ✅ BEST - use a config module
import { config } from './config';
const secret = config.stripeWebhookSecret;  // Validated at startup
```

### Gotcha 2: Multiple Webhook Endpoints

If you have multiple endpoints (e.g., test and live):
- Each endpoint has its own webhook secret
- Make sure you're using the secret for the endpoint receiving webhooks

### Gotcha 3: Webhook Endpoint Was Recreated

If you deleted and recreated your webhook endpoint:
- The webhook secret changed
- Update your environment variable with new secret

### Gotcha 4: Using Payment Intent Secret

```typescript
// ❌ WRONG - payment intent client secret
const secret = payment_intent.client_secret;

// ✅ CORRECT - webhook signing secret
const secret = process.env.STRIPE_WEBHOOK_SECRET;
```

## Still Not Working?

If you've tried everything above and it's still not working:

1. **Enable debug logging:**
   ```typescript
   try {
     const event = stripe.webhooks.constructEvent(body, sig, secret);
   } catch (err) {
     console.error('Signature verification failed');
     console.error('Error message:', err.message);
     console.error('Signature header:', sig);
     console.error('Body length:', body.length);
     console.error('Body type:', typeof body);
     console.error('Secret starts with:', secret?.substring(0, 10));
   }
   ```

2. **Check Stripe Dashboard logs:**
   - Go to Webhooks → Your endpoint
   - Look at recent delivery attempts
   - Check the response code and body
   - Look for patterns in failures

3. **Verify endpoint configuration:**
   - Correct URL?
   - HTTPS not HTTP?
   - Publicly accessible?
   - Not behind authentication?

4. **Test in isolation:**
   - Create a minimal webhook handler
   - Just verify signature and return 200
   - No business logic
   - If this works, the problem is in your business logic

## Framework-Specific Complete Examples

### Next.js 14 App Router

```typescript
// app/api/webhooks/stripe/route.ts
import { headers } from 'next/headers';
import { NextRequest, NextResponse } from 'next/server';
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2023-10-16',
});

const secret = process.env.STRIPE_WEBHOOK_SECRET!;

export async function POST(req: NextRequest) {
  const body = await req.text();
  const signature = headers().get('stripe-signature');

  if (!signature) {
    return NextResponse.json(
      { error: 'No signature' },
      { status: 400 }
    );
  }

  try {
    const event = stripe.webhooks.constructEvent(body, signature, secret);
    console.log('✅ Webhook verified:', event.type);
    
    // Process event...
    
    return NextResponse.json({ received: true });
  } catch (err: any) {
    console.error('❌ Webhook verification failed:', err.message);
    return NextResponse.json(
      { error: err.message },
      { status: 400 }
    );
  }
}
```

### Express with TypeScript

```typescript
// routes/webhooks.ts
import express from 'express';
import Stripe from 'stripe';

const router = express.Router();
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);
const secret = process.env.STRIPE_WEBHOOK_SECRET!;

router.post('/stripe',
  express.raw({ type: 'application/json' }),
  async (req, res) => {
    const sig = req.headers['stripe-signature'];

    if (!sig) {
      return res.status(400).send('No signature');
    }

    try {
      const event = stripe.webhooks.constructEvent(req.body, sig, secret);
      console.log('✅ Webhook verified:', event.type);
      
      // Process event...
      
      res.json({ received: true });
    } catch (err: any) {
      console.error('❌ Webhook verification failed:', err.message);
      res.status(400).send(err.message);
    }
  }
);

export default router;
```

### FastAPI with Python

```python
# routers/webhooks.py
from fastapi import APIRouter, Request, HTTPException, Header
import stripe
import os

router = APIRouter()

stripe.api_key = os.environ['STRIPE_SECRET_KEY']
webhook_secret = os.environ['STRIPE_WEBHOOK_SECRET']

@router.post("/stripe")
async def stripe_webhook(
    request: Request,
    stripe_signature: str = Header(None)
):
    if not stripe_signature:
        raise HTTPException(status_code=400, detail="No signature")
    
    payload = await request.body()
    
    try:
        event = stripe.Webhook.construct_event(
            payload, stripe_signature, webhook_secret
        )
        print(f"✅ Webhook verified: {event['type']}")
        
        # Process event...
        
        return {"status": "success"}
    except Exception as e:
        print(f"❌ Webhook verification failed: {str(e)}")
        raise HTTPException(status_code=400, detail=str(e))
```
