# Stripe Error Codes Reference

Complete guide to Stripe error codes, decline reasons, and failure messages.

## Payment Decline Codes

### Card Decline Codes

| Code | Meaning | Customer Action | Merchant Action | Retry Strategy |
|------|---------|----------------|-----------------|----------------|
| `card_declined` | Generic decline | Try different card | Accept alternative payment | Maybe (low success rate) |
| `insufficient_funds` | Not enough money | Add funds, try again | Retry in 1-2 days | Yes (high success rate) |
| `expired_card` | Card expired | Update expiry date | Prompt for update | No |
| `incorrect_cvc` | Wrong security code | Re-enter CVC correctly | Show CVC location | Yes (immediate) |
| `incorrect_number` | Invalid card number | Check card number | Validate format | Yes (immediate) |
| `incorrect_zip` | Wrong postal code | Re-enter ZIP | Validate format | Yes (immediate) |
| `invalid_expiry_month` | Bad expiry month | Enter valid month | Validate month (1-12) | Yes (immediate) |
| `invalid_expiry_year` | Bad expiry year | Enter valid year | Validate year format | Yes (immediate) |
| `invalid_number` | Invalid card number | Check card number | Luhn validation | No |
| `lost_card` | Card reported lost | Use different card | None | No |
| `stolen_card` | Card reported stolen | Use different card | None | No |
| `processing_error` | Bank/network issue | Retry immediately | Retry with backoff | Yes (immediate) |
| `card_not_supported` | Card type not accepted | Use different card type | Enable card type | No |
| `card_velocity_exceeded` | Too many charges | Wait and retry | Reduce charge frequency | Yes (after delay) |
| `do_not_honor` | Generic "no" from bank | Try different card | None | Maybe |
| `do_not_try_again` | Permanent decline | Use different card | None | No |
| `fraudulent` | Suspected fraud | Contact bank | Review transaction | No |
| `generic_decline` | Unknown reason | Try different card | Contact customer | Maybe |
| `invalid_account` | Invalid card account | Use different card | None | No |
| `new_account_information_available` | Updated card available | Update card info | Prompt update | No |
| `no_action_taken` | Bank did nothing | Retry | Retry | Yes |
| `not_permitted` | Transaction not allowed | Contact bank | None | No |
| `pickup_card` | Card should be taken | Use different card | None | No |
| `restricted_card` | Card has restrictions | Contact bank | None | No |
| `revocation_of_all_authorizations` | All auths revoked | Use different card | None | No |
| `revocation_of_authorization` | Auth revoked | Use different card | None | No |
| `security_violation` | Security issue | Contact bank | None | No |
| `service_not_allowed` | Service unavailable | Try different card | None | No |
| `transaction_not_allowed` | Transaction blocked | Contact bank | None | No |
| `try_again_later` | Temporary issue | Retry in few minutes | Retry with backoff | Yes |
| `withdrawal_count_limit_exceeded` | Too many attempts | Wait 24 hours | None | Yes (after 24h) |

### Soft Declines vs Hard Declines

**Soft Declines** (retry recommended):
- `insufficient_funds` - Customer may add funds
- `try_again_later` - Temporary bank issue
- `processing_error` - Network glitch
- `incorrect_cvc` - Simple typo
- `card_velocity_exceeded` - Rate limit

**Strategy:** Retry after delay, high success rate

**Hard Declines** (don't retry):
- `lost_card` - Card reported lost
- `stolen_card` - Card reported stolen
- `expired_card` - Past expiration
- `fraudulent` - Fraud detected
- `do_not_try_again` - Permanent decline

**Strategy:** Request alternative payment method

## Authentication Error Codes

### 3D Secure / SCA Codes

| Code | Meaning | Action |
|------|---------|--------|
| `authentication_required` | SCA needed | Redirect to 3DS flow |
| `card_declined:authentication_required` | SCA failed or incomplete | Re-attempt authentication |

**Common flow:**
```
1. payment_intent.requires_action event fires
2. Redirect customer to pi.next_action.redirect_to_url.url
3. Customer completes 3D Secure
4. payment_intent.succeeded OR payment_intent.payment_failed fires
```

## API Error Codes

### Request Errors

| Code | HTTP | Meaning | Fix |
|------|------|---------|-----|
| `invalid_request_error` | 400 | Bad request format | Check parameters |
| `api_key_expired` | 401 | Expired API key | Rotate key |
| `authentication_required` | 401 | Missing/invalid API key | Check key format |
| `idempotency_error` | 400 | Idempotency key reused | Generate new key |
| `invalid_grant` | 400 | OAuth grant invalid | Re-authenticate |
| `rate_limit` | 429 | Too many requests | Implement backoff |
| `resource_missing` | 404 | Resource not found | Verify ID |

### Server Errors

| Code | HTTP | Meaning | Action |
|------|------|---------|--------|
| `api_error` | 500+ | Stripe server issue | Retry with backoff |

### Webhook-Specific Errors

| Scenario | HTTP | Cause | Fix |
|----------|------|-------|-----|
| Signature verification failed | 400 | Invalid signature | Check secret, raw body |
| Timeout | 500 | Handler took >30s | Return 200 faster, process async |
| Server error | 500 | Handler crashed | Fix bug, add error handling |

## Invoice Failure Codes

### Payment Failed on Invoice

| Code | Meaning | Customer Impact | Action |
|------|---------|----------------|--------|
| `card_declined` | Card declined | Subscription → past_due | Send payment update link |
| `insufficient_funds` | Not enough money | Subscription → past_due | Retry automatically |
| `expired_card` | Card expired | Subscription → past_due | Prompt card update |
| `payment_method_not_available` | No payment method | Cannot charge | Request payment method |

**Subscription status flow on payment failure:**
```
active → past_due → [retries] → canceled
                              → active (if payment succeeds)
```

## Error Handling Patterns

### Handle Soft Declines

```typescript
async function handlePaymentFailed(pi: Stripe.PaymentIntent) {
  const error = pi.last_payment_error;
  
  if (!error) return;
  
  const softDeclines = [
    'insufficient_funds',
    'try_again_later',
    'processing_error',
    'card_velocity_exceeded'
  ];
  
  if (softDeclines.includes(error.decline_code)) {
    // Retry recommended
    await scheduleRetry(pi.id, '2 days');
    await sendEmail({
      template: 'payment-failed-soft',
      message: 'We'll automatically retry your payment in a few days'
    });
  } else {
    // Hard decline - need new card
    await sendEmail({
      template: 'payment-failed-hard',
      message: 'Please update your payment method'
    });
  }
}
```

### User-Friendly Messages

Map technical codes to friendly messages:

```typescript
const friendlyMessages = {
  'insufficient_funds': 
    "Your card doesn't have enough funds. You can try again in a few days after your next paycheck.",
  
  'expired_card': 
    "Your card has expired. Please update your card information.",
  
  'incorrect_cvc': 
    "The security code doesn't match. Please check the 3-digit code on the back of your card.",
  
  'card_declined': 
    "Your card was declined. Please try a different card or contact your bank.",
  
  'lost_card': 
    "This card was reported lost. Please use a different card.",
  
  'processing_error': 
    "There was a temporary issue processing your payment. Please try again.",
  
  'fraudulent': 
    "This transaction was flagged for security reasons. Please contact your bank or try a different card."
};

function getFriendlyMessage(declineCode: string): string {
  return friendlyMessages[declineCode] || 
    "Your payment couldn't be processed. Please try a different payment method.";
}
```

### Smart Retry Logic

```typescript
interface RetryStrategy {
  shouldRetry: boolean;
  retryAfter: string;
  maxRetries: number;
}

function getRetryStrategy(declineCode: string): RetryStrategy {
  const strategies: Record<string, RetryStrategy> = {
    'insufficient_funds': {
      shouldRetry: true,
      retryAfter: '2 days',  // After potential payday
      maxRetries: 3
    },
    
    'try_again_later': {
      shouldRetry: true,
      retryAfter: '1 hour',
      maxRetries: 5
    },
    
    'processing_error': {
      shouldRetry: true,
      retryAfter: '5 minutes',
      maxRetries: 3
    },
    
    'card_velocity_exceeded': {
      shouldRetry: true,
      retryAfter: '24 hours',
      maxRetries: 2
    },
    
    // Hard declines
    'expired_card': {
      shouldRetry: false,
      retryAfter: 'never',
      maxRetries: 0
    },
    
    'lost_card': {
      shouldRetry: false,
      retryAfter: 'never',
      maxRetries: 0
    },
    
    'stolen_card': {
      shouldRetry: false,
      retryAfter: 'never',
      maxRetries: 0
    },
    
    'do_not_try_again': {
      shouldRetry: false,
      retryAfter: 'never',
      maxRetries: 0
    }
  };
  
  return strategies[declineCode] || {
    shouldRetry: false,
    retryAfter: 'never',
    maxRetries: 0
  };
}
```

### Logging for Debugging

```typescript
async function logPaymentFailure(pi: Stripe.PaymentIntent) {
  const error = pi.last_payment_error;
  
  console.error('Payment failed:', {
    paymentIntentId: pi.id,
    amount: pi.amount,
    currency: pi.currency,
    customer: pi.customer,
    
    // Error details
    declineCode: error?.decline_code,
    failureCode: error?.code,
    failureMessage: error?.message,
    
    // Payment method
    paymentMethodType: error?.payment_method?.type,
    cardBrand: error?.payment_method?.card?.brand,
    cardLast4: error?.payment_method?.card?.last4,
    
    // Context
    metadata: pi.metadata,
    created: new Date(pi.created * 1000).toISOString()
  });
}
```

## Debugging Common Error Scenarios

### "Card Declined" with No Specific Code

**Problem:** Generic `card_declined` with no details

**Causes:**
1. Bank declined without reason
2. Fraud detection
3. Card restrictions
4. International transaction blocked

**Solution:**
- Suggest customer contact their bank
- Offer alternative payment methods
- Try smaller test amount first

### Recurring "Processing Error"

**Problem:** Same card keeps failing with `processing_error`

**Causes:**
1. Bank/network issue
2. 3D Secure not completing
3. Card network downtime

**Solution:**
- Check Stripe status page
- Wait 30 minutes and retry
- Try different payment method

### All Cards Failing

**Problem:** Every card gets declined

**Causes:**
1. Test cards in live mode
2. Your Stripe account has restrictions
3. Network-wide issue

**Solution:**
```bash
# Check if using test cards
if (card.number.startsWith('4242')) {
  console.error('Test card used in live mode!');
}

# Check Stripe status
curl https://status.stripe.com/api/v2/status.json
```

### 3D Secure Never Completing

**Problem:** `payment_intent.requires_action` never progresses

**Causes:**
1. Customer didn't complete challenge
2. Bank's 3DS page error
3. Popup blocked
4. Redirect URL wrong

**Solution:**
- Check `next_action.redirect_to_url.return_url`
- Test in incognito (popup blocker)
- Check browser console for errors
- Provide clear instructions to customer

## Monitoring and Alerts

### Key Metrics to Track

```typescript
interface PaymentMetrics {
  totalPayments: number;
  successfulPayments: number;
  failedPayments: number;
  
  // Decline reasons
  softDeclines: number;
  hardDeclines: number;
  
  // Specific codes
  insufficientFunds: number;
  expiredCards: number;
  lostStolenCards: number;
  
  // Success rate
  successRate: number;  // successful / total
}
```

### Alert Thresholds

```typescript
// Alert if success rate drops
if (metrics.successRate < 0.85) {  // Below 85%
  await sendAlert({
    severity: 'high',
    message: 'Payment success rate dropped below 85%',
    details: metrics
  });
}

// Alert if many expired cards
if (metrics.expiredCards > 10) {
  await sendAlert({
    severity: 'medium',
    message: 'Many payments failing due to expired cards',
    action: 'Send card update reminders'
  });
}
```

## Resources

- Full decline code list: https://stripe.com/docs/declines/codes
- Error handling: https://stripe.com/docs/error-handling
- Testing error scenarios: https://stripe.com/docs/testing#cards-responses
