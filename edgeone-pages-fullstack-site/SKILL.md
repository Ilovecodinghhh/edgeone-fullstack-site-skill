---
name: edgeone-pages-fullstack-site
description: >-
  This skill scaffolds a production-ready full-stack website with pluggable business modules
  (Auth, Commerce/Cart, Payment, AI Assistant, Admin Dashboard) on EdgeOne Pages.
  It generates a complete project skeleton from a single sentence, using Edge Functions,
  Node Cloud Functions (Express), KV Storage, and Middleware — then hands off to
  edgeone-pages-deploy for one-click deployment.
  Trigger when the user wants to: build an e-commerce site, create a site with login/auth,
  add shopping cart / payment / checkout, build a site with AI customer service,
  create an admin dashboard, scaffold a full-stack business site on EdgeOne Pages,
  "帮我搭一个带登录支付的网站", "建一个电商站", "做一个有 AI 客服的网站",
  "一句话生成完整项目", "搭一个带后台管理的站".
  Do NOT trigger for pure static sites without business logic (use a prompt instead).
  Do NOT trigger for SaaS-starter template projects (use edgeone-pages-saas).
  Do NOT trigger for deployment only (use edgeone-pages-deploy).
metadata:
  author: workbuddy-fullstack
  version: "1.0.0"
---

# EdgeOne Pages Full-Stack Site Skill

Scaffold a modular, production-ready website with **Auth · Commerce · Payment · AI Assistant · Admin** on EdgeOne Pages — from one sentence to deployed site.

## Architecture Overview

```
┌─────────────────────────────────────────────────┐
│                  Frontend (React/Vite)           │
│  Landing · Shop · Cart · Checkout · Admin · Chat │
└──────────────┬──────────────────────────────────┘
               │ fetch /api/*
┌──────────────▼──────────────────────────────────┐
│  middleware.js  (Edge V8 — Auth guard, CORS)     │
└──────────────┬──────────────────────────────────┘
               │
  ┌────────────┼────────────┬───────────────┐
  ▼            ▼            ▼               ▼
Edge Fn    Node Fn       Node Fn         Node Fn
KV Store   Express       Express         Express
(session   (commerce     (pay            (ai-agent
 + config)  + cart)       gateway)        + admin)
```

**Tech stack:**
- Frontend: React 18 + Vite + TypeScript + Tailwind CSS
- Auth: Edge Functions + KV (JWT sessions, OAuth stubs)
- Commerce: Node Functions / Express (cart, inventory, orders)
- Payment: Node Functions (Stripe / Alipay / WechatPay webhook stubs)
- AI Assistant: Node Functions (LLM proxy, RAG-ready, intent→action bridge)
- Admin: Node Functions + KV (dashboard API, config hot-reload)
- Middleware: Edge V8 (auth guard, CORS, rate limiting)

## ⛔ Critical Rules (never skip)

1. **Always ask the user what modules they need** before generating. Never assume all modules are wanted.
2. **Use `references/*.md`** for implementation details — do NOT dump all code in SKILL.md.
3. **Never hardcode secrets.** All API keys go in `.env.local`; KV stores only non-secret config.
4. **Deployment is NOT this skill's job.** Hand off to `edgeone-pages-deploy` at the end.
5. **KV namespace must be created in the console first.** Always remind the user of this prerequisite.
6. **Edge Functions cannot use npm.** Auth/session logic in Edge Functions must be pure JS (Web Crypto for JWT).
7. **Express functions use `[[default]].js`** and `export default app` — never call `app.listen()`.

---

## Main Flow

### Step 1 — Intake: What are you building?

Ask the user (use IDE selection UI if available):

> 你想搭建什么类型的网站？先告诉我一句话描述，或者选择预设模板：
>
> - **🛒 电商全家桶** — Auth + 购物车 + 支付 + AI 客服 + 管理后台（全模块）
> - **🔧 SaaS / 工具站** — Auth + AI 助手 + 管理后台（无购物车/支付）
> - **📝 内容 + 变现** — Auth + 支付 + AI 助手（无购物车）
> - **🎯 自定义** — 我来选需要哪些模块

If user picks custom, show module checklist:

| Module | Capability | Dependencies |
|--------|-----------|-------------|
| **Auth** | JWT 登录、注册、OAuth 桩、权限校验 | — (基础模块) |
| **Commerce** | 购物车、库存、订单状态机 | Auth |
| **Pay** | 聚合支付网关、回调处理 | Auth, Commerce |
| **AI Agent** | 智能客服、意图识别、商品推荐 | Auth |
| **Admin** | 数据看板、配置中心、用户管理 | Auth |

Collect: `projectName`, `templateType`, `enabledModules[]`, `brandColor` (optional), `locale` (zh/en/both).

Show the **Spec** back and confirm before proceeding.

---

### Step 2 — Scaffold the project

Read **`references/project-structure.md`** and follow it to:

1. Create project directory
2. Initialize `package.json` with all required deps
3. Create the full directory tree based on enabled modules
4. Set up `edgeone.json` stub

```bash
mkdir <project-name> && cd <project-name>
npm init -y
npm install react react-dom vite @vitejs/plugin-react tailwindcss postcss autoprefixer
npm install -D typescript @types/react @types/react-dom
```

If Commerce or Pay enabled:
```bash
npm install express stripe
```

If AI Agent enabled:
```bash
npm install express
```

---

### Step 3 — Generate module code

For each enabled module, read the corresponding reference and generate code:

| Module | Read | Generates |
|--------|------|-----------|
| Auth | [references/auth-module.md](references/auth-module.md) | `edge-functions/api/auth/*`, `middleware.js`, KV session logic |
| Commerce | [references/commerce-module.md](references/commerce-module.md) | `cloud-functions/api/commerce/*` (Express) |
| Pay | [references/pay-module.md](references/pay-module.md) | `cloud-functions/api/pay/*` (Express) |
| AI Agent | [references/ai-agent-module.md](references/ai-agent-module.md) | `cloud-functions/api/ai/*` (Express) |
| Admin | [references/admin-module.md](references/admin-module.md) | `cloud-functions/api/admin/*` (Express), KV config |
| Frontend | [references/frontend-module.md](references/frontend-module.md) | `src/*` (React pages, components, routing) |

Generate one module at a time. Report progress after each.

---

### Step 4 — Environment variables

Read **`references/env-setup.md`** and:

1. Generate `.env.example` with all required keys (filtered by enabled modules)
2. Copy to `.env.local`
3. Guide user through obtaining each key
4. Ensure `.gitignore` covers `.env.local`

---

### Step 5 — Local verification

```bash
edgeone pages dev    # http://localhost:8088
```

Checklist:
- [ ] Homepage renders
- [ ] Auth login/register works (if enabled)
- [ ] Cart add/remove works (if Commerce enabled)
- [ ] AI chat responds (if AI Agent enabled)
- [ ] Admin dashboard loads (if Admin enabled)

If issues, check: missing env vars, KV not bound, deps not installed.

---

### Step 6 — Hand off to deployment

Same pattern as `edgeone-pages-saas` Step 6:

1. Check if `edgeone-pages-deploy` skill is installed
2. If yes → tell user to say "部署到 EdgeOne Pages"
3. If no → guide install or provide minimal deploy commands

> ✅ 项目已就绪！直接对我说 **"部署到 EdgeOne Pages"** 即可上线。

---

## Routing

| Task | Read |
|------|------|
| Full project directory structure & scaffolding | [references/project-structure.md](references/project-structure.md) |
| Auth module (JWT, OAuth, sessions, middleware) | [references/auth-module.md](references/auth-module.md) |
| Commerce module (cart, inventory, orders) | [references/commerce-module.md](references/commerce-module.md) |
| Payment module (Stripe, Alipay, WechatPay) | [references/pay-module.md](references/pay-module.md) |
| AI Assistant module (LLM proxy, RAG, intent) | [references/ai-agent-module.md](references/ai-agent-module.md) |
| Admin Dashboard module (stats, config, users) | [references/admin-module.md](references/admin-module.md) |
| Frontend pages & components (React + Tailwind) | [references/frontend-module.md](references/frontend-module.md) |
| Environment variable setup & service guides | [references/env-setup.md](references/env-setup.md) |
| Data schema (KV key design + API contracts) | [references/data-schema.md](references/data-schema.md) |

---

## Quick Reference

**Bootstrap:** `mkdir my-site && cd my-site && npm init -y`
**Dev:** `edgeone pages dev` (port 8088)
**Deploy:** hand off to `edgeone-pages-deploy` skill
**KV prerequisite:** Enable in EdgeOne Pages console → Create namespace → Bind to project
