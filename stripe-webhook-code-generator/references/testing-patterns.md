# Testing Patterns for Stripe Webhooks

Complete test generation patterns for all major testing frameworks.

## Jest (Next.js / Express / Node)

### App Router Test

```typescript
// __tests__/app/api/webhooks/stripe/route.test.ts
import { POST } from '@/app/api/webhooks/stripe/route';
import { NextRequest } from 'next/server';
import Stripe from 'stripe';
import { prisma } from '@/lib/prisma';

// Mock Stripe
jest.mock('stripe');
jest.mock('@/lib/prisma', () => ({
  prisma: {
    subscription: {
      create: jest.fn(),
      update: jest.fn(),
    },
  },
}));

describe('Stripe Webhook Handler', () => {
  const mockConstructEvent = jest.fn();
  
  beforeEach(() => {
    jest.clearAllMocks();
    (Stripe as any).mockImplementation(() => ({
      webhooks: {
        constructEvent: mockConstructEvent,
      },
    }));
  });

  it('should verify webhook signature', async () => {
    const mockEvent: Stripe.Event = {
      id: 'evt_test',
      type: 'invoice.paid',
      data: {
        object: {
          id: 'in_test',
          subscription: 'sub_test',
          amount_paid: 2000,
          period_end: 1234567890,
        } as any,
      },
    } as any;

    mockConstructEvent.mockReturnValue(mockEvent);

    const request = new NextRequest(
      'http://localhost:3000/api/webhooks/stripe',
      {
        method: 'POST',
        headers: {
          'stripe-signature': 'test_signature',
        },
        body: JSON.stringify(mockEvent),
      }
    );

    const response = await POST(request);
    const data = await response.json();

    expect(response.status).toBe(200);
    expect(data.received).toBe(true);
    expect(mockConstructEvent).toHaveBeenCalled();
  });

  it('should return 400 for missing signature', async () => {
    const request = new NextRequest(
      'http://localhost:3000/api/webhooks/stripe',
      {
        method: 'POST',
        body: '{}',
      }
    );

    const response = await POST(request);

    expect(response.status).toBe(400);
  });

  it('should return 400 for invalid signature', async () => {
    mockConstructEvent.mockImplementation(() => {
      throw new Error('Invalid signature');
    });

    const request = new NextRequest(
      'http://localhost:3000/api/webhooks/stripe',
      {
        method: 'POST',
        headers: {
          'stripe-signature': 'invalid_signature',
        },
        body: 'invalid body',
      }
    );

    const response = await POST(request);

    expect(response.status).toBe(400);
  });

  it('should handle invoice.paid event', async () => {
    const mockEvent: Stripe.Event = {
      id: 'evt_test',
      type: 'invoice.paid',
      data: {
        object: {
          id: 'in_test',
          subscription: 'sub_test',
          amount_paid: 2000,
          period_end: 1234567890,
          customer_email: 'test@example.com',
        } as any,
      },
    } as any;

    mockConstructEvent.mockReturnValue(mockEvent);

    const request = new NextRequest(
      'http://localhost:3000/api/webhooks/stripe',
      {
        method: 'POST',
        headers: {
          'stripe-signature': 'test_signature',
        },
        body: JSON.stringify(mockEvent),
      }
    );

    await POST(request);

    expect(prisma.subscription.update).toHaveBeenCalledWith({
      where: { stripeSubscriptionId: 'sub_test' },
      data: {
        status: 'active',
        currentPeriodEnd: expect.any(Date),
      },
    });
  });

  it('should handle subscription.created event', async () => {
    const mockEvent: Stripe.Event = {
      id: 'evt_test',
      type: 'customer.subscription.created',
      data: {
        object: {
          id: 'sub_test',
          customer: 'cus_test',
          status: 'active',
          items: {
            data: [{ price: { id: 'price_test' } }],
          },
          current_period_end: 1234567890,
        } as any,
      },
    } as any;

    mockConstructEvent.mockReturnValue(mockEvent);

    const request = new NextRequest(
      'http://localhost:3000/api/webhooks/stripe',
      {
        method: 'POST',
        headers: {
          'stripe-signature': 'test_signature',
        },
        body: JSON.stringify(mockEvent),
      }
    );

    await POST(request);

    expect(prisma.subscription.create).toHaveBeenCalled();
  });
});
```

### Express Test

```typescript
// src/routes/webhooks.test.ts
import request from 'supertest';
import express from 'express';
import webhookRouter from './webhooks';
import Stripe from 'stripe';
import { db } from '../db';

jest.mock('stripe');
jest.mock('../db');

const app = express();
app.use('/webhooks', webhookRouter);

describe('Stripe Webhook Endpoint', () => {
  const mockConstructEvent = jest.fn();

  beforeEach(() => {
    jest.clearAllMocks();
    (Stripe as any).mockImplementation(() => ({
      webhooks: {
        constructEvent: mockConstructEvent,
      },
    }));
  });

  it('should process webhook successfully', async () => {
    const mockEvent = {
      id: 'evt_test',
      type: 'invoice.paid',
      data: {
        object: {
          id: 'in_test',
          subscription: 'sub_test',
          amount_paid: 2000,
          period_end: 1234567890,
        },
      },
    };

    mockConstructEvent.mockReturnValue(mockEvent);

    const response = await request(app)
      .post('/webhooks/stripe')
      .set('stripe-signature', 'test_signature')
      .send(mockEvent);

    expect(response.status).toBe(200);
    expect(response.body.received).toBe(true);
  });

  it('should reject missing signature', async () => {
    const response = await request(app)
      .post('/webhooks/stripe')
      .send({});

    expect(response.status).toBe(400);
  });
});
```

## Pytest (FastAPI / Django / Python)

### FastAPI Test

```python
# tests/test_webhooks.py
import pytest
from fastapi.testclient import TestClient
from unittest.mock import patch, MagicMock
from app.main import app
from app.models import Subscription

client = TestClient(app)

@pytest.fixture
def mock_stripe_event():
    return {
        'id': 'evt_test',
        'type': 'invoice.paid',
        'data': {
            'object': {
                'id': 'in_test',
                'subscription': 'sub_test',
                'amount_paid': 2000,
                'period_end': 1234567890,
                'customer_email': 'test@example.com'
            }
        }
    }


def test_stripe_webhook_valid_signature(mock_stripe_event):
    """Test webhook with valid signature."""
    
    with patch('stripe.Webhook.construct_event', return_value=mock_stripe_event):
        response = client.post(
            '/webhooks/stripe',
            json=mock_stripe_event,
            headers={'stripe-signature': 'valid_signature'}
        )
        
        assert response.status_code == 200
        assert response.json() == {'status': 'success'}


def test_stripe_webhook_missing_signature():
    """Test webhook with missing signature."""
    
    response = client.post('/webhooks/stripe', json={})
    
    assert response.status_code == 400
    assert 'No signature' in response.json()['detail']


def test_stripe_webhook_invalid_signature():
    """Test webhook with invalid signature."""
    
    with patch('stripe.Webhook.construct_event', side_effect=Exception('Invalid signature')):
        response = client.post(
            '/webhooks/stripe',
            json={},
            headers={'stripe-signature': 'invalid_signature'}
        )
        
        assert response.status_code == 400


def test_invoice_paid_handler(mock_stripe_event):
    """Test invoice.paid event processing."""
    
    with patch('stripe.Webhook.construct_event', return_value=mock_stripe_event):
        with patch('app.routers.webhooks.get_db') as mock_db:
            mock_session = MagicMock()
            mock_db.return_value.__next__.return_value = mock_session
            
            mock_subscription = MagicMock(spec=Subscription)
            mock_session.query.return_value.filter.return_value.first.return_value = mock_subscription
            
            response = client.post(
                '/webhooks/stripe',
                json=mock_stripe_event,
                headers={'stripe-signature': 'valid_signature'}
            )
            
            assert response.status_code == 200
            assert mock_subscription.status == 'active'


def test_subscription_created_handler():
    """Test subscription.created event processing."""
    
    event = {
        'id': 'evt_test',
        'type': 'customer.subscription.created',
        'data': {
            'object': {
                'id': 'sub_test',
                'customer': 'cus_test',
                'status': 'active',
                'items': {'data': [{'price': {'id': 'price_test'}}]},
                'current_period_end': 1234567890
            }
        }
    }
    
    with patch('stripe.Webhook.construct_event', return_value=event):
        with patch('app.routers.webhooks.get_db') as mock_db:
            mock_session = MagicMock()
            mock_db.return_value.__next__.return_value = mock_session
            
            response = client.post(
                '/webhooks/stripe',
                json=event,
                headers={'stripe-signature': 'valid_signature'}
            )
            
            assert response.status_code == 200
            assert mock_session.add.called
            assert mock_session.commit.called


def test_subscription_updated_handler():
    """Test subscription.updated event processing."""
    
    event = {
        'id': 'evt_test',
        'type': 'customer.subscription.updated',
        'data': {
            'object': {
                'id': 'sub_test',
                'status': 'past_due',
                'current_period_end': 1234567890
            }
        }
    }
    
    with patch('stripe.Webhook.construct_event', return_value=event):
        with patch('app.routers.webhooks.get_db') as mock_db:
            mock_session = MagicMock()
            mock_db.return_value.__next__.return_value = mock_session
            
            mock_subscription = MagicMock(spec=Subscription)
            mock_session.query.return_value.filter.return_value.first.return_value = mock_subscription
            
            response = client.post(
                '/webhooks/stripe',
                json=event,
                headers={'stripe-signature': 'valid_signature'}
            )
            
            assert response.status_code == 200
            assert mock_subscription.status == 'past_due'
```

### Django Test

```python
# myapp/tests/test_webhooks.py
from django.test import TestCase, Client
from unittest.mock import patch, MagicMock
from myapp.models import Subscription
import json

class StripeWebhookTest(TestCase):
    def setUp(self):
        self.client = Client()
        self.url = '/webhooks/stripe/'
        
        self.mock_event = {
            'id': 'evt_test',
            'type': 'invoice.paid',
            'data': {
                'object': {
                    'id': 'in_test',
                    'subscription': 'sub_test',
                    'amount_paid': 2000,
                    'period_end': 1234567890,
                    'customer_email': 'test@example.com'
                }
            }
        }
    
    @patch('stripe.Webhook.construct_event')
    def test_valid_signature(self, mock_construct):
        """Test webhook with valid signature."""
        mock_construct.return_value = self.mock_event
        
        response = self.client.post(
            self.url,
            data=json.dumps(self.mock_event),
            content_type='application/json',
            HTTP_STRIPE_SIGNATURE='valid_signature'
        )
        
        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.json()['status'], 'success')
    
    def test_missing_signature(self):
        """Test webhook with missing signature."""
        response = self.client.post(
            self.url,
            data=json.dumps(self.mock_event),
            content_type='application/json'
        )
        
        self.assertEqual(response.status_code, 400)
    
    @patch('stripe.Webhook.construct_event')
    def test_invalid_signature(self, mock_construct):
        """Test webhook with invalid signature."""
        mock_construct.side_effect = Exception('Invalid signature')
        
        response = self.client.post(
            self.url,
            data=json.dumps(self.mock_event),
            content_type='application/json',
            HTTP_STRIPE_SIGNATURE='invalid_signature'
        )
        
        self.assertEqual(response.status_code, 400)
    
    @patch('stripe.Webhook.construct_event')
    def test_invoice_paid_handler(self, mock_construct):
        """Test invoice.paid event processing."""
        mock_construct.return_value = self.mock_event
        
        # Create test subscription
        subscription = Subscription.objects.create(
            stripe_subscription_id='sub_test',
            stripe_customer_id='cus_test',
            status='active'
        )
        
        response = self.client.post(
            self.url,
            data=json.dumps(self.mock_event),
            content_type='application/json',
            HTTP_STRIPE_SIGNATURE='valid_signature'
        )
        
        self.assertEqual(response.status_code, 200)
        
        # Verify subscription was updated
        subscription.refresh_from_db()
        self.assertEqual(subscription.status, 'active')
    
    @patch('stripe.Webhook.construct_event')
    def test_subscription_created_handler(self, mock_construct):
        """Test subscription.created event processing."""
        event = {
            'id': 'evt_test',
            'type': 'customer.subscription.created',
            'data': {
                'object': {
                    'id': 'sub_new',
                    'customer': 'cus_test',
                    'status': 'active',
                    'items': {'data': [{'price': {'id': 'price_test'}}]},
                    'current_period_end': 1234567890
                }
            }
        }
        mock_construct.return_value = event
        
        response = self.client.post(
            self.url,
            data=json.dumps(event),
            content_type='application/json',
            HTTP_STRIPE_SIGNATURE='valid_signature'
        )
        
        self.assertEqual(response.status_code, 200)
        
        # Verify subscription was created
        subscription = Subscription.objects.get(stripe_subscription_id='sub_new')
        self.assertEqual(subscription.status, 'active')
```

## Test Configuration Files

### Jest Configuration (TypeScript)

```typescript
// jest.config.ts
import type { Config } from 'jest';
import nextJest from 'next/jest';

const createJestConfig = nextJest({
  dir: './',
});

const config: Config = {
  coverageProvider: 'v8',
  testEnvironment: 'node',
  setupFilesAfterEnv: ['<rootDir>/jest.setup.ts'],
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/$1',
  },
};

export default createJestConfig(config);
```

```typescript
// jest.setup.ts
// Add any global test setup here
```

### Pytest Configuration

```ini
# pytest.ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = 
    --verbose
    --cov=app
    --cov-report=html
    --cov-report=term-missing
```

## Integration Tests with Stripe CLI

### Bash Script for Integration Testing

```bash
#!/bin/bash
# test_webhooks.sh

echo "Starting integration tests..."

# Start local server in background
npm run dev &
SERVER_PID=$!
sleep 5  # Wait for server to start

# Start Stripe CLI listener
stripe listen --forward-to localhost:3000/api/webhooks/stripe &
STRIPE_PID=$!
sleep 3  # Wait for listener to connect

# Trigger test events
echo "Triggering test events..."
stripe trigger payment_intent.succeeded
sleep 2

stripe trigger customer.subscription.created
sleep 2

stripe trigger invoice.paid
sleep 2

# Check logs or database for results
echo "Checking results..."
# Add your verification logic here

# Cleanup
kill $SERVER_PID
kill $STRIPE_PID

echo "Integration tests complete!"
```

## Mock Data Helpers

### TypeScript Mock Helpers

```typescript
// test/helpers/stripe-mocks.ts
import Stripe from 'stripe';

export function createMockSubscription(
  overrides?: Partial<Stripe.Subscription>
): Stripe.Subscription {
  return {
    id: 'sub_test',
    object: 'subscription',
    customer: 'cus_test',
    status: 'active',
    items: {
      object: 'list',
      data: [
        {
          id: 'si_test',
          price: {
            id: 'price_test',
            object: 'price',
            active: true,
            currency: 'usd',
            unit_amount: 2000,
          } as any,
        } as any,
      ],
    } as any,
    current_period_end: 1234567890,
    current_period_start: 1234567800,
    ...overrides,
  } as any;
}

export function createMockInvoice(
  overrides?: Partial<Stripe.Invoice>
): Stripe.Invoice {
  return {
    id: 'in_test',
    object: 'invoice',
    subscription: 'sub_test',
    customer: 'cus_test',
    customer_email: 'test@example.com',
    amount_paid: 2000,
    period_end: 1234567890,
    period_start: 1234567800,
    ...overrides,
  } as any;
}

export function createMockPaymentIntent(
  overrides?: Partial<Stripe.PaymentIntent>
): Stripe.PaymentIntent {
  return {
    id: 'pi_test',
    object: 'payment_intent',
    amount: 2000,
    currency: 'usd',
    status: 'succeeded',
    metadata: {},
    ...overrides,
  } as any;
}
```

### Python Mock Helpers

```python
# tests/helpers/stripe_mocks.py

def create_mock_subscription(**kwargs):
    """Create mock subscription data."""
    return {
        'id': 'sub_test',
        'customer': 'cus_test',
        'status': 'active',
        'items': {
            'data': [{
                'price': {
                    'id': 'price_test',
                    'unit_amount': 2000,
                    'currency': 'usd'
                }
            }]
        },
        'current_period_end': 1234567890,
        'current_period_start': 1234567800,
        **kwargs
    }


def create_mock_invoice(**kwargs):
    """Create mock invoice data."""
    return {
        'id': 'in_test',
        'subscription': 'sub_test',
        'customer': 'cus_test',
        'customer_email': 'test@example.com',
        'amount_paid': 2000,
        'period_end': 1234567890,
        'period_start': 1234567800,
        **kwargs
    }


def create_mock_payment_intent(**kwargs):
    """Create mock payment intent data."""
    return {
        'id': 'pi_test',
        'amount': 2000,
        'currency': 'usd',
        'status': 'succeeded',
        'metadata': {},
        **kwargs
    }
```

## Testing Best Practices

1. **Test signature verification** - Always verify the handler checks signatures
2. **Test each event type** - Create tests for all handled event types
3. **Test error cases** - Missing signature, invalid signature, processing errors
4. **Test idempotency** - Verify duplicate events don't cause duplicate processing
5. **Mock external services** - Mock Stripe SDK, database, email service
6. **Integration tests** - Use Stripe CLI for end-to-end testing
7. **Test data consistency** - Verify database updates are correct
