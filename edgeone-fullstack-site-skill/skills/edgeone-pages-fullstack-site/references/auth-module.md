# Auth Module

JWT-based authentication running on **Edge Functions** (V8 runtime) + **KV Storage** for session persistence + **Middleware** for route protection.

## Why Edge Functions for Auth?

- Ultra-low latency (~5ms) — auth check runs at the edge before hitting origin
- KV Storage available for session/token blacklisting
- No npm needed — Web Crypto API handles JWT signing

## KV Prerequisites

⚠️ User must enable KV in the EdgeOne Pages console and bind a namespace with variable name `auth_kv`.

Key design:
```
session:<user_id>         → JSON { token, refreshToken, expiresAt }
user:<email>              → JSON { id, email, passwordHash, role, createdAt }
user_id:<id>              → JSON (same as above, indexed by id)
rate_limit:<ip>           → number (login attempt count)
config:auth               → JSON { jwtSecret hint, oauthProviders }
```

## Login — `edge-functions/api/auth/login.js`

```javascript
// Edge Function — V8 runtime, NO npm
export async function onRequestPost(context) {
  try {
    const { email, password } = await context.request.json();
    if (!email || !password) {
      return new Response(JSON.stringify({ error: 'Email and password required' }), {
        status: 400, headers: { 'Content-Type': 'application/json' }
      });
    }

    // Rate limiting check
    const ip = context.request.headers.get('CF-Connecting-IP') || 'unknown';
    const attempts = parseInt(await auth_kv.get(`rate_limit:${ip}`) || '0');
    if (attempts > 10) {
      return new Response(JSON.stringify({ error: 'Too many attempts. Try later.' }), {
        status: 429, headers: { 'Content-Type': 'application/json' }
      });
    }

    // Lookup user
    const userData = await auth_kv.get(`user:${email}`, 'json');
    if (!userData) {
      await auth_kv.put(`rate_limit:${ip}`, String(attempts + 1));
      return new Response(JSON.stringify({ error: 'Invalid credentials' }), {
        status: 401, headers: { 'Content-Type': 'application/json' }
      });
    }

    // Verify password (Web Crypto PBKDF2)
    const valid = await verifyPassword(password, userData.passwordHash, userData.salt);
    if (!valid) {
      await auth_kv.put(`rate_limit:${ip}`, String(attempts + 1));
      return new Response(JSON.stringify({ error: 'Invalid credentials' }), {
        status: 401, headers: { 'Content-Type': 'application/json' }
      });
    }

    // Generate JWT
    const secret = context.env.JWT_SECRET;
    const token = await signJWT({ sub: userData.id, email, role: userData.role }, secret, '2h');
    const refreshToken = await signJWT({ sub: userData.id, type: 'refresh' }, secret, '7d');

    // Store session in KV
    await auth_kv.put(`session:${userData.id}`, JSON.stringify({
      token, refreshToken, expiresAt: Date.now() + 7200000
    }));

    // Clear rate limit
    await auth_kv.delete(`rate_limit:${ip}`);

    return new Response(JSON.stringify({
      token, refreshToken,
      user: { id: userData.id, email, role: userData.role }
    }), {
      status: 200, headers: { 'Content-Type': 'application/json' }
    });
  } catch (e) {
    return new Response(JSON.stringify({ error: 'Internal error' }), {
      status: 500, headers: { 'Content-Type': 'application/json' }
    });
  }
}

// --- Crypto helpers (Web Crypto API, NOT Node.js crypto) ---

async function hashPassword(password, salt) {
  const enc = new TextEncoder();
  const keyMaterial = await crypto.subtle.importKey(
    'raw', enc.encode(password), 'PBKDF2', false, ['deriveBits']
  );
  const bits = await crypto.subtle.deriveBits(
    { name: 'PBKDF2', salt: enc.encode(salt), iterations: 100000, hash: 'SHA-256' },
    keyMaterial, 256
  );
  return btoa(String.fromCharCode(...new Uint8Array(bits)));
}

async function verifyPassword(password, storedHash, salt) {
  const hash = await hashPassword(password, salt);
  return hash === storedHash;
}

async function signJWT(payload, secret, expiresIn) {
  const header = { alg: 'HS256', typ: 'JWT' };
  const now = Math.floor(Date.now() / 1000);
  const expSeconds = expiresIn.endsWith('h')
    ? parseInt(expiresIn) * 3600
    : parseInt(expiresIn) * 86400;

  const fullPayload = { ...payload, iat: now, exp: now + expSeconds };
  const enc = new TextEncoder();

  const headerB64 = btoa(JSON.stringify(header)).replace(/=/g, '');
  const payloadB64 = btoa(JSON.stringify(fullPayload)).replace(/=/g, '');
  const signingInput = `${headerB64}.${payloadB64}`;

  const key = await crypto.subtle.importKey(
    'raw', enc.encode(secret), { name: 'HMAC', hash: 'SHA-256' }, false, ['sign']
  );
  const sig = await crypto.subtle.sign('HMAC', key, enc.encode(signingInput));
  const sigB64 = btoa(String.fromCharCode(...new Uint8Array(sig))).replace(/=/g, '');

  return `${signingInput}.${sigB64}`;
}
```

## Register — `edge-functions/api/auth/register.js`

```javascript
export async function onRequestPost(context) {
  const { email, password, name } = await context.request.json();
  if (!email || !password) {
    return new Response(JSON.stringify({ error: 'Email and password required' }), {
      status: 400, headers: { 'Content-Type': 'application/json' }
    });
  }

  // Check existing
  const existing = await auth_kv.get(`user:${email}`);
  if (existing) {
    return new Response(JSON.stringify({ error: 'Email already registered' }), {
      status: 409, headers: { 'Content-Type': 'application/json' }
    });
  }

  // Hash password
  const salt = crypto.randomUUID();
  const passwordHash = await hashPassword(password, salt);
  const id = crypto.randomUUID();

  const user = { id, email, name: name || '', passwordHash, salt, role: 'user', createdAt: Date.now() };
  await auth_kv.put(`user:${email}`, JSON.stringify(user));
  await auth_kv.put(`user_id:${id}`, JSON.stringify(user));

  return new Response(JSON.stringify({ user: { id, email, name: user.name, role: 'user' } }), {
    status: 201, headers: { 'Content-Type': 'application/json' }
  });
}

// hashPassword — same as login.js (duplicate in each edge function since no shared modules)
async function hashPassword(password, salt) {
  const enc = new TextEncoder();
  const keyMaterial = await crypto.subtle.importKey(
    'raw', enc.encode(password), 'PBKDF2', false, ['deriveBits']
  );
  const bits = await crypto.subtle.deriveBits(
    { name: 'PBKDF2', salt: enc.encode(salt), iterations: 100000, hash: 'SHA-256' },
    keyMaterial, 256
  );
  return btoa(String.fromCharCode(...new Uint8Array(bits)));
}
```

## Verify Token — `edge-functions/api/auth/verify.js`

```javascript
export async function onRequestGet(context) {
  const authHeader = context.request.headers.get('Authorization');
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return new Response(JSON.stringify({ valid: false }), {
      status: 401, headers: { 'Content-Type': 'application/json' }
    });
  }

  const token = authHeader.slice(7);
  const payload = await verifyJWT(token, context.env.JWT_SECRET);
  if (!payload) {
    return new Response(JSON.stringify({ valid: false }), {
      status: 401, headers: { 'Content-Type': 'application/json' }
    });
  }

  return new Response(JSON.stringify({ valid: true, user: payload }), {
    headers: { 'Content-Type': 'application/json' }
  });
}

async function verifyJWT(token, secret) {
  try {
    const [headerB64, payloadB64, sigB64] = token.split('.');
    const enc = new TextEncoder();
    const key = await crypto.subtle.importKey(
      'raw', enc.encode(secret), { name: 'HMAC', hash: 'SHA-256' }, false, ['verify']
    );

    const sigBytes = Uint8Array.from(atob(sigB64), c => c.charCodeAt(0));
    const valid = await crypto.subtle.verify('HMAC', key, sigBytes, enc.encode(`${headerB64}.${payloadB64}`));
    if (!valid) return null;

    const payload = JSON.parse(atob(payloadB64));
    if (payload.exp && payload.exp < Math.floor(Date.now() / 1000)) return null;
    return payload;
  } catch {
    return null;
  }
}
```

## Middleware — `middleware.js` (project root)

```javascript
export const config = {
  matcher: ['/api/commerce/:path*', '/api/pay/:path*', '/api/admin/:path*', '/api/ai/:path*'],
};

export async function middleware(context) {
  const { request, next } = context;

  // CORS
  if (request.method === 'OPTIONS') {
    return new Response(null, {
      headers: {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'GET,POST,PUT,DELETE,OPTIONS',
        'Access-Control-Allow-Headers': 'Content-Type,Authorization',
      },
    });
  }

  // Skip auth for public endpoints
  const url = new URL(request.url);
  const publicPaths = ['/api/auth/', '/api/commerce/products'];
  if (publicPaths.some(p => url.pathname.startsWith(p))) {
    return next();
  }

  // Verify JWT
  const authHeader = request.headers.get('Authorization');
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return new Response(JSON.stringify({ error: 'Unauthorized' }), {
      status: 401, headers: { 'Content-Type': 'application/json' }
    });
  }

  // Pass user info downstream via header
  return next({
    headers: { 'x-auth-token': authHeader.slice(7) },
  });
}
```

## OAuth Stub — `edge-functions/api/auth/oauth/[provider].js`

```javascript
export async function onRequestGet(context) {
  const provider = context.params.provider; // github, wechat, google
  const redirectUrls = {
    github: `https://github.com/login/oauth/authorize?client_id=${context.env.GITHUB_CLIENT_ID}&scope=user:email`,
    google: `https://accounts.google.com/o/oauth2/v2/auth?client_id=${context.env.GOOGLE_CLIENT_ID}&response_type=code&scope=email+profile&redirect_uri=${encodeURIComponent(context.env.OAUTH_REDIRECT_URL)}`,
  };

  const url = redirectUrls[provider];
  if (!url) {
    return new Response(JSON.stringify({ error: 'Unsupported provider' }), {
      status: 400, headers: { 'Content-Type': 'application/json' }
    });
  }

  return Response.redirect(url, 302);
}
```

## Security Notes

- JWT secret must be ≥ 32 characters, stored in `JWT_SECRET` env var
- Password hashing uses PBKDF2 with 100k iterations (Web Crypto)
- Rate limiting via KV: 10 attempts per IP, auto-reset on success
- Refresh tokens enable silent re-auth without re-login
- Admin role check: verify `payload.role === 'admin'` in admin routes
