# Event Processing Patterns and Anti-Patterns

Guide to processing Stripe webhook events correctly and avoiding common mistakes.

## Core Processing Principles

### 1. Return 200 OK Immediately

**✅ Correct Pattern:**
```typescript
app.post('/webhook', async (req, res) => {
  // 1. Verify signature
  const event = stripe.webhooks.constructEvent(body, sig, secret);
  
  // 2. Return 200 immediately
  res.json({ received: true });
  
  // 3. Process asynchronously
  processEventAsync(event).catch(console.error);
});
```

**❌ Anti-Pattern:**
```typescript
app.post('/webhook', async (req, res) => {
  const event = stripe.webhooks.constructEvent(body, sig, secret);
  
  // Slow operations before responding
  await updateDatabase(event);  // 2 seconds
  await sendEmail(event);  // 3 seconds
  await syncToAnalytics(event);  // 2 seconds
  
  res.json({ received: true });  // Took 7 seconds!
});
```

**Why it matters:**
- Stripe times out after 30 seconds
- Slow responses → retries → duplicate processing
- Your server can handle more concurrent webhooks

### 2. Implement Idempotency

**✅ Correct Pattern:**
```typescript
// Database-backed idempotency
async function processEvent(event: Stripe.Event) {
  // Check if already processed
  const existing = await db.processedWebhook.findUnique({
    where: { eventId: event.id }
  });
  
  if (existing) {
    console.log('Event already processed:', event.id);
    return;
  }
  
  // Process event
  await handleEventType(event);
  
  // Mark as processed
  await db.processedWebhook.create({
    data: {
      eventId: event.id,
      type: event.type,
      processedAt: new Date()
    }
  });
}
```

**❌ Anti-Pattern:**
```typescript
// No idempotency check
async function processEvent(event: Stripe.Event) {
  // Always processes, even if duplicate
  await db.order.update({
    where: { stripePaymentId: event.data.object.id },
    data: { 
      status: 'paid',
      paidCount: { increment: 1 }  // Increments every time!
    }
  });
}
```

**Why it matters:**
- Stripe retries on network errors
- Same event can arrive multiple times
- Without idempotency → duplicate charges, double emails, wrong state

### 3. Use Database Transactions

**✅ Correct Pattern:**
```typescript
async function handleInvoicePaid(invoice: Stripe.Invoice) {
  await db.$transaction(async (tx) => {
    // All-or-nothing operations
    await tx.subscription.update({
      where: { stripeId: invoice.subscription },
      data: { 
        status: 'active',
        currentPeriodEnd: new Date(invoice.period_end * 1000)
      }
    });
    
    await tx.user.update({
      where: { stripeCustomerId: invoice.customer },
      data: { hasAccess: true }
    });
    
    await tx.invoice.create({
      data: {
        stripeId: invoice.id,
        amount: invoice.amount_paid,
        paidAt: new Date()
      }
    });
  });
}
```

**❌ Anti-Pattern:**
```typescript
async function handleInvoicePaid(invoice: Stripe.Invoice) {
  // No transaction - can fail halfway
  await db.subscription.update({...});  // ✅ Succeeds
  
  throw new Error('Database error');  // ❌ Fails here
  
  await db.user.update({...});  // Never runs
  await db.invoice.create({...});  // Never runs
  
  // Result: Subscription updated but user doesn't have access!
}
```

**Why it matters:**
- Ensures atomic updates
- Prevents inconsistent state
- All updates succeed or none do

### 4. Handle Missing Related Objects

**✅ Correct Pattern:**
```typescript
async function handleSubscriptionUpdated(sub: Stripe.Subscription) {
  // Gracefully handle race condition
  let dbSub = await db.subscription.findUnique({
    where: { stripeId: sub.id }
  });
  
  if (!dbSub) {
    // Webhook arrived before our API call completed
    console.log('Creating subscription record (race condition)');
    dbSub = await db.subscription.create({
      data: {
        stripeId: sub.id,
        userId: sub.metadata.user_id,
        status: sub.status
      }
    });
  }
  
  // Now safe to update
  await db.subscription.update({
    where: { id: dbSub.id },
    data: { status: sub.status }
  });
}
```

**❌ Anti-Pattern:**
```typescript
async function handleSubscriptionUpdated(sub: Stripe.Subscription) {
  // Assumes record exists
  await db.subscription.update({
    where: { stripeId: sub.id },
    data: { status: sub.status }
  });
  // Throws error if record doesn't exist yet
}
```

**Why it matters:**
- Webhooks can arrive before API calls complete
- Network delays cause race conditions
- Handler should be resilient to timing

### 5. Verify Event Freshness

**✅ Correct Pattern:**
```typescript
async function processEvent(event: Stripe.Event) {
  const eventAge = Date.now() - (event.created * 1000);
  const maxAge = 5 * 60 * 1000;  // 5 minutes
  
  if (eventAge > maxAge) {
    // Event is stale, fetch current state
    const current = await stripe.subscriptions.retrieve(
      event.data.object.id
    );
    
    // Use current state instead
    await updateDatabase(current);
  } else {
    // Event is fresh, use it
    await updateDatabase(event.data.object);
  }
}
```

**❌ Anti-Pattern:**
```typescript
async function processEvent(event: Stripe.Event) {
  // Blindly trust event data, even if hours old
  await updateDatabase(event.data.object);
}
```

**Why it matters:**
- Events can be retried hours later
- Stale data can overwrite newer data
- Always verify critical state changes

## Event-Specific Patterns

### Subscription Events

**✅ Handle subscription.updated intelligently:**
```typescript
async function handleSubscriptionUpdated(sub: Stripe.Subscription) {
  // Check what actually changed
  const { previous_attributes } = event;
  
  if (!previous_attributes) {
    console.log('No changes detected');
    return;
  }
  
  // Status changed
  if (previous_attributes.status) {
    const oldStatus = previous_attributes.status;
    const newStatus = sub.status;
    
    console.log(`Status: ${oldStatus} → ${newStatus}`);
    
    if (newStatus === 'past_due') {
      await handlePaymentIssue(sub);
    } else if (newStatus === 'canceled') {
      await revokeAccess(sub);
    } else if (newStatus === 'active' && oldStatus === 'past_due') {
      await restoreAccess(sub);
    }
  }
  
  // Plan changed
  if (previous_attributes.items) {
    console.log('Plan changed');
    await updateFeatureAccess(sub);
  }
  
  // Always sync latest state
  await syncToDatabase(sub);
}
```

**❌ Process every update identically:**
```typescript
async function handleSubscriptionUpdated(sub: Stripe.Subscription) {
  // Runs expensive operations even for trivial changes
  await revokeAccess(sub);
  await grantNewAccess(sub);
  await sendEmail(sub);
  await recalculateEverything(sub);
}
```

### Payment Events

**✅ Choose the right event:**
```typescript
// For one-time payments: use payment_intent.succeeded
async function handlePaymentIntentSucceeded(pi: Stripe.PaymentIntent) {
  if (pi.metadata.type === 'order') {
    await fulfillOrder(pi.metadata.order_id);
  }
}

// For subscriptions: use invoice.paid (not payment_intent.succeeded)
async function handleInvoicePaid(invoice: Stripe.Invoice) {
  if (invoice.subscription) {
    await extendSubscription(invoice.subscription);
  }
}
```

**❌ Handle both for subscriptions:**
```typescript
// Handles payment_intent.succeeded for subscription
async function handlePaymentIntentSucceeded(pi: Stripe.PaymentIntent) {
  await extendSubscription();  // Runs!
}

// Also handles invoice.paid for same subscription
async function handleInvoicePaid(invoice: Stripe.Invoice) {
  await extendSubscription();  // Runs again! Duplicate!
}
```

### Failed Payment Events

**✅ Implement retry strategy:**
```typescript
async function handlePaymentFailed(pi: Stripe.PaymentIntent) {
  const error = pi.last_payment_error;
  const attemptCount = pi.metadata.retry_count || 0;
  
  // Soft decline - retry
  if (error.decline_code === 'insufficient_funds') {
    if (attemptCount < 3) {
      // Schedule retry
      await scheduleRetry(pi.id, '2 days');
      await sendReminderEmail('gentle');
    } else {
      // Give up after 3 tries
      await sendReminderEmail('final');
      await restrictAccess(pi.customer);
    }
  }
  
  // Hard decline - don't retry
  else if (['lost_card', 'stolen_card', 'expired_card'].includes(error.decline_code)) {
    await sendUpdateCardEmail(pi.customer);
    await restrictAccess(pi.customer);
  }
}
```

**❌ Treat all failures the same:**
```typescript
async function handlePaymentFailed(pi: Stripe.PaymentIntent) {
  // Immediately cancels for any failure
  await cancelSubscription(pi.customer);
  await sendCancellationEmail();
}
```

## Async Processing Patterns

### Using Background Jobs

**✅ Queue-based processing:**
```typescript
// Webhook handler
app.post('/webhook', async (req, res) => {
  const event = stripe.webhooks.constructEvent(body, sig, secret);
  
  // Return immediately
  res.json({ received: true });
  
  // Queue for processing
  await queue.add('stripe-webhook', {
    eventId: event.id,
    eventType: event.type,
    data: event.data
  });
});

// Worker process
queue.process('stripe-webhook', async (job) => {
  const { eventId, eventType, data } = job.data;
  
  // Check idempotency
  const processed = await db.processedEvent.findUnique({
    where: { eventId }
  });
  
  if (processed) {
    console.log('Already processed');
    return;
  }
  
  // Process event
  await handleEventType(eventType, data);
  
  // Mark complete
  await db.processedEvent.create({
    data: { eventId, processedAt: new Date() }
  });
});
```

**Benefits:**
- Fast webhook response
- Retry logic built-in
- Scalable processing
- Monitor queue health

### Using Event Sourcing

**✅ Store events, process later:**
```typescript
// Webhook handler - just store
app.post('/webhook', async (req, res) => {
  const event = stripe.webhooks.constructEvent(body, sig, secret);
  
  // Store raw event
  await db.stripeEvent.create({
    data: {
      eventId: event.id,
      type: event.type,
      data: event.data,
      receivedAt: new Date(),
      processed: false
    }
  });
  
  res.json({ received: true });
});

// Separate processor
async function processStoredEvents() {
  const events = await db.stripeEvent.findMany({
    where: { processed: false },
    orderBy: { receivedAt: 'asc' },
    take: 100
  });
  
  for (const event of events) {
    await processEvent(event);
    await db.stripeEvent.update({
      where: { id: event.id },
      data: { processed: true }
    });
  }
}
```

**Benefits:**
- Never lose events
- Replay capability
- Audit trail
- Debugging history

## Error Handling Patterns

### Retry Logic

**✅ Exponential backoff:**
```typescript
async function processWithRetry(event: Stripe.Event, maxRetries = 3) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      await processEvent(event);
      return;  // Success
    } catch (error) {
      console.error(`Attempt ${attempt + 1} failed:`, error);
      
      if (attempt === maxRetries - 1) {
        // Final attempt failed
        await logFailure(event, error);
        throw error;
      }
      
      // Wait before retry (exponential backoff)
      const delay = Math.pow(2, attempt) * 1000;  // 1s, 2s, 4s
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

### Dead Letter Queue

**✅ Handle persistent failures:**
```typescript
async function processEvent(event: Stripe.Event) {
  try {
    await handleEventType(event);
  } catch (error) {
    console.error('Processing failed:', error);
    
    // Store in dead letter queue for manual review
    await db.failedWebhook.create({
      data: {
        eventId: event.id,
        eventType: event.type,
        error: error.message,
        payload: event.data,
        failedAt: new Date()
      }
    });
    
    // Alert team
    await sendAlert({
      severity: 'high',
      message: `Webhook processing failed: ${event.type}`,
      eventId: event.id
    });
  }
}
```

### Partial Failure Handling

**✅ Handle partial failures gracefully:**
```typescript
async function handleInvoicePaid(invoice: Stripe.Invoice) {
  const results = {
    subscription: false,
    email: false,
    analytics: false
  };
  
  // Try each operation independently
  try {
    await updateSubscription(invoice);
    results.subscription = true;
  } catch (error) {
    console.error('Subscription update failed:', error);
  }
  
  try {
    await sendReceiptEmail(invoice);
    results.email = true;
  } catch (error) {
    console.error('Email sending failed:', error);
  }
  
  try {
    await trackInAnalytics(invoice);
    results.analytics = true;
  } catch (error) {
    console.error('Analytics tracking failed:', error);
  }
  
  // Log partial success
  console.log('Processing results:', results);
  
  // Critical operations must succeed
  if (!results.subscription) {
    throw new Error('Critical operation failed');
  }
}
```

## Testing Patterns

### Test Idempotency

```typescript
describe('Webhook Idempotency', () => {
  it('should handle duplicate events', async () => {
    const event = createTestEvent('payment_intent.succeeded');
    
    // Process first time
    await processEvent(event);
    const order1 = await db.order.findUnique({ where: { id: '123' }});
    expect(order1.status).toBe('paid');
    
    // Process second time (duplicate)
    await processEvent(event);
    const order2 = await db.order.findUnique({ where: { id: '123' }});
    expect(order2.status).toBe('paid');  // Still paid, not double-paid
    expect(order2.updatedAt).toEqual(order1.updatedAt);  // Not updated
  });
});
```

### Test Race Conditions

```typescript
describe('Race Conditions', () => {
  it('should handle webhook before API call', async () => {
    // Webhook arrives first
    const webhookEvent = createTestEvent('subscription.created', {
      id: 'sub_123'
    });
    
    // Process webhook (subscription not in DB yet)
    await processEvent(webhookEvent);
    
    // Should create record
    const subscription = await db.subscription.findUnique({
      where: { stripeId: 'sub_123' }
    });
    expect(subscription).toBeTruthy();
  });
});
```

## Performance Patterns

### Batch Processing

**✅ Process related events together:**
```typescript
async function processBatch(events: Stripe.Event[]) {
  // Group by customer
  const byCustomer = groupBy(events, e => e.data.object.customer);
  
  // Process per customer
  for (const [customerId, customerEvents] of Object.entries(byCustomer)) {
    await db.$transaction(async (tx) => {
      for (const event of customerEvents) {
        await processEventInTransaction(event, tx);
      }
    });
  }
}
```

### Caching

**✅ Cache frequently accessed data:**
```typescript
const customerCache = new Map();

async function getCustomer(customerId: string) {
  // Check cache
  if (customerCache.has(customerId)) {
    return customerCache.get(customerId);
  }
  
  // Fetch from Stripe
  const customer = await stripe.customers.retrieve(customerId);
  
  // Cache for 5 minutes
  customerCache.set(customerId, customer);
  setTimeout(() => customerCache.delete(customerId), 5 * 60 * 1000);
  
  return customer;
}
```

## Monitoring Patterns

### Metrics Collection

```typescript
const metrics = {
  received: 0,
  processed: 0,
  failed: 0,
  processingTime: [] as number[]
};

async function processEvent(event: Stripe.Event) {
  metrics.received++;
  const start = Date.now();
  
  try {
    await handleEventType(event);
    metrics.processed++;
  } catch (error) {
    metrics.failed++;
    throw error;
  } finally {
    const duration = Date.now() - start;
    metrics.processingTime.push(duration);
  }
}

// Report metrics
setInterval(() => {
  console.log('Webhook metrics:', {
    received: metrics.received,
    processed: metrics.processed,
    failed: metrics.failed,
    successRate: metrics.processed / metrics.received,
    avgProcessingTime: average(metrics.processingTime)
  });
}, 60 * 1000);  // Every minute
```

### Health Checks

```typescript
app.get('/health/webhooks', async (req, res) => {
  const recentEvents = await db.processedEvent.count({
    where: {
      processedAt: {
        gte: new Date(Date.now() - 5 * 60 * 1000)  // Last 5 minutes
      }
    }
  });
  
  const failedEvents = await db.failedWebhook.count({
    where: {
      failedAt: {
        gte: new Date(Date.now() - 60 * 60 * 1000)  // Last hour
      }
    }
  });
  
  const healthy = recentEvents > 0 && failedEvents === 0;
  
  res.status(healthy ? 200 : 503).json({
    status: healthy ? 'healthy' : 'unhealthy',
    recentEvents,
    failedEvents
  });
});
```

## Key Takeaways

**Always:**
- ✅ Return 200 immediately
- ✅ Implement idempotency
- ✅ Use database transactions
- ✅ Handle race conditions
- ✅ Process asynchronously

**Never:**
- ❌ Block webhook response
- ❌ Process without idempotency
- ❌ Assume data exists
- ❌ Trust stale event data
- ❌ Ignore error handling
