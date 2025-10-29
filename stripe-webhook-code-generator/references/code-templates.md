# Stripe Webhook Code Templates

Complete, production-ready webhook handler templates for all major frameworks.

## Next.js 14 App Router

### Complete Handler

```typescript
// app/api/webhooks/stripe/route.ts
import { headers } from 'next/headers';
import { NextRequest, NextResponse } from 'next/server';
import Stripe from 'stripe';
import { prisma } from '@/lib/prisma';  // Adjust import
import { sendEmail } from '@/lib/email';  // Adjust import

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2024-06-20',  // Match your API version
});

const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET!;

// Required for webhooks
export const dynamic = 'force-dynamic';

export async function POST(req: NextRequest) {
  const body = await req.text();
  const signature = headers().get('stripe-signature');

  if (!signature) {
    return NextResponse.json(
      { error: 'No signature provided' },
      { status: 400 }
    );
  }

  let event: Stripe.Event;

  try {
    event = stripe.webhooks.constructEvent(body, signature, webhookSecret);
  } catch (err: any) {
    console.error('⚠️ Webhook signature verification failed:', err.message);
    return NextResponse.json(
      { error: `Webhook Error: ${err.message}` },
      { status: 400 }
    );
  }

  console.log('✅ Webhook received:', event.type, event.id);

  try {
    switch (event.type) {
      case 'customer.subscription.created':
        await handleSubscriptionCreated(event.data.object as Stripe.Subscription);
        break;

      case 'customer.subscription.updated':
        await handleSubscriptionUpdated(event.data.object as Stripe.Subscription);
        break;

      case 'customer.subscription.deleted':
        await handleSubscriptionDeleted(event.data.object as Stripe.Subscription);
        break;

      case 'invoice.paid':
        await handleInvoicePaid(event.data.object as Stripe.Invoice);
        break;

      case 'invoice.payment_failed':
        await handleInvoicePaymentFailed(event.data.object as Stripe.Invoice);
        break;

      case 'payment_intent.succeeded':
        await handlePaymentIntentSucceeded(event.data.object as Stripe.PaymentIntent);
        break;

      case 'payment_intent.payment_failed':
        await handlePaymentIntentFailed(event.data.object as Stripe.PaymentIntent);
        break;

      default:
        console.log(`Unhandled event type: ${event.type}`);
    }

    return NextResponse.json({ received: true }, { status: 200 });
  } catch (err: any) {
    console.error('❌ Error processing webhook:', err);
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    );
  }
}

// Event Handlers
async function handleSubscriptionCreated(subscription: Stripe.Subscription) {
  console.log('Processing new subscription:', subscription.id);
  
  await prisma.subscription.create({
    data: {
      stripeSubscriptionId: subscription.id,
      stripeCustomerId: subscription.customer as string,
      status: subscription.status,
      priceId: subscription.items.data[0].price.id,
      currentPeriodEnd: new Date(subscription.current_period_end * 1000),
      user: {
        connect: {
          stripeCustomerId: subscription.customer as string,
        },
      },
    },
  });
}

async function handleSubscriptionUpdated(subscription: Stripe.Subscription) {
  console.log('Processing subscription update:', subscription.id);
  
  await prisma.subscription.update({
    where: { stripeSubscriptionId: subscription.id },
    data: {
      status: subscription.status,
      currentPeriodEnd: new Date(subscription.current_period_end * 1000),
    },
  });
}

async function handleSubscriptionDeleted(subscription: Stripe.Subscription) {
  console.log('Processing subscription cancellation:', subscription.id);
  
  await prisma.subscription.update({
    where: { stripeSubscriptionId: subscription.id },
    data: { status: 'canceled' },
  });
}

async function handleInvoicePaid(invoice: Stripe.Invoice) {
  console.log('Processing invoice payment:', invoice.id);
  
  if (!invoice.subscription) return;
  
  await prisma.subscription.update({
    where: { stripeSubscriptionId: invoice.subscription as string },
    data: {
      status: 'active',
      currentPeriodEnd: new Date(invoice.period_end! * 1000),
    },
  });
  
  // Send receipt email
  await sendEmail({
    to: invoice.customer_email!,
    subject: 'Payment Receipt',
    template: 'receipt',
    data: { amount: invoice.amount_paid / 100 },
  });
}

async function handleInvoicePaymentFailed(invoice: Stripe.Invoice) {
  console.log('Processing payment failure:', invoice.id);
  
  if (!invoice.subscription) return;
  
  await prisma.subscription.update({
    where: { stripeSubscriptionId: invoice.subscription as string },
    data: { status: 'past_due' },
  });
  
  // Send payment failure email
  await sendEmail({
    to: invoice.customer_email!,
    subject: 'Payment Failed',
    template: 'payment-failed',
    data: { invoiceId: invoice.id },
  });
}

async function handlePaymentIntentSucceeded(paymentIntent: Stripe.PaymentIntent) {
  console.log('Processing payment success:', paymentIntent.id);
  
  if (paymentIntent.metadata.orderId) {
    await prisma.order.update({
      where: { id: paymentIntent.metadata.orderId },
      data: { status: 'paid' },
    });
  }
}

async function handlePaymentIntentFailed(paymentIntent: Stripe.PaymentIntent) {
  console.log('Processing payment failure:', paymentIntent.id);
  
  if (paymentIntent.metadata.orderId) {
    await prisma.order.update({
      where: { id: paymentIntent.metadata.orderId },
      data: { status: 'payment_failed' },
    });
  }
}
```

## Next.js 13 Pages Router

### Complete Handler

```typescript
// pages/api/webhooks/stripe.ts
import { NextApiRequest, NextApiResponse } from 'next';
import getRawBody from 'raw-body';
import Stripe from 'stripe';
import { prisma } from '@/lib/prisma';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2024-06-20',
});

const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET!;

// Disable body parser (required for signature verification)
export const config = {
  api: {
    bodyParser: false,
  },
};

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }

  const sig = req.headers['stripe-signature'];

  if (!sig) {
    return res.status(400).json({ error: 'No signature' });
  }

  let event: Stripe.Event;

  try {
    const body = await getRawBody(req);
    event = stripe.webhooks.constructEvent(body, sig, webhookSecret);
  } catch (err: any) {
    console.error('Webhook signature verification failed:', err.message);
    return res.status(400).json({ error: err.message });
  }

  console.log('✅ Webhook received:', event.type, event.id);

  try {
    switch (event.type) {
      case 'invoice.paid':
        await handleInvoicePaid(event.data.object as Stripe.Invoice);
        break;
      
      // Add other event handlers...
      
      default:
        console.log(`Unhandled event type: ${event.type}`);
    }

    return res.status(200).json({ received: true });
  } catch (err: any) {
    console.error('Error processing webhook:', err);
    return res.status(500).json({ error: 'Internal server error' });
  }
}

async function handleInvoicePaid(invoice: Stripe.Invoice) {
  // Implementation same as App Router
}
```

## Express.js

### Complete Handler

```typescript
// src/routes/webhooks.ts
import express, { Request, Response } from 'express';
import Stripe from 'stripe';
import { db } from '../db';  // Adjust import
import { sendEmail } from '../services/email';  // Adjust import

const router = express.Router();

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2024-06-20',
});

const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET!;

// Webhook endpoint - MUST use raw body
router.post(
  '/stripe',
  express.raw({ type: 'application/json' }),
  async (req: Request, res: Response) => {
    const sig = req.headers['stripe-signature'];

    if (!sig) {
      return res.status(400).send('No signature');
    }

    let event: Stripe.Event;

    try {
      event = stripe.webhooks.constructEvent(req.body, sig, webhookSecret);
    } catch (err: any) {
      console.error('Webhook signature verification failed:', err.message);
      return res.status(400).send(`Webhook Error: ${err.message}`);
    }

    console.log('✅ Webhook received:', event.type, event.id);

    // Respond immediately
    res.status(200).json({ received: true });

    // Process asynchronously (don't await)
    processWebhookEvent(event).catch((err) => {
      console.error('Error processing webhook:', err);
    });
  }
);

async function processWebhookEvent(event: Stripe.Event) {
  switch (event.type) {
    case 'customer.subscription.created':
      await handleSubscriptionCreated(event.data.object as Stripe.Subscription);
      break;

    case 'invoice.paid':
      await handleInvoicePaid(event.data.object as Stripe.Invoice);
      break;

    // Add more handlers...

    default:
      console.log(`Unhandled event type: ${event.type}`);
  }
}

async function handleSubscriptionCreated(subscription: Stripe.Subscription) {
  await db.subscription.create({
    stripeSubscriptionId: subscription.id,
    stripeCustomerId: subscription.customer,
    status: subscription.status,
    priceId: subscription.items.data[0].price.id,
    currentPeriodEnd: new Date(subscription.current_period_end * 1000),
  });
}

async function handleInvoicePaid(invoice: Stripe.Invoice) {
  if (!invoice.subscription) return;

  await db.subscription.update(
    { stripeSubscriptionId: invoice.subscription },
    {
      status: 'active',
      currentPeriodEnd: new Date(invoice.period_end! * 1000),
    }
  );

  await sendEmail({
    to: invoice.customer_email!,
    subject: 'Receipt',
    template: 'receipt',
    data: { amount: invoice.amount_paid / 100 },
  });
}

export default router;
```

```typescript
// src/index.ts or src/app.ts
import express from 'express';
import webhookRoutes from './routes/webhooks';

const app = express();

// IMPORTANT: Mount webhook route BEFORE JSON body parser
app.use('/webhooks', webhookRoutes);

// Other routes with JSON parsing
app.use(express.json());
app.use('/api', otherRoutes);

app.listen(3000);
```

## FastAPI

### Complete Handler

```python
# app/routers/webhooks.py
from fastapi import APIRouter, Request, HTTPException, Header
from typing import Optional
import stripe
import os
from datetime import datetime
from app.database import get_db  # Adjust import
from app.models import Subscription  # Adjust import
from app.services.email import send_email  # Adjust import

router = APIRouter()

stripe.api_key = os.environ['STRIPE_SECRET_KEY']
webhook_secret = os.environ['STRIPE_WEBHOOK_SECRET']

@router.post("/stripe")
async def stripe_webhook(
    request: Request,
    stripe_signature: Optional[str] = Header(None)
):
    """Handle Stripe webhook events."""
    
    if not stripe_signature:
        raise HTTPException(status_code=400, detail="No signature")
    
    payload = await request.body()
    
    try:
        event = stripe.Webhook.construct_event(
            payload, stripe_signature, webhook_secret
        )
    except ValueError as e:
        raise HTTPException(status_code=400, detail=f"Invalid payload: {str(e)}")
    except stripe.error.SignatureVerificationError as e:
        raise HTTPException(status_code=400, detail=f"Invalid signature: {str(e)}")
    
    print(f"✅ Webhook received: {event['type']} ({event['id']})")
    
    try:
        if event['type'] == 'customer.subscription.created':
            await handle_subscription_created(event['data']['object'])
        
        elif event['type'] == 'customer.subscription.updated':
            await handle_subscription_updated(event['data']['object'])
        
        elif event['type'] == 'customer.subscription.deleted':
            await handle_subscription_deleted(event['data']['object'])
        
        elif event['type'] == 'invoice.paid':
            await handle_invoice_paid(event['data']['object'])
        
        elif event['type'] == 'invoice.payment_failed':
            await handle_invoice_payment_failed(event['data']['object'])
        
        elif event['type'] == 'payment_intent.succeeded':
            await handle_payment_intent_succeeded(event['data']['object'])
        
        else:
            print(f"Unhandled event type: {event['type']}")
    
    except Exception as e:
        print(f"❌ Error processing webhook: {str(e)}")
        raise HTTPException(status_code=500, detail="Error processing webhook")
    
    return {"status": "success"}


async def handle_subscription_created(subscription: dict):
    """Handle new subscription."""
    print(f"Processing new subscription: {subscription['id']}")
    
    db = next(get_db())
    
    new_sub = Subscription(
        stripe_subscription_id=subscription['id'],
        stripe_customer_id=subscription['customer'],
        status=subscription['status'],
        price_id=subscription['items']['data'][0]['price']['id'],
        current_period_end=datetime.fromtimestamp(subscription['current_period_end']),
    )
    
    db.add(new_sub)
    db.commit()


async def handle_subscription_updated(subscription: dict):
    """Handle subscription update."""
    print(f"Processing subscription update: {subscription['id']}")
    
    db = next(get_db())
    
    sub = db.query(Subscription).filter(
        Subscription.stripe_subscription_id == subscription['id']
    ).first()
    
    if sub:
        sub.status = subscription['status']
        sub.current_period_end = datetime.fromtimestamp(subscription['current_period_end'])
        db.commit()


async def handle_subscription_deleted(subscription: dict):
    """Handle subscription cancellation."""
    print(f"Processing subscription cancellation: {subscription['id']}")
    
    db = next(get_db())
    
    sub = db.query(Subscription).filter(
        Subscription.stripe_subscription_id == subscription['id']
    ).first()
    
    if sub:
        sub.status = 'canceled'
        db.commit()


async def handle_invoice_paid(invoice: dict):
    """Handle successful invoice payment."""
    print(f"Processing invoice payment: {invoice['id']}")
    
    if not invoice['subscription']:
        return
    
    db = next(get_db())
    
    sub = db.query(Subscription).filter(
        Subscription.stripe_subscription_id == invoice['subscription']
    ).first()
    
    if sub:
        sub.status = 'active'
        sub.current_period_end = datetime.fromtimestamp(invoice['period_end'])
        db.commit()
    
    # Send receipt email
    await send_email(
        to=invoice['customer_email'],
        subject='Payment Receipt',
        template='receipt',
        data={'amount': invoice['amount_paid'] / 100}
    )


async def handle_invoice_payment_failed(invoice: dict):
    """Handle failed invoice payment."""
    print(f"Processing payment failure: {invoice['id']}")
    
    if not invoice['subscription']:
        return
    
    db = next(get_db())
    
    sub = db.query(Subscription).filter(
        Subscription.stripe_subscription_id == invoice['subscription']
    ).first()
    
    if sub:
        sub.status = 'past_due'
        db.commit()
    
    # Send payment failure email
    await send_email(
        to=invoice['customer_email'],
        subject='Payment Failed',
        template='payment-failed',
        data={'invoice_id': invoice['id']}
    )


async def handle_payment_intent_succeeded(payment_intent: dict):
    """Handle successful payment."""
    print(f"Processing payment success: {payment_intent['id']}")
    
    # Implement based on your requirements
```

```python
# app/main.py
from fastapi import FastAPI
from app.routers import webhooks

app = FastAPI()

app.include_router(webhooks.router, prefix="/webhooks", tags=["webhooks"])
```

## Django

### Complete Handler

```python
# myapp/views/webhooks.py
from django.http import JsonResponse, HttpResponse
from django.views.decorators.csrf import csrf_exempt
from django.views.decorators.http import require_http_methods
import stripe
import os
import json
from datetime import datetime
from myapp.models import Subscription  # Adjust import
from myapp.services.email import send_email  # Adjust import

stripe.api_key = os.environ.get('STRIPE_SECRET_KEY')
webhook_secret = os.environ.get('STRIPE_WEBHOOK_SECRET')

@csrf_exempt
@require_http_methods(["POST"])
def stripe_webhook(request):
    """Handle Stripe webhook events."""
    
    payload = request.body
    sig = request.META.get('HTTP_STRIPE_SIGNATURE')
    
    if not sig:
        return HttpResponse('No signature', status=400)
    
    try:
        event = stripe.Webhook.construct_event(
            payload, sig, webhook_secret
        )
    except ValueError:
        return HttpResponse('Invalid payload', status=400)
    except stripe.error.SignatureVerificationError:
        return HttpResponse('Invalid signature', status=400)
    
    print(f"✅ Webhook received: {event['type']} ({event['id']})")
    
    try:
        event_type = event['type']
        data = event['data']['object']
        
        if event_type == 'customer.subscription.created':
            handle_subscription_created(data)
        
        elif event_type == 'customer.subscription.updated':
            handle_subscription_updated(data)
        
        elif event_type == 'customer.subscription.deleted':
            handle_subscription_deleted(data)
        
        elif event_type == 'invoice.paid':
            handle_invoice_paid(data)
        
        elif event_type == 'invoice.payment_failed':
            handle_invoice_payment_failed(data)
        
        else:
            print(f"Unhandled event type: {event_type}")
    
    except Exception as e:
        print(f"❌ Error processing webhook: {str(e)}")
        return HttpResponse('Error processing webhook', status=500)
    
    return JsonResponse({'status': 'success'})


def handle_subscription_created(subscription):
    """Handle new subscription."""
    print(f"Processing new subscription: {subscription['id']}")
    
    Subscription.objects.create(
        stripe_subscription_id=subscription['id'],
        stripe_customer_id=subscription['customer'],
        status=subscription['status'],
        price_id=subscription['items']['data'][0]['price']['id'],
        current_period_end=datetime.fromtimestamp(subscription['current_period_end']),
    )


def handle_subscription_updated(subscription):
    """Handle subscription update."""
    print(f"Processing subscription update: {subscription['id']}")
    
    sub = Subscription.objects.get(stripe_subscription_id=subscription['id'])
    sub.status = subscription['status']
    sub.current_period_end = datetime.fromtimestamp(subscription['current_period_end'])
    sub.save()


def handle_subscription_deleted(subscription):
    """Handle subscription cancellation."""
    print(f"Processing subscription cancellation: {subscription['id']}")
    
    sub = Subscription.objects.get(stripe_subscription_id=subscription['id'])
    sub.status = 'canceled'
    sub.save()


def handle_invoice_paid(invoice):
    """Handle successful invoice payment."""
    print(f"Processing invoice payment: {invoice['id']}")
    
    if not invoice['subscription']:
        return
    
    sub = Subscription.objects.get(stripe_subscription_id=invoice['subscription'])
    sub.status = 'active'
    sub.current_period_end = datetime.fromtimestamp(invoice['period_end'])
    sub.save()
    
    # Send receipt email
    send_email(
        to=invoice['customer_email'],
        subject='Payment Receipt',
        template='receipt',
        context={'amount': invoice['amount_paid'] / 100}
    )


def handle_invoice_payment_failed(invoice):
    """Handle failed invoice payment."""
    print(f"Processing payment failure: {invoice['id']}")
    
    if not invoice['subscription']:
        return
    
    sub = Subscription.objects.get(stripe_subscription_id=invoice['subscription'])
    sub.status = 'past_due'
    sub.save()
    
    # Send payment failure email
    send_email(
        to=invoice['customer_email'],
        subject='Payment Failed',
        template='payment-failed',
        context={'invoice_id': invoice['id']}
    )
```

```python
# urls.py
from django.urls import path
from myapp.views import webhooks

urlpatterns = [
    path('webhooks/stripe/', webhooks.stripe_webhook, name='stripe_webhook'),
    # Other URLs...
]
```

## Environment Variables Template

```bash
# .env.example

# Stripe API Keys
# Get from: https://dashboard.stripe.com/apikeys
STRIPE_SECRET_KEY=sk_test_...
STRIPE_PUBLISHABLE_KEY=pk_test_...

# Stripe Webhook Secret
# Get from: https://dashboard.stripe.com/webhooks
# After creating endpoint, click "Reveal" on signing secret
STRIPE_WEBHOOK_SECRET=whsec_...

# For local development with Stripe CLI:
# 1. Run: stripe listen --forward-to localhost:3000/api/webhooks/stripe
# 2. Copy the webhook signing secret printed by CLI
# 3. Use that secret in STRIPE_WEBHOOK_SECRET
```

## Common Integration Patterns

### Idempotency Check

```typescript
// With database
async function processEvent(event: Stripe.Event) {
  const existing = await prisma.webhookEvent.findUnique({
    where: { eventId: event.id }
  });
  
  if (existing) {
    console.log('Event already processed');
    return;
  }
  
  await handleEvent(event);
  
  await prisma.webhookEvent.create({
    data: { eventId: event.id, processedAt: new Date() }
  });
}
```

### Email with Resend (Next.js)

```typescript
import { Resend } from 'resend';

const resend = new Resend(process.env.RESEND_API_KEY);

async function sendEmail(options: {
  to: string;
  subject: string;
  template: string;
  data: any;
}) {
  await resend.emails.send({
    from: 'noreply@yourdomain.com',
    to: options.to,
    subject: options.subject,
    html: renderTemplate(options.template, options.data),
  });
}
```

### Email with SendGrid (Python)

```python
from sendgrid import SendGridAPIClient
from sendgrid.helpers.mail import Mail
import os

sg = SendGridAPIClient(os.environ['SENDGRID_API_KEY'])

async def send_email(to: str, subject: str, template: str, data: dict):
    message = Mail(
        from_email='noreply@yourdomain.com',
        to_emails=to,
        subject=subject,
        html_content=render_template(template, data)
    )
    sg.send(message)
```
