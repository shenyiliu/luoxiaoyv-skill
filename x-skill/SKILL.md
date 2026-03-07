---
name: x-ops
description: >
  Full X (Twitter) operations assistant. MUST USE this skill whenever: user mentions X, Twitter,
  posting tweets, replying to tweets, running X account, growing X followers, drafting posts,
  engagement strategy, X运营, 发推, 发帖, 推特, 推文, or any X/Twitter-related task.
  Covers: account diagnostics, tweet drafting, content strategy, daily engagement (reply workflow),
  AI news research, browser-based posting via Chrome CDP (draft or publish), target account pool,
  reply style guide, and automated posting workflow.
  NOT for: non-X social media platforms.
---

# X Operations Skill

Full-stack X (Twitter) account operations: diagnostics, content, engagement, growth.

## ⚙️ Prerequisites Check（首次使用必读）

**每次激活 skill 时，先运行以下检查。缺什么就帮用户装什么。**

```bash
# 一键检查所有依赖
echo "=== Chrome CDP ===" && curl -s http://127.0.0.1:18800/json/version | python3 -c "import sys,json; print('✅ Chrome', json.load(sys.stdin).get('Browser',''))" 2>/dev/null || echo "❌ Chrome not running"
echo "=== Xvfb ===" && ls /tmp/.X1-lock &>/dev/null && echo "✅ Xvfb :1 running" || echo "❌ Xvfb not running"
echo "=== x-reader ===" && which x-reader &>/dev/null && echo "✅ x-reader installed" || echo "❌ x-reader not installed"
echo "=== websocket-client ===" && python3 -c "import websocket; print('✅ websocket-client', websocket.__version__)" 2>/dev/null || echo "❌ websocket-client not installed"
```

### 安装缺失依赖

**x-reader**（抓推文内容，必须）：
```bash
pip install "git+https://github.com/runesleo/x-reader.git" -q
```

**websocket-client**（Chrome CDP 操作，必须）：
```bash
pip install websocket-client -q
```

**Xvfb**（虚拟显示器，云服务器必须，本地 Mac/Windows 不需要）：
```bash
# Ubuntu/Debian
apt-get install -y xvfb x11-utils
Xvfb :1 -screen 0 1920x1080x24 &
export DISPLAY=:1

# CentOS/RHEL
yum install -y xorg-x11-server-Xvfb
Xvfb :1 -screen 0 1920x1080x24 &
export DISPLAY=:1
```

**Google Chrome**（云服务器必须，本地已有可跳过）：
```bash
# Ubuntu/Debian
wget -q https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
apt-get install -y ./google-chrome-stable_current_amd64.deb

# CentOS/RHEL
wget https://dl.google.com/linux/direct/google-chrome-stable_current_x86_64.rpm
yum install -y ./google-chrome-stable_current_x86_64.rpm
```

**KasmVNC**（可选，远程桌面查看 Chrome，云服务器推荐）：
```bash
# Ubuntu 22.04
wget https://github.com/kasmtech/KasmVNC/releases/download/v1.3.3/kasmvncserver_jammy_1.3.3_amd64.deb
apt-get install -y ./kasmvncserver_jammy_1.3.3_amd64.deb
kasmvncpasswd  # 设置密码
vncserver :1 -geometry 1920x1080
# 访问 https://<your-ip>:8444
```

### 启动 Chrome（如未运行）
```bash
DISPLAY=:1 nohup google-chrome-stable \
  --no-sandbox --disable-setuid-sandbox --disable-dev-shm-usage \
  --remote-debugging-port=18800 --remote-allow-origins='*' \
  --user-data-dir=$HOME/.chrome-x-ops \
  --no-first-run --start-maximized https://x.com \
  > /tmp/chrome.log 2>&1 &
sleep 5 && curl -s http://127.0.0.1:18800/json/version | python3 -c "import sys,json; print('✅', json.load(sys.stdin).get('Browser',''))"
```

### X 账号登录（Cookie 注入）
X 在新环境会触发安全验证，用 Cookie 注入绕过：
1. 本地浏览器打开 x.com 登录
2. DevTools → Application → Cookies → 复制 `auth_token`、`ct0`、`twid`、`kdt`
3. 用 Python CDP 注入：
```python
import json, urllib.request, websocket
tabs = json.loads(urllib.request.urlopen('http://127.0.0.1:18800/json').read())
xtab = next(t for t in tabs if 'x.com' in t.get('url','') and t.get('type')=='page')
ws = websocket.create_connection(xtab['webSocketDebuggerUrl'])
for i, (name, value) in enumerate([
    ('auth_token', 'YOUR_AUTH_TOKEN'),
    ('ct0', 'YOUR_CT0'),
    ('twid', 'YOUR_TWID'),
    ('kdt', 'YOUR_KDT'),
]):
    ws.send(json.dumps({'id': i+1, 'method': 'Network.setCookie',
        'params': {'name': name, 'value': value, 'domain': '.x.com', 'path': '/', 'secure': True, 'httpOnly': True}}))
    ws.recv()
ws.send(json.dumps({'id': 99, 'method': 'Page.navigate', 'params': {'url': 'https://x.com/home'}}))
ws.recv(); ws.close(); print('✅ Cookies injected')
```

---

## User Context

- Account: @YOUR_HANDLE (read from AGENTS.md)
- Product: YOUR_PRODUCT (read from AGENTS.md)
- Positioning: English-first, targeting global AI developer / indie hacker / vibe coding community
- Communication: Chinese with user, English for all published content on X
- Goal: **Brand awareness** — 让 AI developer / indie hacker 圈子认识你和你的产品（见 AGENTS.md）

## Fetching Tweets

Use **x-reader** to fetch latest tweets from any account:

```bash
x-reader https://x.com/username
```

Returns structured content with title, latest tweets, author, publish time. No API keys needed.

**Fallback**: If x-reader fails, draft from known account topic/style — never skip.

---

## 🎯 账号定位与品牌认知策略

**阶段目标**：用 2-3 个月，在 indie hacker / AI builder 圈子建立认知
- "一个在做 AI app builder 的开发者，同时对 AI 前沿技术有自己的见解"
- 不是靠单条爆款，而是靠**持续出现 + 有观点**

**内容宽度**：不只发 [YOUR_PRODUCT]，内容分两类：
1. **Builder 视角**（[YOUR_PRODUCT] / 产品 / indie hacking 经验）
2. **AI 前沿观点**（最新模型、技术趋势、工具、agent 架构的真实看法）
   → 以开发者身份发言，不是科普，而是"我用过/研究过，这是我的判断"

**两条增长路径**：
1. **原创内容** — 每天 1 条，建立主页内容资产
2. **互动渗透** — 回复同类小账号，建立社区关系

---

## Target Account Pool（已重构）

> **核心原则**：优先选 **500-5000 粉** 的中小账号。
> 大号（karpathy 级别）一条帖子有几千条回复，@YOUR_HANDLE 当前没有账号权重，回复会被算法完全忽略。
> 中小号的帖子回复数少，你的回复会被帖主本人看到，更容易建立真实连接。

### 第一梯队：优先互动（500-3000 粉，活跃 builder）
- @marc_louvion — 连续创业者，build in public 极度活跃，帖子回复数可控
- @tibo_maker — 独立开发者，产品思考
- @dannypostmaa — AI 产品，每天发
- @jordankdalton — MCP/Claude Code，工具 builder
- @swyx — AI Engineer 社区，内容质量高
- @bentossell — AI 产品社区，互动频繁

### 第二梯队：偶尔互动（5000-50000 粉）
- @simonw — AI 工具开发者，技术讨论
- @gregisenberg — AI 产品创业
- @patio11 — SaaS，indie dev
- @shreyas — 产品管理

### 第三梯队：仅观察，不回复（粉丝 100k+）
- @levelsio — 偶尔有机会时（非常具体的话题）才回
- @karpathy — 同上
- @_catwu — 同上

**轮换规则**：
- 每天从第一梯队选 **2 个**，第二梯队选 **1 个**，共 3 条回复
- 不要连续两天回复同一个人
- 每周至少覆盖第一梯队全部账号一次

---

## Reply Style Guide

- 1-2 句话，最多 3 句
- 口语化，像朋友发消息，不像在引用论文
- 有明确观点或真实 [YOUR_PRODUCT] 经验，**不附和，不夸奖**
- 自然带出 [YOUR_PRODUCT]（有机会时），不强行植入
- 不加 hashtag，不加 emoji，不说 "Great post!" / "This is so true!"
- 有时候一个反问 or 补充视角比直接同意更有记忆点

✅ Good:
> Shipped ugly, charged from day 1. The hardest part is not adding features too early.

> Solo + the right AI stack is basically a small team now. That's what [YOUR_PRODUCT] is built on.

> Curious what you do when users ask for something you deliberately left out.

❌ Bad:
> Great post! Really insightful! 🔥🔥🔥

> As an AI Agent Development Engineer, I believe the paradigm shift...

> I totally agree with everything you said here.

---

## Reply Workflow（每日互动，改版）

1. 从 Target Pool 按梯队选账号：第一梯队 2 个 + 第二梯队 1 个
2. 用 **x-reader** 获取最新推文，挑 **今天发的、主题相关的** 推文
3. x-reader 失败时：基于该账号已知话题风格直接起草，**不要空手而归**
4. 每条回复提供两个版本：
   - 🇺🇸 英文版（用于发布到 X）
   - 🇨🇳 中文版（供用户参考）
5. 附上推文原文摘要 + 链接 + 发布时间（越新越好）
6. 用 message 工具发送到 Telegram，target: `$TELEGRAM_CHAT_ID` (read from AGENTS.md)，channel: `telegram`
7. 用户手动去 X 发布（API 不支持代发回复）
8. **附上当天原创推文草稿**（见下方原创内容节奏）

**格式模板**：
```
🌐 每日英文圈互动 — [日期]

━━━━━━━━━━━━━━━━━
📌 互动 1 — @账号（梯队/粉丝数）
原推摘要：...
🔗 链接

🇺🇸 英文版（发布用）：
"..."

🇨🇳 中文版（参考）：
"..."

━━━━━━━━━━━━━━━━━
📌 互动 2 — @账号
...

━━━━━━━━━━━━━━━━━
📌 互动 3 — @账号
...

━━━━━━━━━━━━━━━━━
✍️ 今日原创推文草稿
[见内容节奏部分]

⚠️ 请手动点链接去 X 发布
```

---

## Content Strategy（原创内容）

### 账号定位：Builder + AI 技术观察者
不是"AI 工具推荐博主"，不是"创业励志号"
而是：**一个在做 AI app builder 的开发者，分享真实 building 过程 + 对 AI 前沿技术的独立判断**

### 内容类型分配（7天轮换）

#### A类：Builder 视角（每周 3-4 条）
真实 building 经历 + [YOUR_PRODUCT] 产品思考，无需真实数据时可用合理的 building 经历起草

#### B类：AI 前沿观点（每周 3-4 条）
以开发者身份，对最新 AI 进展发表简短有力的观点
- 关注领域：LLM 新模型（GPT/Claude/Gemini）、agent 架构、vibe coding、MCP、RAG、工具链
- 风格：不是科普，是"我用过/想过，这是我的判断"
- 来源：每次执行任务前用 web_search 检索当天 AI 圈热点，基于真实新闻起草

### 高效内容类型（按效果排序）
1. **反直觉观点** — 一句话颠覆预期 + 1-2 句支撑，最容易触发转发
2. **Build in Public** — 真实 building 故事 + 具体细节
3. **Lesson 系列** — Lesson N: 有系列感，适合积累回头客
4. **AI 技术 take** — 对新模型/工具/趋势的开发者视角点评
5. **产品公告** — 效果最差，避免

### 绝对避免
- 纯产品公告（"Just shipped X feature！"）
- 空洞 motivation（"Keep building."）
- 人云亦云的 AI 评论（"GPT-5 is amazing!"）
- 没有具体依据的大话

### 每日原创推文框架（按周几轮换）

**周一：Build in Public**
```
[具体场景/决定/经历]
What I thought would happen: ...
What actually happened: ...
Lesson: ...
```

**周二：Lesson 系列（[YOUR_PRODUCT] building）**
```
Lesson [N] building [YOUR_PRODUCT]:
[一句话观点]

[2-3 句具体展开]
```

**周三：AI 前沿 take**
```
[对当周某个 AI 新进展/工具/模型的一句话判断]

[为什么，从开发者/builder 视角，1-2 句]
```

**周四：反直觉观点**
```
Unpopular opinion:
[观点]

[为什么，1-2 句，具体]
```

**周五：AI 技术 + 产品交叉**
```
[某个 AI 技术趋势] is changing how [某类产品/workflow] works.

[具体例子，基于 [YOUR_PRODUCT] 或自身经历]
[一个延伸思考]
```

**周六：工具/技术选择**
```
[具体技术决策：选了什么，放弃了什么]
[trade-off 是什么]
[如果重来会怎么选]
```

**周日：周回顾**
```
This week in AI + building [YOUR_PRODUCT]:
→ [AI圈一件值得关注的事]
→ [[YOUR_PRODUCT] 进展]
→ [一个还没解决的问题]
```

### Build in Public 系列（持续更新）
- ✅ Lesson 1: Ship it, don't wait for perfection
- 📝 Lesson 2: Watch your users, stop building what's impressive
- 📝 Lesson 3: Pricing is a product decision, not a business decision
- 📝 Lesson 4: The users who complain the most are usually your best users

### AI 前沿话题池（每次执行前 web_search 补充最新）
- Claude / GPT / Gemini 新版本
- Vibe coding 趋势
- MCP（Model Context Protocol）生态
- AI agent 架构演进
- 代码生成工具（Cursor / Windsurf / Claude Code）
- Multimodal / reasoning model 进展

---

## Account Diagnostics

账号快照信息见 AGENTS.md。

拉数据方式（用 x-reader）：
```bash
x-reader https://x.com/YOUR_HANDLE
```

---

## Daily Cron（每天北京时间 9:00）

任务：每日英文圈互动 + 原创推文草稿
执行流程：
1. 从 Target Pool 按梯队轮换选账号（第一梯队 2 个 + 第二梯队 1 个）
2. 用 **x-reader** 获取最新推文（不用 X API / web_fetch）
3. 起草 3 条互动回复（英文 + 中文双版本）
4. 根据当天是周几，生成对应框架的**原创推文草稿**（英文）
5. 用 message 工具发到 Telegram ($TELEGRAM_CHAT_ID from AGENTS.md)
6. 不要在任务内用 "me" 作为 target，必须用具体 chat ID

---

## 发帖工具（Chrome CDP 浏览器自动化）

**所有发推、回复、存草稿操作必须使用原生 Chrome CDP，不用 API。**

CDP endpoint: `http://127.0.0.1:18800`
用 Python websocket 直接操控 Chrome，步骤：
1. `GET http://127.0.0.1:18800/json` 找到 x.com tab
2. 连接 tab 的 `webSocketDebuggerUrl`
3. 用 `Page.navigate` / `Runtime.evaluate` / `Page.captureScreenshot` 操作

### Chrome 状态检查 / 启动
```bash
# 检查是否在跑
curl -s http://127.0.0.1:18800/json/version

# 启动（如果没跑）
DISPLAY=:1 nohup google-chrome-stable \
  --no-sandbox --disable-setuid-sandbox --disable-dev-shm-usage \
  --remote-debugging-port=18800 --remote-allow-origins='*' \
  --user-data-dir=/root/.openclaw/browser/openclaw/user-data \
  --no-first-run --start-maximized https://x.com \
  > /tmp/chrome.log 2>&1 &
sleep 5
```

### 发原创推文（存草稿）
```bash
python3 scripts/post_to_x.py --text "推文内容"
```

### 发原创推文（直接发布）
```bash
python3 scripts/post_to_x.py --post --text "推文内容"
```

### 回复某条推文（直接发布）
用 Python CDP 脚本：
1. `Page.navigate` 到目标推文 URL
2. 等页面加载，找 `[data-testid="reply"]` 按钮点击
3. 等回复框出现，用 `execCommand('insertText')` 输入内容
4. 找 `[data-testid="tweetButton"]`（非 disabled）点击发布
5. 截图确认

### 截图确认
每次操作后用 `Page.captureScreenshot` 截图，保存到 `your-workspace/` 发给用户确认。

### 流程说明
- 存草稿 = SideNav_NewTweet_Button → 填内容 → app-bar-close → 点"保存"
- 直接发布 = 同上，最后点 tweetButton/tweetButtonInline
- 回复 = 打开推文页 → 点 reply 按钮 → 填内容 → 点 tweetButton

---

## 重要约束

- ❌ 不要用 "me" 作为 Telegram target，必须用 AGENTS.md 中配置的 TELEGRAM_CHAT_ID
- ❌ 用 x-reader 抓推文，不用 X API / web_fetch
- ❌ 发帖/回复必须用原生 Chrome CDP，不用其他方式
- ✅ 发推前先给用户看中文草稿确认，确认后操作 Chrome 存草稿或直接发布
- ✅ 互动回复：用 Chrome 打开目标推文页，找到回复框直接发布
- ✅ 每次操作后截图发给用户确认
- ✅ 互动回复必须同时提供英文版 + 中文版
- ✅ x-reader 不可用时，基于已知内容手动起草，不要空手而归
- ✅ 每日任务必须包含原创推文草稿
- ✅ 优先第一梯队账号（500-3000 粉），避免每天都选大号
