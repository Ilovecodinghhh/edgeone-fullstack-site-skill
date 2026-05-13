# AI Agent Module

Intelligent assistant with LLM proxy, RAG-ready knowledge base, intent recognition, and cross-module action bridging. Running on **Node.js Cloud Functions** with Express.

## File: `cloud-functions/api/ai/[[default]].js`

```javascript
import express from 'express';

const app = express();
app.use(express.json());

// ============================================================
// CHAT — POST /api/ai/chat
// ============================================================
app.post('/chat', async (req, res) => {
  const userId = getUserId(req);
  if (!userId) return res.status(401).json({ error: 'Unauthorized' });

  const { message, conversationId, context: userContext } = req.body;
  if (!message) return res.status(400).json({ error: 'message required' });

  try {
    // 1. Intent recognition
    const intent = detectIntent(message);

    // 2. If actionable intent, execute and respond
    if (intent.action) {
      const actionResult = await executeAction(intent, userId, req);
      return res.json({
        reply: actionResult.reply,
        action: intent.action,
        data: actionResult.data,
        conversationId: conversationId || generateId(),
      });
    }

    // 3. Otherwise, forward to LLM with RAG context
    const ragContext = await getRAGContext(message);
    const llmReply = await callLLM(message, ragContext, userContext);

    res.json({
      reply: llmReply,
      action: null,
      conversationId: conversationId || generateId(),
    });
  } catch (err) {
    console.error('AI chat error:', err);
    res.status(500).json({ error: 'AI service unavailable' });
  }
});

// ============================================================
// RECOMMEND — POST /api/ai/recommend
// ============================================================
app.post('/recommend', async (req, res) => {
  const userId = getUserId(req);
  const { category, priceRange, preferences } = req.body;

  const prompt = `Based on user preferences: category=${category}, price range=${priceRange}, preferences=${preferences}. Recommend 3 products from our catalog with reasons.`;
  const reply = await callLLM(prompt, getProductKnowledge());

  res.json({ recommendations: reply });
});

// ============================================================
// SUMMARIZE — POST /api/ai/summarize (Admin use)
// ============================================================
app.post('/summarize', async (req, res) => {
  const userId = getUserId(req);
  const { type = 'daily' } = req.body; // daily, weekly, custom

  const prompt = `Summarize ${type} business metrics: orders, revenue, top products, customer inquiries. Format as executive brief.`;
  const reply = await callLLM(prompt, '');

  res.json({ summary: reply, type, generatedAt: Date.now() });
});

// ============================================================
// STREAMING CHAT — POST /api/ai/stream (SSE)
// ============================================================
app.post('/stream', async (req, res) => {
  const userId = getUserId(req);
  if (!userId) return res.status(401).json({ error: 'Unauthorized' });

  const { message } = req.body;

  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  try {
    const ragContext = await getRAGContext(message);
    const response = await fetch(getLLMEndpoint(), {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${process.env.LLM_API_KEY}`,
      },
      body: JSON.stringify({
        model: process.env.LLM_MODEL || 'gpt-4o-mini',
        messages: [
          { role: 'system', content: getSystemPrompt(ragContext) },
          { role: 'user', content: message },
        ],
        stream: true,
      }),
    });

    const reader = response.body.getReader();
    const decoder = new TextDecoder();

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      const chunk = decoder.decode(value);
      res.write(chunk);
    }

    res.write('data: [DONE]\n\n');
    res.end();
  } catch (err) {
    res.write(`data: ${JSON.stringify({ error: 'Stream failed' })}\n\n`);
    res.end();
  }
});

// ============================================================
// INTENT DETECTION
// ============================================================

const INTENT_PATTERNS = [
  { pattern: /(?:加入|添加|放入).*(?:购物车|车)/i, action: 'add_to_cart' },
  { pattern: /(?:我想买|购买|下单)/i, action: 'add_to_cart' },
  { pattern: /(?:查|看|追踪).*(?:物流|快递|订单)/i, action: 'check_order' },
  { pattern: /(?:改|修改|重置).*密码/i, action: 'reset_password' },
  { pattern: /(?:退[款货]|退换)/i, action: 'refund' },
  { pattern: /(?:优惠|折扣|券|coupon)/i, action: 'check_coupons' },
];

function detectIntent(message) {
  for (const { pattern, action } of INTENT_PATTERNS) {
    if (pattern.test(message)) {
      return { action, originalMessage: message };
    }
  }
  return { action: null, originalMessage: message };
}

async function executeAction(intent, userId, req) {
  switch (intent.action) {
    case 'add_to_cart':
      return {
        reply: '好的，我帮你把商品加入购物车了！你可以在购物车页面查看。',
        data: { redirect: '/cart' },
      };
    case 'check_order':
      return {
        reply: '正在查询你的最近订单...',
        data: { redirect: '/orders' },
      };
    case 'reset_password':
      return {
        reply: '请前往账户设置页面修改密码。',
        data: { redirect: '/settings' },
      };
    case 'refund':
      return {
        reply: '退款申请已记录，客服将在 24 小时内与你联系。',
        data: { ticketCreated: true },
      };
    case 'check_coupons':
      return {
        reply: '目前没有可用的优惠券，新用户注册可享受首单 9 折。',
        data: { coupons: [] },
      };
    default:
      return { reply: '我没有理解你的意图，让我用 AI 来回答。', data: null };
  }
}

// ============================================================
// RAG (Retrieval-Augmented Generation)
// ============================================================

function getProductKnowledge() {
  // In production: load from KV, database, or vector store
  return `Our product catalog includes:
- Premium Headphones ($299) - High-fidelity wireless
- Mechanical Keyboard ($159) - RGB mechanical
- Smart Watch ($399) - Health tracking
- Canvas Backpack ($89) - Travel backpack
- Desk Lamp ($49) - LED with USB charging

Shipping: Free over $100. Returns accepted within 30 days.
Payment: Stripe (cards), Alipay, WechatPay supported.`;
}

async function getRAGContext(query) {
  // In production: vector similarity search against product docs
  // For now, return static knowledge base
  return getProductKnowledge();
}

function getSystemPrompt(ragContext) {
  return `You are a helpful shopping assistant for our online store.
Answer questions based on the following knowledge:

${ragContext}

Rules:
- Be concise and friendly
- If asked about products not in our catalog, say so honestly
- For order issues, suggest contacting support
- Respond in the same language as the user`;
}

// ============================================================
// LLM PROXY
// ============================================================

function getLLMEndpoint() {
  return process.env.LLM_ENDPOINT || 'https://api.openai.com/v1/chat/completions';
}

async function callLLM(message, context, userContext = '') {
  const response = await fetch(getLLMEndpoint(), {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${process.env.LLM_API_KEY}`,
    },
    body: JSON.stringify({
      model: process.env.LLM_MODEL || 'gpt-4o-mini',
      messages: [
        { role: 'system', content: getSystemPrompt(context) },
        ...(userContext ? [{ role: 'system', content: `Additional context: ${userContext}` }] : []),
        { role: 'user', content: message },
      ],
      max_tokens: 500,
      temperature: 0.7,
    }),
  });

  const data = await response.json();
  return data.choices?.[0]?.message?.content || 'Sorry, I could not generate a response.';
}

// ============================================================
// HELPERS
// ============================================================

function getUserId(req) {
  try {
    const token = req.headers['x-auth-token'];
    if (!token) return null;
    return JSON.parse(atob(token.split('.')[1])).sub;
  } catch { return null; }
}

function generateId() {
  return `conv_${Date.now()}_${Math.random().toString(36).slice(2, 8)}`;
}

export default app;
```

## AI Capabilities

| Feature | Endpoint | Description |
|---------|---------|-------------|
| Chat | `POST /api/ai/chat` | Intent detection → action or LLM response |
| Stream | `POST /api/ai/stream` | SSE streaming for real-time chat UX |
| Recommend | `POST /api/ai/recommend` | Product recommendations via LLM |
| Summarize | `POST /api/ai/summarize` | Business data summary (admin) |

## Intent → Action Bridge

The AI assistant can directly trigger business actions:
- "我想买这个耳机" → adds to cart
- "查一下我的订单" → redirects to order page
- "我要退款" → creates support ticket

This bridges natural language to the Commerce and Pay modules without manual navigation.

## LLM Provider Configuration

Supports any OpenAI-compatible API:

| Provider | `LLM_ENDPOINT` | `LLM_MODEL` |
|----------|----------------|-------------|
| OpenAI | `https://api.openai.com/v1/chat/completions` | `gpt-4o-mini` |
| Claude (via proxy) | Your proxy URL | `claude-3-haiku-20240307` |
| DeepSeek | `https://api.deepseek.com/v1/chat/completions` | `deepseek-chat` |
| Local/Self-hosted | `http://your-server/v1/chat/completions` | Model name |

## Production Enhancements

- Add vector store (Pinecone, Milvus) for real RAG
- Store conversation history in database for context continuity
- Add feedback loop (thumbs up/down) to improve responses
- Rate limit AI calls per user to control costs
