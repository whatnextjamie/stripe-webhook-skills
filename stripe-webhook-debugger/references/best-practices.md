# Stripe Webhook Best Practices

Comprehensive guide to webhook architecture, security, and operational excellence.

## Security Best Practices

### Always Verify Signatures

**Critical: Never skip signature verification**

```typescript
// ✅ REQUIRED - Verify every webhook
app.post('/webhook', async (req, res) => {
  const sig = req.headers['stripe-signature'];
  
  if (!sig) {
    return res.status(400).send('No signature');
  }
  
  try {
    const event = stripe.webhooks.constructEvent(
      req.body,  // Raw body
      sig,
      process.env.STRIPE_WEBHOOK_SECRET!
    );
    
    // Only process if verification succeeds
    await processEvent(event);
    res.json({ received: true });
  } catch (err) {
    return res.status(400).send('Invalid signature');
  }
});
```

**Why this matters:**
- Prevents malicious webhooks
- Ensures authenticity
- Protects against replay attacks
- Required for PCI compliance

### Protect Webhook Secrets

**✅ Do:**
```typescript
// Store in environment variables
const secret = process.env.STRIPE_WEBHOOK_SECRET;

// Validate at startup
if (!secret || !secret.startsWith('whsec_')) {
  throw new Error('Invalid webhook secret');
}

// Use different secrets per environment
const secret = process.env.NODE_ENV === 'production'
  ? process.env.STRIPE_WEBHOOK_SECRET_LIVE
  : process.env.STRIPE_WEBHOOK_SECRET_TEST;
```

**❌ Don't:**
```typescript
// Hard-coded secrets
const secret = 'whsec_abc123...';  // NEVER!

// Committed to git
// .env file in repository  // NEVER!

// Logged in plain text
console.log('Secret:', secret);  // NEVER!

// Exposed in error messages
throw new Error(`Failed with secret ${secret}`);  // NEVER!
```

### Use HTTPS Only

```
✅ https://yourdomain.com/webhook
❌ http://yourdomain.com/webhook  // Insecure!
```

**Why:**
- Prevents man-in-the-middle attacks
- Encrypts webhook payload
- Required by Stripe
- Industry standard

### No Authentication on Webhook Endpoint

**✅ Correct:**
```typescript
// Public endpoint, verified by signature
app.post('/webhook', async (req, res) => {
  const event = stripe.webhooks.constructEvent(...);
  // Signature is the authentication
});
```

**❌ Wrong:**
```typescript
// Don't require auth tokens
app.post('/webhook', 
  requireAuth,  // ❌ Stripe can't provide this
  async (req, res) => {
    // Stripe will get 401 Unauthorized
  }
);
```

### Minimal Error Information

**✅ Safe error responses:**
```typescript
try {
  const event = stripe.webhooks.constructEvent(body, sig, secret);
  await processEvent(event);
  res.json({ received: true });
} catch (err) {
  // Don't expose internal details
  console.error('Webhook error:', err);
  res.status(400).json({ error: 'Invalid request' });
}
```

**❌ Exposing secrets:**
```typescript
catch (err) {
  // Exposes secret in error message!
  res.status(400).json({
    error: err.message,
    secret: secret,  // ❌ NEVER!
    stack: err.stack  // ❌ Shows internal paths
  });
}
```

## Architecture Best Practices

### Return 200 Within 5 Seconds

**✅ Fast response pattern:**
```typescript
app.post('/webhook', async (req, res) => {
  // 1. Verify (fast)
  const event = stripe.webhooks.constructEvent(body, sig, secret);
  
  // 2. Respond immediately
  res.json({ received: true });
  
  // 3. Process asynchronously
  queueEvent(event).catch(console.error);
});
```

**Why:**
- Stripe retries if no 200 within 30s
- Fast responses prevent duplicate processing
- Better webhook reliability
- Can handle more concurrent webhooks

### Implement Idempotency

**Required: Events can arrive multiple times**

```typescript
// Database-backed idempotency
async function processEvent(event: Stripe.Event) {
  // Use database transaction
  await db.$transaction(async (tx) => {
    // Check if processed
    const existing = await tx.webhookLog.findUnique({
      where: { eventId: event.id }
    });
    
    if (existing) {
      console.log('Already processed:', event.id);
      return;
    }
    
    // Process event
    await handleEvent(event, tx);
    
    // Log as processed
    await tx.webhookLog.create({
      data: {
        eventId: event.id,
        type: event.type,
        processedAt: new Date()
      }
    });
  });
}
```

**Why:**
- Stripe retries on network errors
- Server restarts can duplicate
- Prevents duplicate charges/emails
- Data consistency

### Use Database Transactions

```typescript
async function handleInvoicePaid(invoice: Stripe.Invoice) {
  // Atomic updates
  await db.$transaction(async (tx) => {
    await tx.subscription.update({...});
    await tx.user.update({...});
    await tx.payment.create({...});
  });
  // All succeed or all fail
}
```

**Why:**
- Prevents partial updates
- Ensures data consistency
- Enables rollback on errors

### Process Asynchronously

**Use job queues for heavy operations:**

```typescript
// Webhook handler
app.post('/webhook', async (req, res) => {
  const event = stripe.webhooks.constructEvent(body, sig, secret);
  res.json({ received: true });
  
  // Queue for processing
  await queue.add('webhook', {
    eventId: event.id,
    eventType: event.type,
    payload: event.data
  });
});

// Worker process
worker.process('webhook', async (job) => {
  await processEvent(job.data);
});
```

**Benefits:**
- Fast webhook response
- Retry logic built-in
- Horizontal scaling
- Monitor queue health
- Rate limiting

## Operational Best Practices

### Log Everything

```typescript
async function processEvent(event: Stripe.Event) {
  console.log('[WEBHOOK]', {
    timestamp: new Date().toISOString(),
    eventId: event.id,
    eventType: event.type,
    livemode: event.livemode
  });
  
  try {
    await handleEvent(event);
    console.log('[SUCCESS]', event.id);
  } catch (error) {
    console.error('[FAILED]', {
      eventId: event.id,
      error: error.message,
      stack: error.stack
    });
    throw error;
  }
}
```

**What to log:**
- ✅ Event ID and type
- ✅ Timestamp
- ✅ Success/failure
- ✅ Processing time
- ✅ Error details

**What NOT to log:**
- ❌ Webhook secrets
- ❌ Full credit card numbers
- ❌ Customer PII (unless necessary)
- ❌ API keys

### Monitor Webhook Health

**Key metrics to track:**

```typescript
const metrics = {
  // Volume
  received: 0,
  processed: 0,
  failed: 0,
  
  // Performance
  avgProcessingTime: 0,
  p95ProcessingTime: 0,
  p99ProcessingTime: 0,
  
  // Health
  successRate: 0,
  errorRate: 0,
  
  // By event type
  byType: {
    'payment_intent.succeeded': 0,
    'invoice.paid': 0,
    // ...
  }
};
```

**Alert thresholds:**
- Success rate < 95%
- Avg processing time > 1s
- Any failures in last 5 minutes
- Queue depth > 1000

### Set Up Alerting

```typescript
async function processEvent(event: Stripe.Event) {
  try {
    await handleEvent(event);
  } catch (error) {
    // Alert on critical failures
    if (event.type === 'invoice.paid' || 
        event.type === 'payment_intent.succeeded') {
      await sendAlert({
        severity: 'critical',
        message: `Failed to process ${event.type}`,
        eventId: event.id,
        error: error.message
      });
    }
  }
}
```

### Implement Health Checks

```typescript
app.get('/health/webhooks', async (req, res) => {
  const checks = {
    database: false,
    queue: false,
    recentEvents: false
  };
  
  // Check database
  try {
    await db.$queryRaw`SELECT 1`;
    checks.database = true;
  } catch (error) {
    console.error('Database check failed:', error);
  }
  
  // Check queue
  try {
    const queueHealth = await queue.getHealth();
    checks.queue = queueHealth.status === 'healthy';
  } catch (error) {
    console.error('Queue check failed:', error);
  }
  
  // Check recent events
  const recentCount = await db.webhookLog.count({
    where: {
      processedAt: {
        gte: new Date(Date.now() - 5 * 60 * 1000)
      }
    }
  });
  checks.recentEvents = recentCount > 0;
  
  const healthy = Object.values(checks).every(v => v);
  
  res.status(healthy ? 200 : 503).json({
    status: healthy ? 'healthy' : 'unhealthy',
    checks
  });
});
```

### Test Regularly

**Automated testing:**
```bash
# Daily smoke test
#!/bin/bash

echo "Testing webhook endpoint..."

# Test signature verification
stripe trigger payment_intent.succeeded

# Wait for processing
sleep 5

# Check database
psql -c "SELECT * FROM webhook_logs ORDER BY created_at DESC LIMIT 1"

# Check metrics
curl http://localhost:3000/metrics/webhooks
```

**Manual testing:**
- Test in staging before production
- Use Stripe CLI for local testing
- Test with Dashboard "Send test webhook"
- Verify all event types work

## Error Handling Best Practices

### Graceful Degradation

```typescript
async function handleInvoicePaid(invoice: Stripe.Invoice) {
  const results = {
    critical: false,
    email: false,
    analytics: false
  };
  
  // Critical: Must succeed
  try {
    await updateSubscription(invoice);
    results.critical = true;
  } catch (error) {
    console.error('Critical failure:', error);
    throw error;  // Fail webhook
  }
  
  // Non-critical: Can fail
  try {
    await sendReceiptEmail(invoice);
    results.email = true;
  } catch (error) {
    console.error('Email failed (non-critical):', error);
    // Continue processing
  }
  
  try {
    await trackInAnalytics(invoice);
    results.analytics = true;
  } catch (error) {
    console.error('Analytics failed (non-critical):', error);
    // Continue processing
  }
  
  console.log('Processing results:', results);
}
```

### Retry with Backoff

```typescript
async function processWithRetry(
  fn: () => Promise<void>,
  maxRetries = 3
) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      await fn();
      return;
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      
      const delay = Math.pow(2, i) * 1000;
      await new Promise(r => setTimeout(r, delay));
    }
  }
}
```

### Dead Letter Queue

```typescript
async function processEvent(event: Stripe.Event) {
  try {
    await handleEvent(event);
  } catch (error) {
    // Store failed events
    await db.failedWebhook.create({
      data: {
        eventId: event.id,
        eventType: event.type,
        payload: event.data,
        error: error.message,
        failedAt: new Date()
      }
    });
    
    // Alert team
    await sendAlert({
      severity: 'high',
      message: 'Webhook processing failed',
      eventId: event.id
    });
    
    // Return 500 so Stripe retries
    throw error;
  }
}
```

## Data Management Best Practices

### Store Webhook History

```typescript
// Store all received webhooks
await db.webhookEvent.create({
  data: {
    eventId: event.id,
    type: event.type,
    livemode: event.livemode,
    apiVersion: event.api_version,
    payload: event.data,
    receivedAt: new Date(),
    processed: false
  }
});
```

**Benefits:**
- Audit trail
- Replay capability
- Debugging history
- Compliance requirements

### Handle API Version Changes

```typescript
async function processEvent(event: Stripe.Event) {
  const currentVersion = '2023-10-16';
  
  if (event.api_version !== currentVersion) {
    console.warn('API version mismatch:', {
      eventVersion: event.api_version,
      currentVersion
    });
    
    // Handle version differences
    if (event.api_version < currentVersion) {
      await handleLegacyEvent(event);
    } else {
      await handleNewerEvent(event);
    }
  } else {
    await handleCurrentEvent(event);
  }
}
```

### Sync State with Stripe

**Don't trust stale events:**
```typescript
async function handleCriticalUpdate(event: Stripe.Event) {
  // Always fetch current state for critical operations
  const subscription = await stripe.subscriptions.retrieve(
    event.data.object.id
  );
  
  // Use current state, not event data
  await updateDatabase(subscription);
}
```

## Performance Best Practices

### Batch Database Operations

```typescript
async function processBatch(events: Stripe.Event[]) {
  // Collect all updates
  const updates = events.map(e => ({
    where: { stripeId: e.data.object.id },
    data: { status: e.data.object.status }
  }));
  
  // Execute in transaction
  await db.$transaction(
    updates.map(u => db.subscription.update(u))
  );
}
```

### Use Connection Pooling

```typescript
// Database configuration
const db = new PrismaClient({
  datasources: {
    db: {
      url: process.env.DATABASE_URL,
    },
  },
  // Connection pool
  connection_limit: 20,
  pool_timeout: 10,
});
```

### Cache Frequently Accessed Data

```typescript
const customerCache = new LRU({
  max: 1000,
  ttl: 5 * 60 * 1000  // 5 minutes
});

async function getCustomer(id: string) {
  let customer = customerCache.get(id);
  
  if (!customer) {
    customer = await stripe.customers.retrieve(id);
    customerCache.set(id, customer);
  }
  
  return customer;
}
```

## Deployment Best Practices

### Environment Separation

```
Development → Staging → Production

Each with:
- Separate Stripe accounts (test → live)
- Separate webhook endpoints
- Separate secrets
- Separate databases
```

### Gradual Rollouts

```typescript
// Feature flag for new webhook logic
async function handlePaymentIntent(pi: Stripe.PaymentIntent) {
  if (shouldUseNewLogic(pi.customer)) {
    await newPaymentLogic(pi);
  } else {
    await oldPaymentLogic(pi);
  }
}

function shouldUseNewLogic(customerId: string) {
  // Gradually enable for % of customers
  const hash = hashCode(customerId);
  return (hash % 100) < parseInt(process.env.ROLLOUT_PERCENTAGE || '0');
}
```

### Zero-Downtime Deployments

```typescript
// Support multiple webhook versions during deployment
const handlers = {
  v1: require('./webhooks/v1'),
  v2: require('./webhooks/v2')
};

app.post('/webhook', async (req, res) => {
  const version = req.headers['x-webhook-version'] || 'v1';
  const handler = handlers[version];
  
  if (!handler) {
    return res.status(400).send('Unknown version');
  }
  
  await handler.process(req, res);
});
```

## Compliance Best Practices

### PCI Compliance

- ✅ Verify all webhook signatures
- ✅ Use HTTPS endpoints
- ✅ Don't log full credit card numbers
- ✅ Encrypt webhook secrets at rest
- ✅ Restrict access to webhook logs

### GDPR Compliance

```typescript
// Don't store unnecessary PII
async function processCustomerEvent(customer: Stripe.Customer) {
  // ✅ Store minimum necessary
  await db.customer.create({
    data: {
      stripeId: customer.id,
      // Don't store email/name unless required
    }
  });
}

// Support data deletion
async function handleCustomerDeleted(customer: Stripe.Customer) {
  // Delete all customer data
  await db.customer.delete({
    where: { stripeId: customer.id }
  });
}
```

### Audit Logging

```typescript
// Log all webhook processing for audit
await db.auditLog.create({
  data: {
    action: 'webhook_processed',
    eventType: event.type,
    eventId: event.id,
    userId: event.data.object.customer,
    timestamp: new Date(),
    metadata: {
      livemode: event.livemode,
      success: true
    }
  }
});
```

## Documentation Best Practices

### Document Event Handling

```typescript
/**
 * Handles successful subscription payments
 * 
 * Actions:
 * 1. Extends subscription period
 * 2. Updates user access
 * 3. Sends receipt email
 * 4. Tracks in analytics
 * 
 * Idempotency: Safe to call multiple times
 * Critical: Yes (must succeed)
 * Retry: Automatic via Stripe
 */
async function handleInvoicePaid(invoice: Stripe.Invoice) {
  // ...
}
```

### Maintain Runbook

```markdown
# Webhook Runbook

## Common Issues

### All webhooks failing
1. Check server status
2. Check database connection
3. Check Stripe Dashboard for errors
4. Review recent deployments
5. Check webhook secret is correct

### Specific event type failing
1. Check handler implementation
2. Check for recent code changes
3. Test locally with Stripe CLI
4. Review error logs

## Recovery Procedures

### Replay missed webhooks
```bash
# Get missed events
stripe events list --created-after 2024-01-01

# Retry each event
for event in $(stripe events list --limit 100); do
  stripe events resend $event
done
```
```

## Key Takeaways

**Security:**
- Always verify signatures
- Use HTTPS
- Protect secrets
- Minimal error info

**Reliability:**
- Return 200 fast
- Implement idempotency
- Process asynchronously
- Use transactions

**Operations:**
- Log everything
- Monitor metrics
- Set up alerts
- Test regularly

**Performance:**
- Batch operations
- Cache data
- Use job queues
- Connection pooling
