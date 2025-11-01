# Stripe Webhook Testing Guide

Complete guide to testing Stripe webhooks locally and in production.

## Local Testing with Stripe CLI

### Installation

**macOS:**
```bash
brew install stripe/stripe-cli/stripe
```

**Linux:**
```bash
# Download and install
curl -s https://packages.stripe.dev/api/security/keypair/stripe-cli-gpg/public | gpg --dearmor | sudo tee /usr/share/keyrings/stripe.gpg
echo "deb [signed-by=/usr/share/keyrings/stripe.gpg] https://packages.stripe.dev/stripe-cli-debian-local stable main" | sudo tee -a /etc/apt/sources.list.d/stripe.list
sudo apt update
sudo apt install stripe
```

**Windows:**
```bash
# Using Scoop
scoop bucket add stripe https://github.com/stripe/scoop-stripe-cli.git
scoop install stripe
```

### Initial Setup

```bash
# Login to your Stripe account
stripe login

# This opens browser for authentication
# Generates restricted API key automatically
```

### Forwarding Webhooks

**Basic forwarding:**
```bash
# Forward webhooks to local endpoint
stripe listen --forward-to localhost:3000/api/webhooks/stripe

# Output shows:
# > Ready! Your webhook signing secret is whsec_abc123...
# Copy this secret to your .env file
```

**Framework-specific:**
```bash
# Next.js (default port 3000)
stripe listen --forward-to localhost:3000/api/webhooks/stripe

# Express (custom port)
stripe listen --forward-to localhost:4000/webhook

# FastAPI (default port 8000)
stripe listen --forward-to localhost:8000/webhooks/stripe

# Django (default port 8000)
stripe listen --forward-to localhost:8000/webhooks/stripe/
```

**Filter specific events:**
```bash
# Only forward subscription events
stripe listen \
  --events customer.subscription.created,customer.subscription.updated,customer.subscription.deleted \
  --forward-to localhost:3000/webhook

# Only forward payment events
stripe listen \
  --events payment_intent.succeeded,payment_intent.payment_failed \
  --forward-to localhost:3000/webhook
```

**Load webhooks from config file:**
```bash
# Create .stripeclirc
{
  "listen": {
    "forward_to": "localhost:3000/api/webhooks/stripe",
    "events": ["payment_intent.*", "invoice.*"]
  }
}

# Run with config
stripe listen --load-from-webhooks-api
```

### Triggering Test Events

**Basic triggers:**
```bash
# Successful payment
stripe trigger payment_intent.succeeded

# Failed payment
stripe trigger payment_intent.payment_failed

# New subscription
stripe trigger customer.subscription.created

# Subscription updated
stripe trigger customer.subscription.updated

# Subscription canceled
stripe trigger customer.subscription.deleted

# Invoice paid
stripe trigger invoice.paid

# Invoice payment failed
stripe trigger invoice.payment_failed

# Refund
stripe trigger charge.refunded

# Checkout completed
stripe trigger checkout.session.completed
```

**With custom data:**
```bash
# Custom amount
stripe trigger payment_intent.succeeded --amount 5000

# Custom metadata
stripe trigger payment_intent.succeeded \
  --add payment_intent:metadata.order_id=12345 \
  --add payment_intent:metadata.user_id=user_abc

# Custom customer
stripe trigger customer.subscription.created \
  --add subscription:customer=cus_abc123
```

**See full payload:**
```bash
# Debug mode shows complete JSON
stripe trigger payment_intent.succeeded --log-level debug
```

**Trigger from file:**
```bash
# Create fixture file: fixtures/payment_failed.json
{
  "amount": 2000,
  "currency": "usd",
  "last_payment_error": {
    "decline_code": "insufficient_funds",
    "message": "Your card has insufficient funds."
  }
}

# Trigger with fixture
stripe trigger payment_intent.payment_failed \
  --override fixtures/payment_failed.json
```

### Monitoring Webhooks

**Watch webhook activity:**
```bash
# Forward webhooks and show activity
stripe listen --forward-to localhost:3000/webhook --print-json

# Output shows:
# âœ" Received event payment_intent.succeeded [evt_123]
# âœ" Successfully POSTed to local endpoint
```

**View recent events:**
```bash
# List recent events
stripe events list --limit 10

# Get specific event
stripe events retrieve evt_abc123

# Resend event to webhook
stripe events resend evt_abc123
```

### Troubleshooting CLI Issues

**CLI not forwarding:**
```bash
# Check CLI version
stripe --version

# Update CLI
stripe update  # or brew upgrade stripe

# Check login status
stripe config --list

# Re-login if needed
stripe login
```

**Webhook secret mismatch:**
```bash
# CLI outputs webhook secret when you start listening
# Copy exact secret to .env
# Format: whsec_abc123...

# Verify environment variable
echo $STRIPE_WEBHOOK_SECRET
```

**Endpoint not accessible:**
```bash
# Check your server is running
curl http://localhost:3000/api/webhooks/stripe

# Check firewall/proxy settings
# Make sure port is not blocked
```

## Test Cards

### Successful Payments

```
4242 4242 4242 4242 - Always succeeds
```
- Any future expiry date
- Any 3-digit CVC
- Any postal code

### Declined Cards

```
4000 0000 0000 0002 - Generic decline
4000 0000 0000 9995 - Insufficient funds
4000 0000 0000 9987 - Lost card
4000 0000 0000 9979 - Stolen card
4000 0000 0000 0069 - Expired card
4000 0000 0000 0127 - Incorrect CVC
4000 0000 0000 0119 - Processing error
4100 0000 0000 0019 - Card declined (fraudulent)
```

### Authentication Required (3D Secure)

```
4000 0025 0000 3155 - Requires authentication
4000 0027 6000 3184 - Requires authentication (2)
```

### Testing Specific Scenarios

**Insufficient funds:**
```bash
# Use test card
4000 0000 0000 9995

# Or trigger event
stripe trigger payment_intent.payment_failed \
  --add payment_intent:last_payment_error.decline_code=insufficient_funds
```

**Expired card:**
```bash
# Use test card
4000 0000 0000 0069

# Or trigger event
stripe trigger payment_intent.payment_failed \
  --add payment_intent:last_payment_error.decline_code=expired_card
```

**3D Secure flow:**
```bash
# Use test card (requires authentication)
4000 0025 0000 3155

# This triggers:
# 1. payment_intent.created
# 2. payment_intent.requires_action (redirect customer)
# 3. payment_intent.succeeded (after authentication)

# Or trigger directly
stripe trigger payment_intent.requires_action
```

## Production Testing

### Dashboard Testing

**Send test webhook:**
1. Go to https://dashboard.stripe.com/webhooks
2. Click on your webhook endpoint
3. Click "Send test webhook"
4. Select event type
5. Click "Send test webhook"
6. Check logs in your application

**View webhook history:**
1. Go to https://dashboard.stripe.com/webhooks
2. Click on your endpoint
3. See recent delivery attempts
4. Check success/failure status
5. View request/response details

**Retry failed webhook:**
1. Find failed webhook in history
2. Click on the event
3. Click "Retry webhook"
4. Check if it succeeds

### Real Transaction Testing

**Small amount tests:**
```bash
# Create small charge to test live mode
stripe charges create \
  --amount 50 \  # $0.50
  --currency usd \
  --source tok_visa \
  --description "Live mode test"

# Refund immediately
stripe refunds create --charge ch_abc123
```

**Create test subscription:**
```bash
# Create customer
stripe customers create \
  --email test@example.com \
  --payment_method pm_card_visa \
  --invoice_settings default_payment_method=pm_card_visa

# Create subscription (with trial to avoid charge)
stripe subscriptions create \
  --customer cus_abc123 \
  --items[][price]=price_123 \
  --trial_period_days 7

# Cancel immediately after testing
stripe subscriptions delete sub_abc123
```

## Testing Patterns

### Test Idempotency

**Verify duplicate event handling:**
```bash
# Send same event twice
EVENT_ID=$(stripe trigger payment_intent.succeeded | grep "evt_" | cut -d" " -f2)
stripe events resend $EVENT_ID

# Your handler should:
# 1. Process once
# 2. Log "already processed" on second attempt
```

**Implementation:**
```typescript
const processedEvents = new Set<string>();

async function handleWebhook(event: Stripe.Event) {
  // Check if already processed
  if (processedEvents.has(event.id)) {
    console.log('Event already processed:', event.id);
    return { received: true };
  }
  
  // Process event
  await processEvent(event);
  
  // Mark as processed
  processedEvents.add(event.id);
  // In production: store in database
}
```

### Test Race Conditions

**Simulate webhook arriving before API call completes:**
```typescript
// Test scenario:
// 1. User clicks "subscribe"
// 2. Your API calls Stripe to create subscription
// 3. Webhook arrives before API call returns
// 4. Your webhook handler tries to find subscription in DB (not there yet!)

async function testRaceCondition() {
  // In one terminal: trigger webhook
  stripe trigger customer.subscription.created
  
  // In another: create subscription
  const subscription = await stripe.subscriptions.create({...});
  
  // Webhook handler should handle missing DB record gracefully
}
```

**Solution:**
```typescript
async function handleSubscriptionCreated(sub: Stripe.Subscription) {
  // Try to find existing record
  let dbSub = await db.subscription.findUnique({
    where: { stripeSubscriptionId: sub.id }
  });
  
  if (!dbSub) {
    // Create if doesn't exist (race condition)
    dbSub = await db.subscription.create({
      data: {
        stripeSubscriptionId: sub.id,
        // ...
      }
    });
  }
  
  // Update with latest data
  await db.subscription.update({
    where: { id: dbSub.id },
    data: { status: sub.status }
  });
}
```

### Test Event Ordering

**Events may arrive out of order:**
```bash
# Trigger in sequence
stripe trigger customer.subscription.created
stripe trigger invoice.paid
stripe trigger customer.subscription.updated

# Webhooks might arrive in different order!
```

**Solution:**
```typescript
async function handleEvent(event: Stripe.Event) {
  // Always fetch current state from Stripe if critical
  if (event.type === 'customer.subscription.updated') {
    const subscription = await stripe.subscriptions.retrieve(
      event.data.object.id
    );
    
    // Use fetched data, not event data
    await updateDatabase(subscription);
  }
}
```

### Load Testing

**Simulate webhook volume:**
```bash
# Trigger many events rapidly
for i in {1..100}; do
  stripe trigger payment_intent.succeeded &
done
wait

# Monitor:
# - Response times
# - Error rates
# - Queue backlog
# - Database performance
```

### Error Recovery Testing

**Test handler failures:**
```typescript
async function handleWebhook(event: Stripe.Event) {
  try {
    await processEvent(event);
    return { received: true };  // 200 OK
  } catch (error) {
    console.error('Handler failed:', error);
    
    // Return 500 so Stripe retries
    throw error;
  }
}
```

**Test Stripe retry behavior:**
```bash
# Return 500 to trigger retry
# Stripe will retry with exponential backoff:
# - Immediately
# - 5 minutes
# - 30 minutes
# - 2 hours
# - 6 hours
# - 12 hours
# - 24 hours (stops here)
```

## Automated Testing

### Unit Tests

**Mock Stripe webhook signature verification:**
```typescript
import { stripe } from './stripe';

// Mock verification
jest.mock('stripe', () => ({
  webhooks: {
    constructEvent: jest.fn()
  }
}));

describe('Webhook Handler', () => {
  it('should handle payment_intent.succeeded', async () => {
    const mockEvent: Stripe.Event = {
      id: 'evt_test',
      type: 'payment_intent.succeeded',
      data: {
        object: {
          id: 'pi_test',
          amount: 2000,
          metadata: { order_id: '123' }
        }
      }
    };
    
    (stripe.webhooks.constructEvent as jest.Mock)
      .mockReturnValue(mockEvent);
    
    const response = await handleWebhook(mockRequest, mockResponse);
    
    expect(response.status).toBe(200);
    // Verify order was updated
    const order = await db.order.findUnique({ where: { id: '123' }});
    expect(order.status).toBe('paid');
  });
});
```

### Integration Tests

**Test with real Stripe CLI:**
```typescript
import { exec } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

describe('Webhook Integration', () => {
  it('should process real webhook from Stripe CLI', async () => {
    // Start webhook listener
    const listener = exec('stripe listen --forward-to localhost:3000/webhook');
    
    // Wait for ready
    await new Promise(resolve => setTimeout(resolve, 2000));
    
    // Trigger event
    await execAsync('stripe trigger payment_intent.succeeded');
    
    // Wait for processing
    await new Promise(resolve => setTimeout(resolve, 1000));
    
    // Verify result
    const logs = await getApplicationLogs();
    expect(logs).toContain('âœ… Webhook processed');
    
    // Cleanup
    listener.kill();
  });
});
```

## Monitoring and Debugging

### Log Webhook Activity

```typescript
app.post('/webhook', async (req, res) => {
  const eventId = req.body.id;
  const eventType = req.body.type;
  
  console.log(`[${new Date().toISOString()}] Webhook received:`, {
    eventId,
    eventType,
    signature: req.headers['stripe-signature']?.substring(0, 20) + '...'
  });
  
  try {
    await processWebhook(req, res);
    console.log(`[${eventId}] âœ… Processed successfully`);
  } catch (error) {
    console.error(`[${eventId}] âŒ Failed:`, error);
    throw error;
  }
});
```

### Dashboard Monitoring

**Check webhook health:**
1. Go to https://dashboard.stripe.com/webhooks
2. Click on endpoint
3. Check "Recent deliveries"
4. Look for:
   - Success rate (should be >95%)
   - Response times (<5s)
   - Failed deliveries

**Investigate failures:**
1. Click on failed delivery
2. Check HTTP status code returned
3. View response body
4. Check request payload
5. Retry if fixed

## Best Practices

### Always Test Locally First

```bash
# Development workflow:
# 1. Start local server
npm run dev

# 2. Start webhook forwarding
stripe listen --forward-to localhost:3000/webhook

# 3. Trigger events
stripe trigger payment_intent.succeeded

# 4. Check logs
# 5. Fix issues
# 6. Repeat

# Only deploy to production after local testing!
```

### Test All Event Types

```bash
# Create test script
#!/bin/bash

echo "Testing all webhook events..."

stripe trigger payment_intent.succeeded
stripe trigger payment_intent.payment_failed
stripe trigger customer.subscription.created
stripe trigger customer.subscription.updated
stripe trigger customer.subscription.deleted
stripe trigger invoice.paid
stripe trigger invoice.payment_failed
stripe trigger charge.refunded

echo "All events triggered!"
```

### Validate Production Setup

```bash
# Pre-deployment checklist
# □ Webhook endpoint is HTTPS
# □ Endpoint is publicly accessible
# □ Correct webhook secret in environment
# □ Signature verification implemented
# □ Handler returns 200 within 5s
# □ Idempotency check implemented
# □ Error handling implemented
# □ Logging configured
# □ Tested with Stripe CLI
# □ Tested in Dashboard
```

## Resources

- Stripe CLI docs: https://stripe.com/docs/cli
- Testing guide: https://stripe.com/docs/testing
- Test cards: https://stripe.com/docs/testing#cards
- Webhooks best practices: https://stripe.com/docs/webhooks/best-practices
