# Payment Module

Aggregated payment gateway — Stripe (international), Alipay, WechatPay stubs. Running on **Node.js Cloud Functions** with Express.

## File: `cloud-functions/api/pay/[[default]].js`

```javascript
import express from 'express';

const app = express();

// Raw body needed for Stripe webhook signature verification
app.use('/webhook', express.raw({ type: 'application/json' }));
app.use(express.json());

// ============================================================
// CHECKOUT — POST /api/pay/checkout
// ============================================================
app.post('/checkout', async (req, res) => {
  const userId = getUserId(req);
  if (!userId) return res.status(401).json({ error: 'Unauthorized' });

  const { orderId, method = 'stripe' } = req.body;
  if (!orderId) return res.status(400).json({ error: 'orderId required' });

  try {
    switch (method) {
      case 'stripe':
        return await handleStripeCheckout(req, res, orderId, userId);
      case 'alipay':
        return await handleAlipayCheckout(req, res, orderId, userId);
      case 'wechatpay':
        return await handleWechatPayCheckout(req, res, orderId, userId);
      default:
        return res.status(400).json({ error: `Unsupported payment method: ${method}` });
    }
  } catch (err) {
    console.error('Checkout error:', err);
    res.status(500).json({ error: 'Payment initiation failed' });
  }
});

// ============================================================
// STRIPE
// ============================================================
async function handleStripeCheckout(req, res, orderId, userId) {
  const stripe = (await import('stripe')).default(process.env.STRIPE_SECRET_KEY);

  // In production, fetch order from database
  const { amount, currency = 'usd', description } = req.body;

  const session = await stripe.checkout.sessions.create({
    payment_method_types: ['card'],
    line_items: [{
      price_data: {
        currency,
        product_data: { name: description || `Order ${orderId}` },
        unit_amount: Math.round(amount * 100), // Stripe uses cents
      },
      quantity: 1,
    }],
    mode: 'payment',
    success_url: `${process.env.SITE_URL}/order-success?orderId=${orderId}`,
    cancel_url: `${process.env.SITE_URL}/cart`,
    metadata: { orderId, userId },
  });

  res.json({ sessionId: session.id, url: session.url });
}

// ============================================================
// STRIPE WEBHOOK — POST /api/pay/webhook/stripe
// ============================================================
app.post('/webhook/stripe', async (req, res) => {
  const stripe = (await import('stripe')).default(process.env.STRIPE_SECRET_KEY);
  const sig = req.headers['stripe-signature'];

  let event;
  try {
    event = stripe.webhooks.constructEvent(req.body, sig, process.env.STRIPE_WEBHOOK_SECRET);
  } catch (err) {
    console.error('Webhook signature verification failed:', err.message);
    return res.status(400).send(`Webhook Error: ${err.message}`);
  }

  switch (event.type) {
    case 'checkout.session.completed': {
      const session = event.data.object;
      const { orderId } = session.metadata;
      console.log(`✅ Payment succeeded for order: ${orderId}`);

      // Update order status: pending → paid
      // In production: call commerce API or update database directly
      await updateOrderStatus(orderId, 'paid');
      break;
    }
    case 'payment_intent.payment_failed': {
      const intent = event.data.object;
      console.log(`❌ Payment failed: ${intent.id}`);
      break;
    }
  }

  res.json({ received: true });
});

// ============================================================
// ALIPAY STUB — Placeholder for China payments
// ============================================================
async function handleAlipayCheckout(req, res, orderId, userId) {
  // Alipay integration requires alipay-sdk npm package
  // This is a stub showing the expected flow
  res.json({
    method: 'alipay',
    orderId,
    message: 'Alipay integration stub — configure ALIPAY_APP_ID and ALIPAY_PRIVATE_KEY in env',
    // In production: return a payment URL or QR code data
    redirectUrl: null,
  });
}

// ============================================================
// WECHAT PAY STUB — Placeholder for WeChat payments
// ============================================================
async function handleWechatPayCheckout(req, res, orderId, userId) {
  // WechatPay integration requires wechatpay-node-v3 npm package
  // This is a stub showing the expected flow
  res.json({
    method: 'wechatpay',
    orderId,
    message: 'WechatPay integration stub — configure WECHAT_MCH_ID and WECHAT_API_KEY in env',
    // In production: return prepay_id for JSAPI or QR code for Native
    prepayId: null,
  });
}

// ============================================================
// PAYMENT STATUS — GET /api/pay/status/:orderId
// ============================================================
app.get('/status/:orderId', (req, res) => {
  const userId = getUserId(req);
  if (!userId) return res.status(401).json({ error: 'Unauthorized' });

  // In production: look up payment record in database
  res.json({
    orderId: req.params.orderId,
    status: 'pending', // Would be 'paid', 'failed', etc. from database
    method: null,
    paidAt: null,
  });
});

// ============================================================
// REFUND — POST /api/pay/refund
// ============================================================
app.post('/refund', async (req, res) => {
  const userId = getUserId(req);
  if (!userId) return res.status(401).json({ error: 'Unauthorized' });

  const { orderId, reason } = req.body;
  // Admin check would go here (role === 'admin')

  try {
    // In production: look up original payment, call stripe.refunds.create()
    console.log(`Refund requested for order ${orderId}: ${reason}`);
    res.json({ orderId, refundStatus: 'processing', reason });
  } catch (err) {
    res.status(500).json({ error: 'Refund failed' });
  }
});

// ============================================================
// HELPERS
// ============================================================

function getUserId(req) {
  try {
    const token = req.headers['x-auth-token'];
    if (!token) return null;
    const payload = JSON.parse(atob(token.split('.')[1]));
    return payload.sub;
  } catch { return null; }
}

async function updateOrderStatus(orderId, status) {
  // In production: call internal commerce API or update database
  // Example: await fetch('http://localhost:8088/api/commerce/orders/' + orderId + '/status', {
  //   method: 'PATCH', headers: { 'Content-Type': 'application/json' },
  //   body: JSON.stringify({ status })
  // });
  console.log(`Order ${orderId} → ${status}`);
}

export default app;
```

## Supported Payment Methods

| Method | Status | npm Package | Required Env Vars |
|--------|--------|------------|-------------------|
| Stripe | ✅ Full | `stripe` | `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET` |
| Alipay | 🔲 Stub | `alipay-sdk` | `ALIPAY_APP_ID`, `ALIPAY_PRIVATE_KEY` |
| WechatPay | 🔲 Stub | `wechatpay-node-v3` | `WECHAT_MCH_ID`, `WECHAT_API_KEY`, `WECHAT_SERIAL_NO` |

## Webhook Security

- Stripe: signature verified via `stripe.webhooks.constructEvent`
- Raw body required for webhook endpoints (configured via `express.raw`)
- Always return 200 to acknowledge receipt, even on processing errors

## Payment Flow

```
User clicks "Pay" → POST /api/pay/checkout
                    → Stripe Checkout Session created
                    → User redirected to Stripe hosted page
                    → Payment completes
                    → Stripe sends webhook POST /api/pay/webhook/stripe
                    → Order status updated to "paid"
                    → User redirected to /order-success
```
