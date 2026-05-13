# Project Structure

Complete directory layout for a full-stack EdgeOne Pages site. Only include directories for enabled modules.

## Full Template (all modules enabled)

```
my-site/
├── index.html                          # Vite entry
├── vite.config.ts                      # Vite + React config
├── tailwind.config.js
├── postcss.config.js
├── tsconfig.json
├── package.json
├── edgeone.json                        # EdgeOne Pages project config
├── .env.example                        # All possible env vars
├── .env.local                          # Actual secrets (gitignored)
├── .gitignore
│
├── middleware.js                        # Edge V8 — CORS + Auth guard
│
├── edge-functions/                     # V8 runtime (no npm)
│   └── api/
│       └── auth/
│           ├── login.js                # POST /api/auth/login
│           ├── register.js             # POST /api/auth/register
│           ├── verify.js               # GET  /api/auth/verify
│           ├── refresh.js              # POST /api/auth/refresh
│           └── oauth/
│               └── [provider].js       # GET  /api/auth/oauth/:provider
│
├── cloud-functions/                    # Node.js runtime (npm OK)
│   └── api/
│       ├── commerce/
│       │   └── [[default]].js          # Express: cart, products, orders
│       ├── pay/
│       │   └── [[default]].js          # Express: checkout, webhooks
│       ├── ai/
│       │   └── [[default]].js          # Express: chat, recommend
│       └── admin/
│           └── [[default]].js          # Express: dashboard, config, users
│
├── src/
│   ├── main.tsx                        # React entry
│   ├── App.tsx                         # Router
│   ├── index.css                       # Tailwind imports
│   ├── lib/
│   │   ├── api.ts                      # fetch wrapper with auth
│   │   ├── auth.ts                     # login/register/token helpers
│   │   └── types.ts                    # Shared TypeScript types
│   ├── components/
│   │   ├── Layout.tsx                  # Header + Footer + Sidebar
│   │   ├── Navbar.tsx
│   │   ├── CartIcon.tsx
│   │   ├── ChatWidget.tsx              # Floating AI assistant
│   │   └── ui/                         # Reusable UI (Button, Input, Card, Modal)
│   │       ├── Button.tsx
│   │       ├── Input.tsx
│   │       ├── Card.tsx
│   │       └── Modal.tsx
│   └── pages/
│       ├── Home.tsx
│       ├── Login.tsx
│       ├── Register.tsx
│       ├── Shop.tsx                    # Product listing
│       ├── ProductDetail.tsx
│       ├── Cart.tsx
│       ├── Checkout.tsx
│       ├── OrderSuccess.tsx
│       ├── admin/
│       │   ├── Dashboard.tsx           # Stats overview
│       │   ├── Products.tsx            # CRUD products
│       │   ├── Orders.tsx              # Order management
│       │   ├── Users.tsx               # User list
│       │   └── Settings.tsx            # Config hot-reload
│       └── ai/
│           └── Chat.tsx                # Full-page AI chat
│
└── public/
    ├── favicon.ico
    └── logo.svg
```

## Scaffolding Commands

```bash
# 1. Create and enter project
mkdir <project-name> && cd <project-name>

# 2. Init package.json
npm init -y

# 3. Install core deps
npm install react react-dom react-router-dom
npm install -D vite @vitejs/plugin-react typescript @types/react @types/react-dom
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p

# 4. Module-specific deps (only if enabled)
# Commerce + Pay:
npm install express stripe
# AI Agent:
npm install express
# (express is shared — only install once)
```

## vite.config.ts

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  build: {
    outDir: 'dist',
  },
  server: {
    proxy: {
      '/api': 'http://localhost:8088',
    },
  },
});
```

## edgeone.json stub

```json
{
  "envDir": "."
}
```

## Module-conditional generation

When generating the project, **only create directories/files for enabled modules**:

| Module disabled | Skip these paths |
|----------------|-----------------|
| Auth off | `edge-functions/api/auth/`, `middleware.js` auth guard section, `src/pages/Login.tsx`, `src/pages/Register.tsx` |
| Commerce off | `cloud-functions/api/commerce/`, `src/pages/Shop.tsx`, `src/pages/ProductDetail.tsx`, `src/pages/Cart.tsx` |
| Pay off | `cloud-functions/api/pay/`, `src/pages/Checkout.tsx`, `src/pages/OrderSuccess.tsx` |
| AI Agent off | `cloud-functions/api/ai/`, `src/components/ChatWidget.tsx`, `src/pages/ai/` |
| Admin off | `cloud-functions/api/admin/`, `src/pages/admin/` |

Always generate: `index.html`, `vite.config.ts`, `src/main.tsx`, `src/App.tsx`, `src/pages/Home.tsx`, `src/components/Layout.tsx`, `src/lib/api.ts`.
