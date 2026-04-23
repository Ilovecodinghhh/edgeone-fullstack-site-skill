# Environment Variables

All secrets go in `.env.local` (gitignored). Copy from `.env.example` and fill in.

## Full Variable List

```bash
# =============================================
# AUTH (Required if Auth module enabled)
# =============================================
JWT_SECRET=your-secret-key-at-least-32-chars-long
# Generate: node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"

# OAuth (Optional)
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
OAUTH_REDIRECT_URL=https://your-site.edgeone.cool/api/auth/oauth/callback

# =============================================
# PAYMENT (Required if Pay module enabled)
# =============================================

# Stripe
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
# Get keys: https://dashboard.stripe.com/apikeys
# Set webhook: https://dashboard.stripe.com/webhooks → endpoint URL: https://your-site.edgeone.cool/api/pay/webhook/stripe

# Alipay (Optional)
ALIPAY_APP_ID=
ALIPAY_PRIVATE_KEY=

# WechatPay (Optional)
WECHAT_MCH_ID=
WECHAT_API_KEY=
WECHAT_SERIAL_NO=

# =============================================
# AI ASSISTANT (Required if AI module enabled)
# =============================================
LLM_API_KEY=sk-...
LLM_ENDPOINT=https://api.openai.com/v1/chat/completions
LLM_MODEL=gpt-4o-mini
# Supported: OpenAI, DeepSeek, Claude (via proxy), any OpenAI-compatible API

# =============================================
# SITE
# =============================================
SITE_URL=https://your-site.edgeone.cool
SITE_NAME=My Store

# =============================================
# ADMIN
# =============================================
ADMIN_EMAILS=admin@example.com
```

## Module → Required Variables

| Module | Required | Optional |
|--------|----------|----------|
| Auth | `JWT_SECRET` | `GITHUB_CLIENT_ID/SECRET`, `GOOGLE_CLIENT_ID/SECRET`, `OAUTH_REDIRECT_URL` |
| Commerce | (none — uses in-memory or database) | Database connection string if using external DB |
| Pay | `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET` | `ALIPAY_*`, `WECHAT_*` |
| AI Agent | `LLM_API_KEY` | `LLM_ENDPOINT`, `LLM_MODEL` |
| Admin | `ADMIN_EMAILS` | (none) |
| All | `SITE_URL` | `SITE_NAME` |

## How to Obtain Keys

### JWT Secret
```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

### Stripe
1. Go to https://dashboard.stripe.com/register (or login)
2. **API Keys** → Copy `Secret key` (starts with `sk_test_`)
3. **Webhooks** → Add endpoint → URL: `https://your-site/api/pay/webhook/stripe`
4. Select events: `checkout.session.completed`, `payment_intent.payment_failed`
5. Copy **Signing secret** (starts with `whsec_`)

### OpenAI (for AI module)
1. Go to https://platform.openai.com/api-keys
2. Create new key → Copy (starts with `sk-`)

### DeepSeek (alternative AI)
1. Go to https://platform.deepseek.com/api_keys
2. Create key → set `LLM_ENDPOINT=https://api.deepseek.com/v1/chat/completions` and `LLM_MODEL=deepseek-chat`

### GitHub OAuth
1. https://github.com/settings/developers → New OAuth App
2. Authorization callback URL: `https://your-site/api/auth/oauth/callback`
3. Copy Client ID and Client Secret

## `.gitignore` additions

```
.env.local
.env.*.local
.edgeone/.token
node_modules/
dist/
```

## Setting Environment Variables on EdgeOne Pages

After deployment, set env vars in the EdgeOne Pages console:
1. Open project → **Settings** → **Environment Variables**
2. Add each key-value pair
3. Redeploy for changes to take effect

Or use CLI:
```bash
edgeone pages env pull    # Sync from console to local .env
```
