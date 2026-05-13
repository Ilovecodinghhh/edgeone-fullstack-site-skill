# Frontend Module

React 18 + TypeScript + Tailwind CSS + React Router. Vite-based SPA with pages for each enabled module.

## Core Files (always generated)

### `index.html`

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>My Site</title>
  <link rel="icon" href="/favicon.ico" />
</head>
<body>
  <div id="root"></div>
  <script type="module" src="/src/main.tsx"></script>
</body>
</html>
```

### `src/main.tsx`

```tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import App from './App';
import './index.css';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </React.StrictMode>
);
```

### `src/index.css`

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

:root {
  --primary: #3B82F6;
  --primary-dark: #2563EB;
  --accent: #10B981;
}
```

### `src/App.tsx` (full modules)

```tsx
import { Routes, Route } from 'react-router-dom';
import Layout from './components/Layout';
import Home from './pages/Home';
// Auth
import Login from './pages/Login';
import Register from './pages/Register';
// Commerce
import Shop from './pages/Shop';
import ProductDetail from './pages/ProductDetail';
import Cart from './pages/Cart';
// Pay
import Checkout from './pages/Checkout';
import OrderSuccess from './pages/OrderSuccess';
// AI
import Chat from './pages/ai/Chat';
// Admin
import Dashboard from './pages/admin/Dashboard';
import AdminProducts from './pages/admin/Products';
import AdminOrders from './pages/admin/Orders';
import AdminUsers from './pages/admin/Users';
import AdminSettings from './pages/admin/Settings';

export default function App() {
  return (
    <Layout>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/login" element={<Login />} />
        <Route path="/register" element={<Register />} />
        <Route path="/shop" element={<Shop />} />
        <Route path="/product/:id" element={<ProductDetail />} />
        <Route path="/cart" element={<Cart />} />
        <Route path="/checkout" element={<Checkout />} />
        <Route path="/order-success" element={<OrderSuccess />} />
        <Route path="/chat" element={<Chat />} />
        <Route path="/admin" element={<Dashboard />} />
        <Route path="/admin/products" element={<AdminProducts />} />
        <Route path="/admin/orders" element={<AdminOrders />} />
        <Route path="/admin/users" element={<AdminUsers />} />
        <Route path="/admin/settings" element={<AdminSettings />} />
      </Routes>
    </Layout>
  );
}
```

Only include routes for enabled modules. Remove unused imports.

### `src/lib/api.ts`

```typescript
const BASE = '';

export async function api<T>(path: string, options: RequestInit = {}): Promise<T> {
  const token = localStorage.getItem('token');
  const res = await fetch(`${BASE}${path}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      ...options.headers,
    },
  });

  if (res.status === 401) {
    localStorage.removeItem('token');
    window.location.href = '/login';
    throw new Error('Unauthorized');
  }

  if (!res.ok) {
    const err = await res.json().catch(() => ({ error: 'Request failed' }));
    throw new Error(err.error || `HTTP ${res.status}`);
  }

  return res.json();
}
```

### `src/lib/auth.ts`

```typescript
import { api } from './api';

interface AuthResponse {
  token: string;
  refreshToken: string;
  user: { id: string; email: string; role: string };
}

export async function login(email: string, password: string): Promise<AuthResponse> {
  const data = await api<AuthResponse>('/api/auth/login', {
    method: 'POST',
    body: JSON.stringify({ email, password }),
  });
  localStorage.setItem('token', data.token);
  localStorage.setItem('user', JSON.stringify(data.user));
  return data;
}

export async function register(email: string, password: string, name?: string) {
  return api('/api/auth/register', {
    method: 'POST',
    body: JSON.stringify({ email, password, name }),
  });
}

export function logout() {
  localStorage.removeItem('token');
  localStorage.removeItem('user');
  window.location.href = '/';
}

export function getUser() {
  const raw = localStorage.getItem('user');
  return raw ? JSON.parse(raw) : null;
}

export function isAdmin() {
  return getUser()?.role === 'admin';
}
```

## Page Templates

### `src/pages/Home.tsx`

```tsx
import { Link } from 'react-router-dom';

export default function Home() {
  return (
    <div className="min-h-screen">
      <section className="bg-gradient-to-r from-blue-600 to-purple-600 text-white py-20 text-center">
        <h1 className="text-5xl font-bold mb-4">Welcome to Our Store</h1>
        <p className="text-xl mb-8 opacity-90">Premium products, AI-powered experience</p>
        <Link to="/shop" className="bg-white text-blue-600 px-8 py-3 rounded-lg font-semibold hover:bg-gray-100 transition">
          Start Shopping →
        </Link>
      </section>
    </div>
  );
}
```

### `src/pages/Login.tsx`

```tsx
import { useState } from 'react';
import { useNavigate, Link } from 'react-router-dom';
import { login } from '../lib/auth';

export default function Login() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);
  const navigate = useNavigate();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);
    setError('');
    try {
      await login(email, password);
      navigate('/');
    } catch (err: any) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-50">
      <form onSubmit={handleSubmit} className="bg-white p-8 rounded-xl shadow-lg w-full max-w-md">
        <h2 className="text-2xl font-bold mb-6 text-center">登录</h2>
        {error && <div className="bg-red-50 text-red-600 p-3 rounded mb-4">{error}</div>}
        <input type="email" value={email} onChange={e => setEmail(e.target.value)} placeholder="Email" required
          className="w-full p-3 border rounded-lg mb-4 focus:ring-2 focus:ring-blue-500 outline-none" />
        <input type="password" value={password} onChange={e => setPassword(e.target.value)} placeholder="Password" required
          className="w-full p-3 border rounded-lg mb-6 focus:ring-2 focus:ring-blue-500 outline-none" />
        <button type="submit" disabled={loading}
          className="w-full bg-blue-600 text-white py-3 rounded-lg font-semibold hover:bg-blue-700 disabled:opacity-50 transition">
          {loading ? '登录中...' : '登录'}
        </button>
        <p className="text-center mt-4 text-gray-600">
          没有账号？<Link to="/register" className="text-blue-600 hover:underline">注册</Link>
        </p>
      </form>
    </div>
  );
}
```

### `src/components/ChatWidget.tsx` (floating AI assistant)

```tsx
import { useState } from 'react';
import { api } from '../lib/api';

export default function ChatWidget() {
  const [open, setOpen] = useState(false);
  const [messages, setMessages] = useState<{ role: string; content: string }[]>([]);
  const [input, setInput] = useState('');
  const [loading, setLoading] = useState(false);

  const send = async () => {
    if (!input.trim()) return;
    const userMsg = { role: 'user', content: input };
    setMessages(prev => [...prev, userMsg]);
    setInput('');
    setLoading(true);

    try {
      const data = await api<{ reply: string }>('/api/ai/chat', {
        method: 'POST',
        body: JSON.stringify({ message: input }),
      });
      setMessages(prev => [...prev, { role: 'assistant', content: data.reply }]);
    } catch {
      setMessages(prev => [...prev, { role: 'assistant', content: '抱歉，AI 服务暂时不可用。' }]);
    } finally {
      setLoading(false);
    }
  };

  return (
    <>
      <button onClick={() => setOpen(!open)}
        className="fixed bottom-6 right-6 w-14 h-14 bg-blue-600 text-white rounded-full shadow-lg flex items-center justify-center text-2xl hover:bg-blue-700 z-50">
        {open ? '✕' : '💬'}
      </button>
      {open && (
        <div className="fixed bottom-24 right-6 w-96 h-[500px] bg-white rounded-xl shadow-2xl flex flex-col z-50 border">
          <div className="bg-blue-600 text-white p-4 rounded-t-xl font-semibold">AI 助手</div>
          <div className="flex-1 overflow-y-auto p-4 space-y-3">
            {messages.map((m, i) => (
              <div key={i} className={`${m.role === 'user' ? 'text-right' : ''}`}>
                <span className={`inline-block p-3 rounded-lg max-w-[80%] ${
                  m.role === 'user' ? 'bg-blue-100 text-blue-900' : 'bg-gray-100 text-gray-900'
                }`}>{m.content}</span>
              </div>
            ))}
            {loading && <div className="text-gray-400">思考中...</div>}
          </div>
          <div className="p-3 border-t flex gap-2">
            <input value={input} onChange={e => setInput(e.target.value)}
              onKeyDown={e => e.key === 'Enter' && send()}
              placeholder="问我任何问题..." className="flex-1 p-2 border rounded-lg outline-none focus:ring-2 focus:ring-blue-500" />
            <button onClick={send} className="bg-blue-600 text-white px-4 rounded-lg hover:bg-blue-700">发送</button>
          </div>
        </div>
      )}
    </>
  );
}
```

### `src/components/Layout.tsx`

```tsx
import { Link } from 'react-router-dom';
import { getUser, logout, isAdmin } from '../lib/auth';
import ChatWidget from './ChatWidget'; // Only if AI module enabled

export default function Layout({ children }: { children: React.ReactNode }) {
  const user = getUser();

  return (
    <div className="min-h-screen bg-gray-50">
      <nav className="bg-white shadow-sm border-b">
        <div className="max-w-7xl mx-auto px-4 py-3 flex items-center justify-between">
          <Link to="/" className="text-xl font-bold text-blue-600">🏪 My Store</Link>
          <div className="flex items-center gap-6">
            <Link to="/shop" className="text-gray-700 hover:text-blue-600">商品</Link>
            <Link to="/cart" className="text-gray-700 hover:text-blue-600">🛒 购物车</Link>
            {user ? (
              <>
                {isAdmin() && <Link to="/admin" className="text-gray-700 hover:text-blue-600">管理后台</Link>}
                <button onClick={logout} className="text-gray-500 hover:text-red-500">退出</button>
              </>
            ) : (
              <Link to="/login" className="bg-blue-600 text-white px-4 py-2 rounded-lg hover:bg-blue-700">登录</Link>
            )}
          </div>
        </div>
      </nav>
      <main>{children}</main>
      <ChatWidget />
    </div>
  );
}
```

## Conditional Generation Rules

- If Auth disabled: remove Login/Register pages, remove auth checks from Layout, remove token logic from api.ts
- If Commerce disabled: remove Shop, ProductDetail, Cart pages and nav links
- If Pay disabled: remove Checkout, OrderSuccess pages
- If AI disabled: remove ChatWidget, Chat page
- If Admin disabled: remove admin/* pages and admin nav link

Adjust `App.tsx` routes and `Layout.tsx` nav accordingly.
