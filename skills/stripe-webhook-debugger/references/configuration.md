# Stripe Dashboard Configuration Guide

Complete guide to setting up and monitoring webhooks in the Stripe Dashboard.

## Creating a Webhook Endpoint

### Step-by-Step Setup

**1. Navigate to Webhooks**
```
Dashboard → Developers → Webhooks
URL: https://dashboard.stripe.com/webhooks
```

**2. Add Endpoint**
- Click "Add endpoint" button
- Enter your webhook URL
  - Format: `https://yourdomain.com/path/to/webhook`
  - Must be HTTPS (not HTTP)
  - Must be publicly accessible
  - Cannot require authentication

**3. Select Events**

Choose which events to listen for:

**For SaaS/Subscriptions:**
- ✅ `customer.subscription.created`
- ✅ `customer.subscription.updated`
- ✅ `customer.subscription.deleted`
- ✅ `invoice.paid`
- ✅ `invoice.payment_failed`
- ✅ `customer.subscription.trial_will_end`

**For E-commerce:**
- ✅ `payment_intent.succeeded`
- ✅ `payment_intent.payment_failed`
- ✅ `charge.refunded`

**For All (optional but helpful):**
- ✅ `customer.created`
- ✅ `customer.updated`
- ✅ `payment_method.attached`

**Or select "Send all event types"** (not recommended for production)

**4. Complete Setup**
- Click "Add endpoint"
- Copy the webhook signing secret (starts with `whsec_...`)
- Add secret to your environment variables

### Test Mode vs Live Mode

**Stripe has two modes:**

**Test Mode:**
- For development and testing
- Uses test API keys (sk_test_...)
- Uses test webhook secrets (whsec_test_...)
- No real money involved
- Test cards only

**Live Mode:**
- For production
- Uses live API keys (sk_live_...)
- Uses live webhook secrets (whsec_live_...)
- Real transactions and charges

**Switching modes:**
- Toggle in top right: "Viewing test data" / "Viewing live data"
- Test and live webhooks are completely separate
- Need separate endpoint configurations for each mode

**Best practice:**
```
Development:
- Use test mode
- Use Stripe CLI for local testing
- Use test webhook secret

Staging:
- Use test mode
- Use test webhook endpoint
- Use test webhook secret

Production:
- Use live mode
- Use production webhook endpoint
- Use live webhook secret
```

## Dashboard Navigation

### Webhooks Page

**Location:** Dashboard → Developers → Webhooks

**Features:**
- List all webhook endpoints
- Create new endpoints
- View webhook secrets
- See delivery history
- Monitor endpoint health

### Events Page

**Location:** Dashboard → Developers → Events

**Features:**
- Search all events by type, ID, customer
- View event details
- See related API requests
- Replay events
- Export events

**Useful filters:**
```
Filter by event type: payment_intent.succeeded
Filter by customer: cus_abc123
Filter by date range: Last 7 days
Filter by status: succeeded, failed
```

### API Logs Page

**Location:** Dashboard → Developers → Logs

**Features:**
- See all API requests and responses
- Filter by endpoint, method, status
- View request/response bodies
- Correlate with webhooks

## Webhook Endpoint Management

### Viewing Endpoint Details

**Click on webhook endpoint to see:**
- URL and description
- Signing secret (click "Reveal")
- Events subscribed to
- Recent deliveries (successes and failures)
- Response times
- Retry attempts

### Recent Deliveries

**Shows for each delivery:**
- Event type
- Event ID (evt_...)
- Timestamp
- HTTP status code returned by your endpoint
- Response time
- Response body

**Green checkmark:** Successful (200 status)
**Red X:** Failed (non-200 status)

### Delivery Details

**Click on a delivery to see:**
- Complete request payload
- Request headers (including Stripe-Signature)
- Your endpoint's response status
- Your endpoint's response body
- Retry attempts (if failed)

### Retrying Failed Webhooks

**When webhook fails:**
1. Fix the issue in your code
2. Go to webhook delivery details
3. Click "Retry webhook"
4. Check if it succeeds

**Or retry via API:**
```bash
stripe events resend evt_abc123
```

### Disabling/Deleting Endpoints

**Disable temporarily:**
- Go to webhook endpoint
- Click "⋯" (three dots)
- Click "Disable endpoint"
- Webhooks will not be sent until re-enabled

**Delete permanently:**
- Go to webhook endpoint
- Click "⋯" (three dots)
- Click "Delete endpoint"
- WARNING: Cannot be recovered
- Webhook secret will be invalidated

### Updating Endpoint

**Change URL:**
- Cannot edit existing endpoint URL
- Must create new endpoint
- Get new webhook secret
- Update environment variables
- Delete old endpoint

**Change events:**
- Go to webhook endpoint
- Click "⋯" (three dots)
- Click "Update endpoint events"
- Select/deselect events
- Click "Update endpoint"

## Monitoring Webhook Health

### Success Rate

**Healthy endpoint indicators:**
- Success rate >95%
- Average response time <1s
- No failed deliveries in last 24h

**Unhealthy endpoint indicators:**
- Success rate <80%
- Response times >5s
- Multiple consecutive failures

### Common Failure Patterns

**All webhooks failing:**
- Endpoint URL wrong/unreachable
- Signature verification failing
- Server down

**Intermittent failures:**
- Timeout issues (handler too slow)
- Database connection issues
- Rate limiting

**Specific event types failing:**
- Handler missing for that event type
- Bug in event-specific logic

### Response Time Analysis

**Stripe requires 200 within 30 seconds:**
- <1s: Excellent ✅
- 1-5s: Good ✅
- 5-10s: Warning ⚠️
- >10s: Poor ❌
- >30s: Timeout (Stripe retries)

**If response times high:**
- Process webhooks asynchronously
- Return 200 immediately
- Move heavy operations to queue/background job

## Testing in Dashboard

### Send Test Webhook

**To test your endpoint:**
1. Go to webhook endpoint
2. Click "Send test webhook"
3. Select event type to test
4. Click "Send test webhook"
5. Check delivery status

**This helps verify:**
- Endpoint is reachable
- Signature verification works
- Handler processes event correctly

### Event Debugging

**To understand an event:**
1. Go to Events page
2. Click on event
3. View full JSON payload
4. Check related objects
5. See which webhooks received it

**Example use:**
```
"Why didn't my webhook fire?"
1. Find the event in Events page
2. Check if event was created
3. Look at "Webhook deliveries" section
4. See if your endpoint received it
5. Check delivery status
```

## Webhook Secret Management

### Finding Webhook Secret

**Location:**
1. Dashboard → Developers → Webhooks
2. Click on your webhook endpoint
3. Click "Reveal" next to "Signing secret"
4. Copy the secret (starts with `whsec_...`)

**Format:**
```
Test mode: whsec_test_abc123...
Live mode: whsec_abc123...
```

### Rolling Webhook Secrets

**To rotate secret:**
1. Add new webhook endpoint (same URL)
2. Get new secret
3. Deploy code with new secret
4. Verify new endpoint receives webhooks
5. Delete old endpoint

**Or use multiple secrets:**
```typescript
const secrets = [
  process.env.STRIPE_WEBHOOK_SECRET_OLD,
  process.env.STRIPE_WEBHOOK_SECRET_NEW
];

// Try each secret
for (const secret of secrets) {
  try {
    const event = stripe.webhooks.constructEvent(body, sig, secret);
    return event;
  } catch (err) {
    continue;
  }
}

throw new Error('Invalid signature');
```

## API Version Management

### Webhook API Version

Each webhook endpoint has an API version:
- Determines event structure
- Different versions = different fields
- Important for compatibility

**Check version:**
1. Go to webhook endpoint
2. Look for "API version"
3. Shows version like "2023-10-16"

**Event includes version:**
```json
{
  "id": "evt_123",
  "api_version": "2023-10-16",
  "type": "payment_intent.succeeded"
}
```

**Handle version differences:**
```typescript
async function handleEvent(event: Stripe.Event) {
  if (event.api_version !== '2023-10-16') {
    console.warn('Event API version mismatch');
    // Handle differently based on version
  }
}
```

### Upgrading API Version

**When Stripe releases new version:**
1. Test in test mode first
2. Create new webhook endpoint with new version
3. Deploy code that handles new structure
4. Verify working correctly
5. Switch production to new version
6. Delete old endpoint

## Troubleshooting Common Issues

### Webhook Not Receiving Events

**Check:**
1. ✅ Endpoint URL is correct
2. ✅ Endpoint is publicly accessible (not localhost)
3. ✅ Using HTTPS (not HTTP)
4. ✅ Events are selected in endpoint configuration
5. ✅ Endpoint is enabled (not disabled)
6. ✅ Server is running and reachable

**Test:**
```bash
# Check if endpoint is reachable
curl -X POST https://yourdomain.com/webhook \
  -H "Content-Type: application/json" \
  -d '{"test": true}'

# Should return response (not 404 or timeout)
```

### Signature Verification Always Failing

**Check:**
1. ✅ Using correct webhook secret (not API key)
2. ✅ Test secret with test webhooks
3. ✅ Live secret with live webhooks
4. ✅ Using raw request body
5. ✅ Stripe-Signature header present

**Verify in Dashboard:**
- Look at failed webhook delivery
- Check response body for error message
- Should say "signature verification failed"

### Webhooks Timing Out

**Symptoms:**
- Status 500 in Dashboard
- Response time >30s
- Stripe retrying constantly

**Solutions:**
```typescript
// ❌ BAD - synchronous processing
app.post('/webhook', async (req, res) => {
  await heavyDatabaseOperation();  // Slow!
  await sendEmails();  // Slow!
  res.json({ received: true });
});

// ✅ GOOD - immediate response
app.post('/webhook', async (req, res) => {
  // Verify signature first
  const event = stripe.webhooks.constructEvent(...);
  
  // Respond immediately
  res.json({ received: true });
  
  // Process asynchronously
  processEventAsync(event).catch(console.error);
});
```

### Events Not in Chronological Order

**Symptom:**
- `subscription.updated` arrives before `subscription.created`
- Events seem out of sequence

**Solution:**
```typescript
// Always fetch current state from Stripe
async function handleSubscriptionEvent(subscription: Stripe.Subscription) {
  // Fetch latest state
  const latest = await stripe.subscriptions.retrieve(subscription.id);
  
  // Use latest data, not event data
  await updateDatabase(latest);
}
```

## Environment-Specific Configuration

### Development Environment

**Setup:**
```bash
# Use Stripe CLI
stripe listen --forward-to localhost:3000/webhook

# Or create test mode endpoint
URL: https://dev.yourdomain.com/webhook
Mode: Test
Secret: whsec_test_...
```

**Environment variables:**
```bash
STRIPE_API_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_test_...  # From Stripe CLI or Dashboard
```

### Staging Environment

**Setup:**
```bash
# Create test mode endpoint
URL: https://staging.yourdomain.com/webhook
Mode: Test
Secret: whsec_test_...
```

**Environment variables:**
```bash
STRIPE_API_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_test_...  # Different from dev!
```

### Production Environment

**Setup:**
```bash
# Create live mode endpoint
URL: https://yourdomain.com/webhook
Mode: Live
Secret: whsec_...
```

**Environment variables:**
```bash
STRIPE_API_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...  # Live mode secret
```

## Security Best Practices

### Endpoint Security

**✅ Do:**
- Use HTTPS only
- Verify webhook signatures
- Use environment variables for secrets
- Return 200 only after verification succeeds
- Log all webhook activity
- Rate limit webhook endpoint (prevent abuse)

**❌ Don't:**
- Use HTTP endpoints
- Skip signature verification
- Hard-code secrets
- Expose secrets in logs
- Require authentication on webhook endpoint

### Access Control

**Webhook endpoints should:**
- Be publicly accessible (no auth)
- Verify via signature (not auth tokens)
- Use IP allowlisting cautiously (Stripe IPs change)
- Return minimal error information

### Monitoring

**Set up alerts for:**
- Webhook success rate drops below 95%
- Response times exceed 5s
- Multiple consecutive failures
- Signature verification failures

## Resources

- Webhook settings: https://dashboard.stripe.com/webhooks
- Events: https://dashboard.stripe.com/events
- API logs: https://dashboard.stripe.com/logs
- Webhook docs: https://stripe.com/docs/webhooks
