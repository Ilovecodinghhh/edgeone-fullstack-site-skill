# Commerce Module

Shopping cart, product catalog, inventory, and order management — running on **Node.js Cloud Functions** with Express.

## File: `cloud-functions/api/commerce/[[default]].js`

Single Express app handling all `/api/commerce/*` routes.

```javascript
import express from 'express';

const app = express();
app.use(express.json());

// --- In-memory store (replace with database in production) ---
// For demo, products stored in KV via admin; orders in memory
// Production: use MySQL/PostgreSQL via cloud-functions npm packages

const carts = new Map();    // userId → CartItem[]
const orders = new Map();   // orderId → Order

// ============================================================
// PRODUCTS — GET /api/commerce/products
// ============================================================
app.get('/products', async (req, res) => {
  // Products loaded from env or external API
  // In production, fetch from database
  const products = getProductCatalog();
  const { category, search, page = 1, limit = 20 } = req.query;

  let filtered = products;
  if (category) filtered = filtered.filter(p => p.category === category);
  if (search) filtered = filtered.filter(p =>
    p.name.toLowerCase().includes(search.toLowerCase()) ||
    p.description.toLowerCase().includes(search.toLowerCase())
  );

  const start = (page - 1) * limit;
  const paginated = filtered.slice(start, start + Number(limit));

  res.json({
    products: paginated,
    total: filtered.length,
    page: Number(page),
    totalPages: Math.ceil(filtered.length / limit)
  });
});

// GET /api/commerce/products/:id
app.get('/products/:id', (req, res) => {
  const product = getProductCatalog().find(p => p.id === req.params.id);
  if (!product) return res.status(404).json({ error: 'Product not found' });
  res.json(product);
});

// ============================================================
// CART — /api/commerce/cart
// ============================================================
app.get('/cart', (req, res) => {
  const userId = getUserId(req);
  if (!userId) return res.status(401).json({ error: 'Unauthorized' });

  const cart = carts.get(userId) || [];
  const total = calculateCartTotal(cart);
  res.json({ items: cart, total, itemCount: cart.length });
});

app.post('/cart/add', (req, res) => {
  const userId = getUserId(req);
  if (!userId) return res.status(401).json({ error: 'Unauthorized' });

  const { productId, quantity = 1 } = req.body;
  const product = getProductCatalog().find(p => p.id === productId);
  if (!product) return res.status(404).json({ error: 'Product not found' });
  if (product.stock < quantity) return res.status(400).json({ error: 'Insufficient stock' });

  const cart = carts.get(userId) || [];
  const existing = cart.find(item => item.productId === productId);
  if (existing) {
    existing.quantity += quantity;
  } else {
    cart.push({ productId, name: product.name, price: product.price, quantity, image: product.image });
  }
  carts.set(userId, cart);

  res.json({ items: cart, total: calculateCartTotal(cart) });
});

app.post('/cart/update', (req, res) => {
  const userId = getUserId(req);
  if (!userId) return res.status(401).json({ error: 'Unauthorized' });

  const { productId, quantity } = req.body;
  const cart = carts.get(userId) || [];

  if (quantity <= 0) {
    carts.set(userId, cart.filter(item => item.productId !== productId));
  } else {
    const item = cart.find(i => i.productId === productId);
    if (item) item.quantity = quantity;
  }

  const updated = carts.get(userId) || [];
  res.json({ items: updated, total: calculateCartTotal(updated) });
});

app.delete('/cart/clear', (req, res) => {
  const userId = getUserId(req);
  if (!userId) return res.status(401).json({ error: 'Unauthorized' });

  carts.delete(userId);
  res.json({ items: [], total: 0 });
});

// ============================================================
// ORDERS — /api/commerce/orders
// ============================================================

// Order state machine: pending → paid → shipped → delivered → completed
//                                  └→ cancelled   └→ refunded
const VALID_TRANSITIONS = {
  pending:   ['paid', 'cancelled'],
  paid:      ['shipped', 'refunded'],
  shipped:   ['delivered', 'refunded'],
  delivered: ['completed', 'refunded'],
  completed: [],
  cancelled: [],
  refunded:  [],
};

app.post('/orders', (req, res) => {
  const userId = getUserId(req);
  if (!userId) return res.status(401).json({ error: 'Unauthorized' });

  const cart = carts.get(userId);
  if (!cart || cart.length === 0) return res.status(400).json({ error: 'Cart is empty' });

  const { shippingAddress } = req.body;
  const total = calculateCartTotal(cart);

  const order = {
    id: `order_${Date.now()}_${Math.random().toString(36).slice(2, 8)}`,
    userId,
    items: [...cart],
    total,
    status: 'pending',
    shippingAddress: shippingAddress || null,
    createdAt: Date.now(),
    updatedAt: Date.now(),
  };

  orders.set(order.id, order);
  carts.delete(userId); // Clear cart after order

  res.status(201).json(order);
});

app.get('/orders', (req, res) => {
  const userId = getUserId(req);
  if (!userId) return res.status(401).json({ error: 'Unauthorized' });

  const userOrders = [...orders.values()]
    .filter(o => o.userId === userId)
    .sort((a, b) => b.createdAt - a.createdAt);

  res.json({ orders: userOrders });
});

app.get('/orders/:id', (req, res) => {
  const userId = getUserId(req);
  const order = orders.get(req.params.id);
  if (!order || order.userId !== userId) return res.status(404).json({ error: 'Order not found' });
  res.json(order);
});

// Status transition (called by pay webhook or admin)
app.patch('/orders/:id/status', (req, res) => {
  const order = orders.get(req.params.id);
  if (!order) return res.status(404).json({ error: 'Order not found' });

  const { status } = req.body;
  const allowed = VALID_TRANSITIONS[order.status] || [];
  if (!allowed.includes(status)) {
    return res.status(400).json({
      error: `Cannot transition from ${order.status} to ${status}`,
      allowed
    });
  }

  order.status = status;
  order.updatedAt = Date.now();
  res.json(order);
});

// ============================================================
// HELPERS
// ============================================================

function getUserId(req) {
  // Extracted from JWT by middleware, passed via header
  try {
    const token = req.headers['x-auth-token'];
    if (!token) return null;
    const payload = JSON.parse(atob(token.split('.')[1]));
    return payload.sub;
  } catch { return null; }
}

function calculateCartTotal(cart) {
  return cart.reduce((sum, item) => sum + item.price * item.quantity, 0);
}

function getProductCatalog() {
  // Demo catalog — in production, load from database
  return [
    { id: 'prod_001', name: 'Premium Headphones', price: 299, category: 'electronics', stock: 50, image: '/images/headphones.jpg', description: 'High-fidelity wireless headphones' },
    { id: 'prod_002', name: 'Mechanical Keyboard', price: 159, category: 'electronics', stock: 30, image: '/images/keyboard.jpg', description: 'RGB mechanical keyboard' },
    { id: 'prod_003', name: 'Smart Watch', price: 399, category: 'electronics', stock: 20, image: '/images/watch.jpg', description: 'Health tracking smartwatch' },
    { id: 'prod_004', name: 'Canvas Backpack', price: 89, category: 'accessories', stock: 100, image: '/images/backpack.jpg', description: 'Durable canvas travel backpack' },
    { id: 'prod_005', name: 'Desk Lamp', price: 49, category: 'home', stock: 80, image: '/images/lamp.jpg', description: 'LED desk lamp with USB charging' },
  ];
}

export default app;
```

## Data Contracts

### CartItem
```typescript
interface CartItem {
  productId: string;
  name: string;
  price: number;
  quantity: number;
  image?: string;
}
```

### Order
```typescript
interface Order {
  id: string;
  userId: string;
  items: CartItem[];
  total: number;
  status: 'pending' | 'paid' | 'shipped' | 'delivered' | 'completed' | 'cancelled' | 'refunded';
  shippingAddress?: string;
  createdAt: number;
  updatedAt: number;
}
```

### Order State Machine

```
pending ──→ paid ──→ shipped ──→ delivered ──→ completed
  │          │         │
  └→ cancelled └→ refunded └→ refunded
```

Transition validation is enforced server-side. Invalid transitions return 400.

## Production Notes

- Replace in-memory Maps with a database (MySQL via `mysql2` npm package)
- Product catalog should be managed via Admin module
- Inventory deduction should happen atomically when order is created
- Consider adding coupon/discount logic in `calculateCartTotal`
