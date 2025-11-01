# Stripe Event Types Reference

Complete guide to Stripe webhook events with use cases and handling guidance.

## Event Flow Visualization

```
Customer Action → Stripe API → Event Created → Webhook Sent → Your Handler → 200 OK
                                                     ↓
                                              [Retries if no 200]
```

## Payment Flow Events

### payment_intent.created
**When it fires:** Customer initiates a payment

**Contains:**
- Payment amount and currency
- Customer ID (if attached)
- Payment method details
- Metadata from checkout

**Typical use:** Track payment initiated for analytics

**Example:**
```json
{
  "type": "payment_intent.created",
  "data": {
    "object": {
      "id": "pi_123",
      "amount": 2000,
      "currency": "usd",
      "status": "requires_payment_method"
    }
  }
}
```

### payment_intent.requires_action
**When it fires:** Payment needs additional customer action (usually 3D Secure)

**Contains:**
- Payment intent ID
- `next_action` object with redirect URL
- Customer challenge details

**Typical use:** Send notification that customer needs to complete authentication

**Important:** Customer must complete action at `next_action.redirect_to_url.url`

### payment_intent.succeeded
**When it fires:** Payment completed successfully ✅

**Contains:**
- Confirmed payment amount
- Charge ID
- Payment method details
- Customer email (if provided)

**Typical use:**
- Fulfill orders
- Send confirmation email
- Update database to "paid" status
- Grant access to product

**Example handler logic:**
```typescript
async function handlePaymentSucceeded(pi: Stripe.PaymentIntent) {
  // 1. Find order from metadata
  const orderId = pi.metadata.order_id;
  
  // 2. Update order status
  await db.order.update({
    where: { id: orderId },
    data: { 
      status: 'paid',
      stripePaymentIntentId: pi.id
    }
  });
  
  // 3. Send confirmation email
  await sendEmail({
    to: pi.receipt_email,
    subject: 'Order confirmed',
    template: 'order-confirmation',
    data: { orderId, amount: pi.amount / 100 }
  });
}
```

### payment_intent.payment_failed
**When it fires:** Payment attempt was declined ❌

**Contains:**
- `last_payment_error` object with details
- `failure_code` (e.g., insufficient_funds, expired_card)
- `failure_message` (user-friendly description)
- Decline code

**Typical use:**
- Send payment failure notification
- Provide retry or alternative payment option
- Track failed payment analytics
- Update order status to "payment_failed"

**Important:** These often succeed if retried (especially insufficient_funds)

**Common decline codes:**
- `insufficient_funds` → Retry in 1-2 days (soft decline)
- `expired_card` → Customer must update card
- `incorrect_cvc` → Re-enter CVC
- `card_declined` → Try different card
- `lost_card` / `stolen_card` → Must use different card

### payment_intent.canceled
**When it fires:** Payment was canceled (by you or customer)

**Contains:**
- Cancellation reason
- Last payment error (if any)

**Typical use:** Update order status to "canceled"

### payment_intent.amount_capturable_updated
**When it fires:** Amount available to capture changed (for manual capture payments)

**Typical use:** Update internal records of capturable amount

## Charge Events

### charge.succeeded
**When it fires:** A charge completed successfully

**Relationship to PaymentIntent:**
- Every successful PaymentIntent creates a Charge
- You'll get both `payment_intent.succeeded` AND `charge.succeeded`
- Usually you only need to handle `payment_intent.succeeded`

**When to use charge events:**
- Legacy integrations (before Payment Intents)
- Direct Charge API usage
- Need low-level charge details

### charge.failed
**When it fires:** Charge failed (payment method declined)

**Typical use:** Same as `payment_intent.payment_failed`

### charge.refunded
**When it fires:** A charge was refunded (partial or full)

**Contains:**
- Refund amount
- Refund reason
- Original charge details

**Typical use:**
- Update order status to "refunded"
- Send refund confirmation email
- Adjust inventory
- Update accounting records

**Example:**
```json
{
  "type": "charge.refunded",
  "data": {
    "object": {
      "id": "ch_123",
      "amount": 2000,
      "amount_refunded": 2000,  // Full refund
      "refunds": {
        "data": [{
          "amount": 2000,
          "reason": "requested_by_customer"
        }]
      }
    }
  }
}
```

## Subscription Lifecycle Events

### customer.subscription.created
**When it fires:** New subscription created

**Contains:**
- Subscription ID
- Customer ID
- Plan/price details
- Billing cycle dates
- Trial info (if applicable)
- Status (usually `active` or `trialing`)

**Typical use:**
- Create subscription record in database
- Grant user access to features
- Send welcome email
- Start trial countdown

**Example:**
```typescript
async function handleSubscriptionCreated(sub: Stripe.Subscription) {
  await db.subscription.create({
    data: {
      stripeSubscriptionId: sub.id,
      stripeCustomerId: sub.customer,
      status: sub.status,
      priceId: sub.items.data[0].price.id,
      currentPeriodEnd: new Date(sub.current_period_end * 1000),
      userId: sub.metadata.user_id
    }
  });
  
  // Grant access
  await grantAccess(sub.metadata.user_id, 'premium');
  
  // Send welcome email
  await sendWelcomeEmail(sub.customer);
}
```

### customer.subscription.updated
**When it fires:** ANY change to subscription (very common event)

**Contains:**
- All subscription fields
- Previous values in `previous_attributes` (very useful!)

**Changes that trigger this:**
- Plan/price change
- Quantity change
- Status change (active → past_due → canceled)
- Trial end
- Payment method update
- Metadata update
- Schedule update

**Typical use:** Update subscription record in database to match Stripe

**Important:** This fires A LOT. Compare `previous_attributes` to see what actually changed.

**Example with change detection:**
```typescript
async function handleSubscriptionUpdated(sub: Stripe.Subscription) {
  const { previous_attributes } = event;
  
  // Check what changed
  if (previous_attributes?.status) {
    console.log(`Status changed: ${previous_attributes.status} → ${sub.status}`);
    
    // Handle status changes
    if (sub.status === 'past_due') {
      await handlePaymentIssue(sub);
    } else if (sub.status === 'canceled') {
      await revokeAccess(sub.metadata.user_id);
    }
  }
  
  if (previous_attributes?.items) {
    console.log('Plan changed');
    await updateFeatureAccess(sub);
  }
  
  // Always sync to database
  await db.subscription.update({
    where: { stripeSubscriptionId: sub.id },
    data: {
      status: sub.status,
      currentPeriodEnd: new Date(sub.current_period_end * 1000)
    }
  });
}
```

### customer.subscription.deleted
**When it fires:** Subscription canceled or expired

**Contains:**
- Final subscription state
- Cancellation reason (if provided)
- Canceled_at timestamp

**Typical use:**
- Revoke access to features
- Send cancellation confirmation email
- Update database status to "canceled"
- Offer win-back campaign

**Note:** This fires at the END of the billing period for cancellations

### customer.subscription.trial_will_end
**When it fires:** 3 days before trial ends

**Contains:**
- Subscription details
- Trial end date

**Typical use:**
- Send reminder email
- Notify user to add payment method
- Offer trial extension

**Important:** Only fires if subscription has a trial

### customer.subscription.paused
**When it fires:** Subscription paused (new feature)

**Typical use:** Maintain partial access or send reminder emails

### customer.subscription.resumed
**When it fires:** Paused subscription resumed

**Typical use:** Restore full access

## Invoice Events

### invoice.created
**When it fires:** New invoice generated (before payment attempt)

**Typical use:** Preview upcoming charges, send advance notice

### invoice.finalized
**When it fires:** Invoice finalized and ready for payment

**Contains:**
- Total amount
- Line items
- Due date

**Typical use:** Send invoice to customer for payment

### invoice.paid
**When it fires:** Invoice payment succeeded ✅

**Contains:**
- Payment amount
- Subscription ID (if subscription invoice)
- Line items
- Period covered

**Typical use:**
- Extend subscription access
- Send receipt
- Update billing history
- Reset usage limits (if applicable)

**Key point:** For subscriptions, handle this instead of `payment_intent.succeeded` to avoid duplicate processing

**Example:**
```typescript
async function handleInvoicePaid(invoice: Stripe.Invoice) {
  if (!invoice.subscription) return;  // One-off invoice
  
  // Extend subscription period
  await db.subscription.update({
    where: { stripeSubscriptionId: invoice.subscription },
    data: {
      currentPeriodEnd: new Date(invoice.period_end * 1000),
      status: 'active'
    }
  });
  
  // Send receipt
  await sendReceipt(invoice.customer_email, invoice);
}
```

### invoice.payment_failed
**When it fires:** Invoice payment attempt failed ❌

**Contains:**
- Failed payment details
- Attempt count
- Next retry date

**Typical use:**
- Send payment failure notification
- Update subscription status to "past_due"
- Provide payment update link
- Consider access restrictions

**Important:** Stripe auto-retries failed payments. This fires on EACH failure.

**Example:**
```typescript
async function handleInvoicePaymentFailed(invoice: Stripe.Invoice) {
  const attemptCount = invoice.attempt_count;
  
  if (attemptCount === 1) {
    // First failure - gentle reminder
    await sendEmail({
      template: 'payment-failed-reminder',
      data: { updatePaymentUrl: '...' }
    });
  } else if (attemptCount >= 3) {
    // Multiple failures - restrict access
    await restrictAccess(invoice.subscription);
    await sendEmail({
      template: 'payment-failed-urgent',
      data: { updatePaymentUrl: '...' }
    });
  }
  
  // Update status
  await db.subscription.update({
    where: { stripeSubscriptionId: invoice.subscription },
    data: { status: 'past_due' }
  });
}
```

### invoice.upcoming
**When it fires:** 7 days before renewal (for subscriptions)

**Typical use:**
- Send renewal reminder
- Notify about upcoming charge
- Offer plan changes before renewal

**Important:** This is a forecast, not a real invoice yet

### invoice.payment_action_required
**When it fires:** Invoice payment needs customer action (3D Secure)

**Typical use:** Send notification with authentication link

## Customer Events

### customer.created
**When it fires:** New customer record created

**Typical use:** Sync customer to your database

### customer.updated
**When it fires:** Customer details changed (name, email, metadata, etc.)

**Typical use:** Keep customer data in sync

### customer.deleted
**When it fires:** Customer record deleted from Stripe

**Typical use:** Clean up related records in your database

### customer.source.created
**When it fires:** Payment method added to customer (legacy Cards API)

**Modern equivalent:** `payment_method.attached`

### customer.source.updated
**When it fires:** Payment method details changed

**Typical use:** Update stored payment method info

### customer.source.deleted
**When it fires:** Payment method removed from customer

**Typical use:** Update UI to show no payment method

## Payment Method Events

### payment_method.attached
**When it fires:** Payment method saved to customer

**Typical use:**
- Update UI to show payment method
- Enable subscription creation
- Send confirmation

### payment_method.detached
**When it fires:** Payment method removed from customer

**Typical use:** Update UI to reflect removal

### payment_method.updated
**When it fires:** Payment method details changed (e.g., billing address)

**Typical use:** Sync updated details

## Dispute Events

### charge.dispute.created
**When it fires:** Customer disputed a charge (chargeback)

**Contains:**
- Dispute reason
- Disputed amount
- Evidence deadline

**Typical use:**
- Send notification to merchant
- Gather evidence for dispute
- Update order record

**Important:** You typically have 7-21 days to respond with evidence

### charge.dispute.updated
**When it fires:** Dispute status changed or evidence submitted

**Typical use:** Track dispute progress

### charge.dispute.closed
**When it fires:** Dispute resolved (won or lost)

**Contains:**
- Outcome (won/lost)
- Reason for outcome

**Typical use:**
- Update accounting if lost
- Celebrate if won
- Analyze patterns

## Checkout Events

### checkout.session.completed
**When it fires:** Checkout Session completed successfully

**Contains:**
- Payment status
- Customer ID
- Subscription ID (if subscription checkout)
- Line items

**Typical use:**
- Fulfill orders
- Send confirmation
- Grant access

**Important:** For subscriptions, also listen to subscription events for ongoing management

### checkout.session.expired
**When it fires:** Checkout Session expired without completion

**Typical use:** Analytics (track abandoned checkouts)

## Event Processing Best Practices

### Handle Idempotency

```typescript
// Store processed event IDs
const processedEvents = new Set();

async function processEvent(event: Stripe.Event) {
  if (processedEvents.has(event.id)) {
    console.log('Event already processed:', event.id);
    return;
  }
  
  // Process event...
  
  processedEvents.add(event.id);
  // In production, store in database
}
```

### Check API Version

```typescript
// Events include the API version they were created with
if (event.api_version !== '2023-10-16') {
  console.warn('Event API version mismatch:', event.api_version);
  // Handle version differences
}
```

### Handle Related Events

Some actions trigger multiple events:
- Creating a subscription → `customer.subscription.created` + `invoice.paid` + `payment_intent.succeeded`
- Refunding a charge → `charge.refunded` + `payment_intent.canceled`

**Solution:** Choose one event to handle the business logic (usually the higher-level one)

### Event Timing

Events may arrive out of order or delayed:
- Always check current state via API if critical
- Don't assume event order
- Handle missing related objects gracefully

## Quick Event Selection Guide

**For one-time payments:**
→ Handle `payment_intent.succeeded`

**For subscriptions:**
→ Handle `invoice.paid` (not `payment_intent.succeeded`)

**For failed payments:**
→ Handle `invoice.payment_failed` (subscriptions) or `payment_intent.payment_failed` (one-time)

**For cancellations:**
→ Handle `customer.subscription.deleted`

**For refunds:**
→ Handle `charge.refunded`
