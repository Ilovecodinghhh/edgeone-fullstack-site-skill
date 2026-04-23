# EdgeOne Pages Full-Stack Site Skill

一份引导 AI Agent 从一句话生成具备**登录 · 购物车 · 支付 · AI 客服 · 管理后台**完整商业闭环的全栈网站，并部署到 EdgeOne Pages 的 Skill。

## 这个 Skill 做什么

用户只要说一句话，比如 **"帮我搭一个带登录支付和 AI 客服的电商网站"**，AI Agent 会：

1. 询问需要哪些模块（Auth / Commerce / Pay / AI / Admin）
2. 生成完整项目骨架（React + Vite + Tailwind 前端 + EdgeOne 全栈后端）
3. 按需生成每个模块的代码（Edge Functions + Cloud Functions + Middleware + KV）
4. 引导配置环境变量（支付密钥、LLM API Key 等）
5. 本地验证后交由 `edgeone-pages-deploy` 一键部署

## 架构亮点

| 特性 | 实现 |
|------|------|
| **Auth 登录** | Edge Functions + KV（JWT，Web Crypto PBKDF2，零 npm 依赖，边缘级延迟） |
| **购物车 + 订单** | Node Cloud Functions / Express（状态机订单流转，库存管理） |
| **聚合支付** | Node Cloud Functions（Stripe 完整集成 + Alipay/WechatPay 桩） |
| **AI 智能客服** | Node Cloud Functions（LLM 代理 + RAG 知识库 + 意图→动作桥接） |
| **管理后台** | Node Cloud Functions + KV（数据看板、配置热更新、用户管理） |
| **路由守卫** | Middleware（V8 边缘，CORS + JWT 验证 + 频率限制） |

## 技术栈

- **前端**: React 18 + TypeScript + Vite + Tailwind CSS + React Router
- **后端**: EdgeOne Pages (Edge Functions V8 + Node.js Cloud Functions + Express)
- **存储**: EdgeOne KV Storage（会话、配置）
- **中间件**: EdgeOne Middleware（Auth Guard、CORS）
- **支付**: Stripe（+ Alipay/WechatPay 扩展桩）
- **AI**: 任意 OpenAI 兼容 API（GPT-4o、DeepSeek、Claude 等）

## 目录结构

```
edgeone-fullstack-site-skill/
├── README.md                           # 你正在看的文件
└── skills/
    └── edgeone-pages-fullstack-site/
        ├── SKILL.md                    # Skill 入口（触发词 + 主决策流程）
        └── references/                 # 按需加载的详细文档
            ├── project-structure.md    # 项目目录结构 & 脚手架命令
            ├── auth-module.md          # 登录/注册/JWT/OAuth/中间件
            ├── commerce-module.md      # 购物车/库存/订单状态机
            ├── pay-module.md           # Stripe/Alipay/WechatPay 支付网关
            ├── ai-agent-module.md      # LLM 代理/RAG/意图识别/流式输出
            ├── admin-module.md         # 数据看板/配置中心/用户管理
            ├── frontend-module.md      # React 页面/组件/路由
            ├── env-setup.md            # 环境变量清单 & 获取指引
            └── data-schema.md          # KV 键设计 + API 数据契约 + TypeScript 类型
```

## 安装方式

### 方式一：自然语言安装（推荐）

在支持 Skills 的 AI 编程工具中：

> 帮我安装这个 skill：`https://github.com/<your-repo>/edgeone-fullstack-site-skill`

### 方式二：手动安装

将 `skills/edgeone-pages-fullstack-site/` 目录复制到你的工具 skills 目录：

```bash
# Claude Code
cp -r skills/edgeone-pages-fullstack-site/ ~/.claude/skills/

# CodeBuddy / WorkBuddy
cp -r skills/edgeone-pages-fullstack-site/ ~/.codebuddy/skills/

# Cursor
cp -r skills/edgeone-pages-fullstack-site/ .cursor/rules/
```

## 触发方式

在 AI 对话中使用自然语言：

- "帮我搭一个电商网站部署到 EdgeOne Pages"
- "建一个带登录、购物车和 AI 客服的网站"
- "一句话生成完整商业网站"
- "Build a full-stack e-commerce site on EdgeOne Pages"
- "Create a website with auth, payments, and AI assistant"

## 与官方 EdgeOne Pages Skills 的关系

| Skill | 职责 | 来源 |
|-------|------|------|
| **edgeone-pages-fullstack-site**（本 Skill） | 全栈商业站脚手架 + 模块化代码生成 | 本仓库 |
| `edgeone-pages-dev` | Edge/Cloud Functions、中间件、KV 开发指南 | [官方](https://github.com/TencentEdgeOne/edgeone-pages-skills) |
| `edgeone-pages-deploy` | 部署上线到 EdgeOne Pages | [官方](https://github.com/TencentEdgeOne/edgeone-pages-skills) |

推荐串联使用：**fullstack-site → dev（可选，加自定义逻辑） → deploy**

## 模板预设

| 模板 | 包含模块 | 适用场景 |
|------|---------|---------|
| 🛒 电商全家桶 | Auth + Commerce + Pay + AI + Admin | 电商、独立站 |
| 🔧 SaaS / 工具站 | Auth + AI + Admin | SaaS 工具、内部系统 |
| 📝 内容 + 变现 | Auth + Pay + AI | 知识付费、会员站 |
| 🎯 自定义 | 自选模块组合 | 任意场景 |

## License

MIT
