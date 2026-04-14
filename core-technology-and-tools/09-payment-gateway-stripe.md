# Payment Gateway Integration (Stripe) - Study Material

> Based on skills from [ganeshavhad.com](https://ganeshavhad.com) — Software Developer Specialist
> *"Implemented a secure payment gateway solution using Stripe, ensuring seamless and reliable payment processing"*

---

## 1. Payment Gateway Overview

A payment gateway authorizes and processes online payments. It acts as a bridge between the merchant and the financial networks.

### Payment Flow

```
Customer      Merchant        Payment        Card           Issuing
(Browser)     (Server)        Gateway        Network        Bank
   │             │              │              │              │
   │─── Card ───►│              │              │              │
   │  Details    │── Charge ───►│              │              │
   │             │              │── Authorize ►│              │
   │             │              │              │── Verify ───►│
   │             │              │              │◄── OK ───────│
   │             │              │◄─ Approved ──│              │
   │             │◄── Token ────│              │              │
   │◄── Receipt ─│              │              │              │
```

---

## 2. Stripe Integration (Node.js)

### Setup

```bash
npm install stripe
```

### Server-Side Setup

```typescript
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2024-04-10'
});
```

### 2.1 Payment Intents (Recommended)

```typescript
// Create a PaymentIntent
app.post('/api/create-payment-intent', async (req, res) => {
  try {
    const { amount, currency = 'usd', metadata } = req.body;

    // Validate amount
    if (!amount || amount <= 0) {
      return res.status(400).json({ error: 'Invalid amount' });
    }

    const paymentIntent = await stripe.paymentIntents.create({
      amount: Math.round(amount * 100), // Convert to cents
      currency,
      automatic_payment_methods: { enabled: true },
      metadata: {
        orderId: metadata?.orderId,
        userId: req.user.id
      }
    });

    res.json({
      clientSecret: paymentIntent.client_secret
    });
  } catch (error) {
    console.error('PaymentIntent error:', error);
    res.status(500).json({ error: 'Payment initialization failed' });
  }
});
```

### 2.2 Checkout Sessions (Hosted Payment Page)

```typescript
app.post('/api/create-checkout-session', async (req, res) => {
  try {
    const { items, successUrl, cancelUrl } = req.body;

    const session = await stripe.checkout.sessions.create({
      payment_method_types: ['card'],
      mode: 'payment',
      line_items: items.map((item: any) => ({
        price_data: {
          currency: 'usd',
          product_data: {
            name: item.name,
            description: item.description,
            images: [item.imageUrl]
          },
          unit_amount: Math.round(item.price * 100)
        },
        quantity: item.quantity
      })),
      success_url: `${successUrl}?session_id={CHECKOUT_SESSION_ID}`,
      cancel_url: cancelUrl,
      customer_email: req.user.email,
      metadata: { userId: req.user.id }
    });

    res.json({ sessionId: session.id, url: session.url });
  } catch (error) {
    console.error('Checkout error:', error);
    res.status(500).json({ error: 'Failed to create checkout session' });
  }
});
```

### 2.3 Subscriptions

```typescript
// Create a subscription
app.post('/api/create-subscription', async (req, res) => {
  try {
    const { priceId, paymentMethodId } = req.body;

    // Get or create Stripe customer
    let customer = await getOrCreateStripeCustomer(req.user);

    // Attach payment method
    await stripe.paymentMethods.attach(paymentMethodId, {
      customer: customer.id
    });

    // Set as default payment method
    await stripe.customers.update(customer.id, {
      invoice_settings: { default_payment_method: paymentMethodId }
    });

    // Create subscription
    const subscription = await stripe.subscriptions.create({
      customer: customer.id,
      items: [{ price: priceId }],
      payment_behavior: 'default_incomplete',
      expand: ['latest_invoice.payment_intent']
    });

    const invoice = subscription.latest_invoice as Stripe.Invoice;
    const paymentIntent = invoice.payment_intent as Stripe.PaymentIntent;

    res.json({
      subscriptionId: subscription.id,
      clientSecret: paymentIntent.client_secret
    });
  } catch (error) {
    console.error('Subscription error:', error);
    res.status(500).json({ error: 'Subscription creation failed' });
  }
});
```

---

## 3. Webhook Handling (Critical)

```typescript
// Webhook endpoint — receives events from Stripe
app.post('/api/webhooks/stripe',
  express.raw({ type: 'application/json' }), // Raw body needed for verification
  async (req, res) => {
    const sig = req.headers['stripe-signature'] as string;

    let event: Stripe.Event;

    try {
      // CRITICAL: Verify webhook signature
      event = stripe.webhooks.constructEvent(
        req.body,
        sig,
        process.env.STRIPE_WEBHOOK_SECRET!
      );
    } catch (err) {
      console.error('Webhook signature verification failed:', err);
      return res.status(400).send(`Webhook Error: ${err.message}`);
    }

    // Handle specific events
    switch (event.type) {
      case 'payment_intent.succeeded': {
        const paymentIntent = event.data.object as Stripe.PaymentIntent;
        await handlePaymentSuccess(paymentIntent);
        break;
      }

      case 'payment_intent.payment_failed': {
        const paymentIntent = event.data.object as Stripe.PaymentIntent;
        await handlePaymentFailure(paymentIntent);
        break;
      }

      case 'checkout.session.completed': {
        const session = event.data.object as Stripe.Checkout.Session;
        await fulfillOrder(session);
        break;
      }

      case 'customer.subscription.updated': {
        const subscription = event.data.object as Stripe.Subscription;
        await updateSubscriptionStatus(subscription);
        break;
      }

      case 'customer.subscription.deleted': {
        const subscription = event.data.object as Stripe.Subscription;
        await cancelSubscription(subscription);
        break;
      }

      case 'invoice.payment_failed': {
        const invoice = event.data.object as Stripe.Invoice;
        await handleFailedInvoice(invoice);
        break;
      }

      default:
        console.log(`Unhandled event type: ${event.type}`);
    }

    res.json({ received: true });
  }
);

// Business logic handlers
async function handlePaymentSuccess(paymentIntent: Stripe.PaymentIntent) {
  const orderId = paymentIntent.metadata.orderId;

  await db.query(
    'UPDATE orders SET status = $1, payment_id = $2, paid_at = NOW() WHERE id = $3',
    ['paid', paymentIntent.id, orderId]
  );

  await sendReceiptEmail(paymentIntent);
}

async function handlePaymentFailure(paymentIntent: Stripe.PaymentIntent) {
  const orderId = paymentIntent.metadata.orderId;

  await db.query(
    'UPDATE orders SET status = $1 WHERE id = $2',
    ['payment_failed', orderId]
  );

  await notifyCustomerOfFailure(paymentIntent);
}
```

---

## 4. Client-Side (Angular/React)

```typescript
// Angular service for Stripe
import { loadStripe, Stripe } from '@stripe/stripe-js';

@Injectable({ providedIn: 'root' })
export class PaymentService {
  private stripePromise = loadStripe(environment.stripePublicKey);

  async createPaymentIntent(amount: number): Promise<string> {
    const response = await this.http.post<{ clientSecret: string }>(
      '/api/create-payment-intent',
      { amount }
    ).toPromise();
    return response!.clientSecret;
  }

  async confirmPayment(clientSecret: string, cardElement: any): Promise<void> {
    const stripe = await this.stripePromise;
    if (!stripe) throw new Error('Stripe not loaded');

    const { error, paymentIntent } = await stripe.confirmCardPayment(
      clientSecret,
      { payment_method: { card: cardElement } }
    );

    if (error) {
      throw new Error(error.message);
    }

    if (paymentIntent?.status !== 'succeeded') {
      throw new Error('Payment not completed');
    }
  }

  async redirectToCheckout(sessionId: string): Promise<void> {
    const stripe = await this.stripePromise;
    if (!stripe) throw new Error('Stripe not loaded');

    const { error } = await stripe.redirectToCheckout({ sessionId });
    if (error) throw new Error(error.message);
  }
}
```

---

## 5. Refunds & Disputes

```typescript
// Issue a refund
app.post('/api/refund', async (req, res) => {
  try {
    const { paymentIntentId, amount, reason } = req.body;

    const refund = await stripe.refunds.create({
      payment_intent: paymentIntentId,
      amount: amount ? Math.round(amount * 100) : undefined, // Partial or full
      reason: reason || 'requested_by_customer'
    });

    await db.query(
      'UPDATE orders SET status = $1, refund_id = $2 WHERE payment_id = $3',
      ['refunded', refund.id, paymentIntentId]
    );

    res.json({ refundId: refund.id, status: refund.status });
  } catch (error) {
    console.error('Refund error:', error);
    res.status(500).json({ error: 'Refund failed' });
  }
});
```

---

## 6. Security Best Practices

| Practice | Implementation |
|----------|---------------|
| **Never expose secret key** | Only on server-side, use env variables |
| **Verify webhooks** | Always validate Stripe signature |
| **Use PaymentIntents** | Modern, SCA-compliant flow |
| **PCI Compliance** | Use Stripe Elements (card data never hits your server) |
| **Idempotency keys** | Prevent duplicate charges on retries |
| **Amount validation** | Validate on server, never trust client |
| **HTTPS only** | All payment endpoints over TLS |
| **Audit logging** | Log all payment events for reconciliation |

### Idempotency
```typescript
// Prevent duplicate charges
const paymentIntent = await stripe.paymentIntents.create(
  { amount: 2000, currency: 'usd' },
  { idempotencyKey: `order-${orderId}` }
);
```

---

## 7. Testing

```typescript
// Use Stripe test card numbers
const TEST_CARDS = {
  success: '4242424242424242',
  declined: '4000000000000002',
  requires_auth: '4000002500003155',
  insufficient_funds: '4000000000009995'
};

// Test webhook events via CLI
// stripe listen --forward-to localhost:3000/api/webhooks/stripe
// stripe trigger payment_intent.succeeded
```

---

## 8. Juspay & Cashfree Integration (India)

> *"Incorporating Juspay and Cashfree payment gateways"*

### Key Differences

| Feature | Stripe | Juspay | Cashfree |
|---------|--------|--------|----------|
| Region | Global | India-focused | India-focused |
| UPI Support | Limited | Native | Native |
| Net Banking | Limited | Extensive | Extensive |
| Wallets | Apple/Google Pay | PayTM, PhonePe | PayTM, Amazon |
| Integration | Direct API | Orchestration layer | Direct API |

### Cashfree Example (Node.js)
```typescript
import { Cashfree } from 'cashfree-pg';

Cashfree.XClientId = process.env.CASHFREE_APP_ID;
Cashfree.XClientSecret = process.env.CASHFREE_SECRET_KEY;
Cashfree.XEnvironment = Cashfree.Environment.SANDBOX;

// Create order
const orderRequest = {
  order_amount: 100.00,
  order_currency: 'INR',
  order_id: `order_${Date.now()}`,
  customer_details: {
    customer_id: 'customer_123',
    customer_email: 'user@example.com',
    customer_phone: '9876543210'
  }
};

const response = await Cashfree.PGCreateOrder('2023-08-01', orderRequest);
// Returns payment_session_id for client-side SDK
```

---

## 9. Interview Questions

1. **What is PCI DSS compliance?** — Payment Card Industry Data Security Standard. Stripe Elements handle card data so merchants stay PCI compliant (SAQ-A).
2. **Why use webhooks?** — Payments are async. Webhooks notify your server of events (success, failure, refund) reliably.
3. **PaymentIntent vs Charge?** — PaymentIntent is the modern API supporting SCA (3D Secure). Charge is legacy.
4. **What is SCA?** — Strong Customer Authentication. EU regulation requiring two-factor authentication for payments.
5. **How to handle failed payments?** — Webhooks for `payment_intent.payment_failed`, retry logic, notify customer, update order status.
6. **What are idempotency keys?** — Unique keys to prevent duplicate operations when retrying API calls.
7. **How to test payments?** — Use Stripe test mode with test card numbers. `stripe listen` for webhook testing.
8. **What is a payment orchestration layer?** — Juspay-style layer that routes payments to multiple gateways for optimal success rates.

---

## 10. Practice Exercises

1. Implement a complete checkout flow with Stripe PaymentIntents
2. Set up webhook handling with signature verification
3. Build a subscription management system
4. Implement refund functionality with audit logging
5. Create an order management system tracking payment states
