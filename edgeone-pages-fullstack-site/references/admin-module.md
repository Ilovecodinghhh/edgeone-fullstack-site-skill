# Admin Dashboard Module

Unified management panel — stats, user management, config hot-reload, Skill status. Running on **Node.js Cloud Functions** (Express) + **KV Storage** for config persistence.

## File: `cloud-functions/api/admin/[[default]].js`

```javascript
import express from 'express';

const app = express();
app.use(express.json());

// Admin auth middleware — check role
app.use((req, res, next) => {
  const user = getUser(req);
  if (!user || user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  req.adminUser = user;
  next();
});

// ============================================================
// DASHBOARD — GET /api/admin/stats
// ============================================================
app.get('/stats', async (req, res) => {
  // In production: aggregate from database
  const stats = {
    totalUsers: 1250,
    activeToday: 89,
    totalOrders: 3420,
    pendingOrders: 15,
    revenue: {
      today: 4580,
      week: 28900,
      month: 112000,
    },
    topProducts: [
      { name: 'Premium Headphones', sold: 234, revenue: 69966 },
      { name: 'Smart Watch', sold: 156, revenue: 62244 },
      { name: 'Mechanical Keyboard', sold: 198, revenue: 31482 },
    ],
    recentActivity: [
      { type: 'order', message: 'New order #3421 — $299', time: Date.now() - 300000 },
      { type: 'user', message: 'New user registered: alice@example.com', time: Date.now() - 600000 },
      { type: 'payment', message: 'Payment confirmed for order #3420', time: Date.now() - 900000 },
    ],
    generatedAt: Date.now(),
  };

  res.json(stats);
});

// ============================================================
// USER MANAGEMENT — /api/admin/users
// ============================================================
app.get('/users', (req, res) => {
  const { page = 1, limit = 20, role, search } = req.query;

  // In production: query database
  const users = [
    { id: 'u001', email: 'alice@example.com', name: 'Alice', role: 'user', createdAt: Date.now() - 86400000 * 30, orderCount: 5 },
    { id: 'u002', email: 'bob@example.com', name: 'Bob', role: 'user', createdAt: Date.now() - 86400000 * 15, orderCount: 12 },
    { id: 'u003', email: 'admin@example.com', name: 'Admin', role: 'admin', createdAt: Date.now() - 86400000 * 90, orderCount: 0 },
  ];

  res.json({ users, total: users.length, page: Number(page) });
});

app.patch('/users/:id/role', (req, res) => {
  const { role } = req.body;
  if (!['user', 'admin', 'editor'].includes(role)) {
    return res.status(400).json({ error: 'Invalid role' });
  }
  // In production: update database
  res.json({ userId: req.params.id, role, updated: true });
});

app.delete('/users/:id', (req, res) => {
  // In production: soft-delete in database
  res.json({ userId: req.params.id, deleted: true });
});

// ============================================================
// ORDER MANAGEMENT — /api/admin/orders
// ============================================================
app.get('/orders', (req, res) => {
  const { status, page = 1, limit = 20 } = req.query;
  // In production: query database with filters
  res.json({ orders: [], total: 0, page: Number(page) });
});

app.patch('/orders/:id/status', async (req, res) => {
  const { status } = req.body;
  // In production: call commerce module's status transition
  // await fetch('http://localhost:8088/api/commerce/orders/' + req.params.id + '/status', ...)
  res.json({ orderId: req.params.id, status, updated: true });
});

// ============================================================
// CONFIG CENTER — /api/admin/config (KV-backed hot reload)
// ============================================================

// Config is stored in KV for hot reload without redeploy
// Keys: config:site, config:payment, config:ai, config:modules

app.get('/config', async (req, res) => {
  // In production with KV:
  // const siteConfig = await config_kv.get('config:site', 'json');
  // const payConfig = await config_kv.get('config:payment', 'json');
  // const aiConfig = await config_kv.get('config:ai', 'json');

  const config = {
    site: {
      name: 'My Store',
      tagline: 'Best products online',
      logo: '/logo.svg',
      primaryColor: '#3B82F6',
      locale: 'zh-CN',
    },
    payment: {
      stripeEnabled: true,
      alipayEnabled: false,
      wechatPayEnabled: false,
      currency: 'USD',
    },
    ai: {
      enabled: true,
      model: 'gpt-4o-mini',
      systemPrompt: 'You are a helpful shopping assistant.',
      maxTokens: 500,
    },
    modules: {
      auth: true,
      commerce: true,
      pay: true,
      ai: true,
      admin: true,
    },
  };

  res.json(config);
});

app.patch('/config/:section', async (req, res) => {
  const { section } = req.params;
  const updates = req.body;

  const validSections = ['site', 'payment', 'ai', 'modules'];
  if (!validSections.includes(section)) {
    return res.status(400).json({ error: `Invalid section: ${section}` });
  }

  // In production with KV:
  // const existing = await config_kv.get(`config:${section}`, 'json') || {};
  // const merged = { ...existing, ...updates };
  // await config_kv.put(`config:${section}`, JSON.stringify(merged));

  console.log(`Config updated [${section}]:`, updates);

  res.json({
    section,
    updated: true,
    message: 'Config updated. Changes take effect immediately (no redeploy needed).',
  });
});

// ============================================================
// MODULE STATUS — GET /api/admin/modules
// ============================================================
app.get('/modules', (req, res) => {
  const modules = [
    { name: 'auth', status: 'active', version: '1.0.0', type: 'edge-function', endpoint: '/api/auth' },
    { name: 'commerce', status: 'active', version: '1.0.0', type: 'cloud-function', endpoint: '/api/commerce' },
    { name: 'pay', status: 'active', version: '1.0.0', type: 'cloud-function', endpoint: '/api/pay' },
    { name: 'ai', status: 'active', version: '1.0.0', type: 'cloud-function', endpoint: '/api/ai' },
    { name: 'admin', status: 'active', version: '1.0.0', type: 'cloud-function', endpoint: '/api/admin' },
  ];

  res.json({ modules });
});

// ============================================================
// LOGS / ACTIVITY — GET /api/admin/logs
// ============================================================
app.get('/logs', (req, res) => {
  const { type, limit = 50 } = req.query;
  // In production: read from logging service or database
  res.json({ logs: [], total: 0 });
});

// ============================================================
// HELPERS
// ============================================================

function getUser(req) {
  try {
    const token = req.headers['x-auth-token'];
    if (!token) return null;
    return JSON.parse(atob(token.split('.')[1]));
  } catch { return null; }
}

export default app;
```

## Config Hot-Reload Architecture

```
Admin UI → PATCH /api/admin/config/:section
              → Writes to KV (config_kv)
              → Immediately available to all edge/cloud functions
              → No redeploy needed

Other modules read config:
  Edge Functions → await config_kv.get('config:site', 'json')
  Cloud Functions → fetch('/api/admin/config') or direct KV read
```

## KV Key Design for Admin

```
config:site       → { name, tagline, logo, primaryColor, locale }
config:payment    → { stripeEnabled, alipayEnabled, currency }
config:ai         → { enabled, model, systemPrompt, maxTokens }
config:modules    → { auth: true, commerce: true, ... }
stats:daily:YYYY-MM-DD → { orders, revenue, newUsers }
```

## Admin Routes Summary

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/admin/stats` | Dashboard statistics |
| GET | `/api/admin/users` | List users (paginated) |
| PATCH | `/api/admin/users/:id/role` | Change user role |
| DELETE | `/api/admin/users/:id` | Delete user |
| GET | `/api/admin/orders` | List orders with filters |
| PATCH | `/api/admin/orders/:id/status` | Update order status |
| GET | `/api/admin/config` | Get all config sections |
| PATCH | `/api/admin/config/:section` | Update config section |
| GET | `/api/admin/modules` | Module health/status |
| GET | `/api/admin/logs` | Activity logs |
