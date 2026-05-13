# Data Schema

Unified data model across all modules. KV key design for Edge Functions; JSON contracts for API communication.

## KV Key Namespace Design

All KV keys use prefixed namespaces for isolation and queryability.

### auth_kv (Auth module)

| Key Pattern | Value Type | Description |
|-------------|-----------|-------------|
| `user:<email>` | JSON | User record (by email lookup) |
| `user_id:<uuid>` | JSON | User record (by ID lookup) |
| `session:<userId>` | JSON | Active session { token, refreshToken, expiresAt } |
| `rate_limit:<ip>` | string (number) | Login attempt count per IP |

### config_kv (Admin module)

| Key Pattern | Value Type | Description |
|-------------|-----------|-------------|
| `config:site` | JSON | Site name, tagline, logo, color, locale |
| `config:payment` | JSON | Payment provider toggles, currency |
| `config:ai` | JSON | LLM model, system prompt, max tokens |
| `config:modules` | JSON | Module enable/disable flags |
| `stats:daily:YYYY-MM-DD` | JSON | Daily aggregated stats |

## API Contract — Standard Response Format

All APIs return:

```typescript
// Success
{ data: T, message?: string }

// Error
{ error: string, code?: string }

// Paginated
{ items: T[], total: number, page: number, totalPages: number }
```

## TypeScript Types (`src/lib/types.ts`)

```typescript
// === Auth ===
export interface User {
  id: string;
  email: string;
  name: string;
  role: 'user' | 'admin' | 'editor';
  createdAt: number;
}

export interface AuthResponse {
  token: string;
  refreshToken: string;
  user: Pick<User, 'id' | 'email' | 'role'>;
}

// === Commerce ===
export interface Product {
  id: string;
  name: string;
  price: number;
  description: string;
  category: string;
  stock: number;
  image?: string;
}

export interface CartItem {
  productId: string;
  name: string;
  price: number;
  quantity: number;
  image?: string;
}

export interface Cart {
  items: CartItem[];
  total: number;
  itemCount: number;
}

export type OrderStatus = 'pending' | 'paid' | 'shipped' | 'delivered' | 'completed' | 'cancelled' | 'refunded';

export interface Order {
  id: string;
  userId: string;
  items: CartItem[];
  total: number;
  status: OrderStatus;
  shippingAddress?: string;
  createdAt: number;
  updatedAt: number;
}

// === Payment ===
export interface CheckoutRequest {
  orderId: string;
  method: 'stripe' | 'alipay' | 'wechatpay';
  amount?: number;
  currency?: string;
  description?: string;
}

export interface CheckoutResponse {
  sessionId?: string;
  url?: string;
  method: string;
}

// === AI ===
export interface ChatMessage {
  role: 'user' | 'assistant' | 'system';
  content: string;
}

export interface ChatResponse {
  reply: string;
  action: string | null;
  data: Record<string, any> | null;
  conversationId: string;
}

// === Admin ===
export interface DashboardStats {
  totalUsers: number;
  activeToday: number;
  totalOrders: number;
  pendingOrders: number;
  revenue: { today: number; week: number; month: number };
  topProducts: { name: string; sold: number; revenue: number }[];
  recentActivity: { type: string; message: string; time: number }[];
}

export interface SiteConfig {
  site: { name: string; tagline: string; logo: string; primaryColor: string; locale: string };
  payment: { stripeEnabled: boolean; alipayEnabled: boolean; wechatPayEnabled: boolean; currency: string };
  ai: { enabled: boolean; model: string; systemPrompt: string; maxTokens: number };
  modules: Record<string, boolean>;
}
```

## Order State Machine (canonical)

```
VALID_TRANSITIONS = {
  pending:   ['paid', 'cancelled'],
  paid:      ['shipped', 'refunded'],
  shipped:   ['delivered', 'refunded'],
  delivered: ['completed', 'refunded'],
  completed: [],
  cancelled: [],
  refunded:  [],
}
```

This state machine is enforced server-side in the Commerce module. The Pay module triggers `pending → paid` via webhook. Admin can trigger any valid transition.
