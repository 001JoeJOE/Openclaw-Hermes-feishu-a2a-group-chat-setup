---
name: feishu-a2a-group-chat-setup
description: 完整配置和排查飞书群聊 A2A（Agent 2 Agent）模式。当多个 AI Agent Bot 在同一飞书群协作时，用来配置、诊断、修复群消息收发问题。适用于 Hermes Gateway、OpenClaw、feishu-claude-bridge 等任意飞书 Bot 框架。
version: 2.1.0
author: Hermes System Assistant
license: MIT
metadata:
  hermes:
    tags: [feishu, bot, a2a, group-chat, gateway, openclaw, config]
    related_skills: [feishu-bot, feishu-long-connection, hermes-gateway-wsl, openclaw-upgrade-troubleshooting]
---

# 飞书群聊 A2A（Agent 2 Agent）模式配置与排查

> 多个 AI Agent Bot 在同一飞书群协作时，按本文档执行即可完成配置或快速修复。
>
> **本指南适用于所有飞书 Bot 框架**——Hermes Gateway、OpenClaw、feishu-claude-bridge 或其他自定义 Bot 均可使用。飞书侧的权限配置是通用的，框架侧的具体配置会分别说明。

---

## Overview

飞书群聊 A2A 意味着多个 Bot 可以在同一个飞书群里互相 @、对话、协作。这套配置的核心难点不在 Bot 框架本身，而在**飞书开发者后台的权限开关和事件路由**——飞书侧漏掉任何一个开关，消息就压根到不了你的 Bot 程序。

本文档把配置和排查封装成 agent 可执行的操作步骤。用户问"Bot 在群里收不到 @消息"、"群聊不回"、"A2A 配不起来"时，加载本 skill 按步骤执行即可。

---

## When to Use

- 用户说"我的 Bot 在群里收不到 @消息"
- 用户说"Bot 在群里能发消息但收不到回复"
- 用户想配置多个 Bot 在同一群协作（A2A 模式）
- 用户说"重启后 Bot 突然不回群消息了"
- 用户说"日志只有 `group_policy_rejected`"

**不要用于：** 单聊（DM）消息问题、Bot 纯 API 调用问题、框架自身启动失败（如 Hermes Gateway 崩溃走对应 skill）

---

## Prerequisites（让用户准备的）

开始前，让用户确认以下信息，缺一不可：

1. **飞书开发者后台访问权限** — 用户需要能登录 `open.feishu.cn` 并操作对应的 Bot 应用
2. **对应 Bot 的 App ID 和 App Secret** — 在应用凭证页面可找到
3. **群 ID（`oc_xxx`）** — 目标群的 ID
4. **所有群成员（人和 Bot）的 open_id 列表** — 后面步骤会获取
5. **确认用户在用什么 Bot 框架** — Hermes Gateway / OpenClaw / feishu-claude-bridge / 其他

---

## 配置步骤

### Step 1：飞书开发者后台 — 检查并开通权限

让用户打开飞书开发者后台 → 找到对应的 Bot 应用 → **权限管理 → 添加权限**，确认以下权限全部开启：

| 权限 | 作用 | 必要程度 |
|------|------|----------|
| `im:message` | 收发消息基础权限 | ✅ 必须 |
| `im:message:group_at_msg` | **接收群 @消息** | ✅ **关键——最容易遗漏** |
| `im:message:send_as_bot` | 以 Bot 身份发送消息 | ✅ 必须 |
| `im:message:msg_preview` | 消息预览 | ✅ 推荐 |
| `im:chat:readonly` | 读取群信息、成员列表 | ✅ 排查必备 |
| `im:resource` | 下载图片、文件等资源 | ✅ 收发文件时必需 |
| `im:message.p2p_msg` | 接收单聊消息（私信） | ✅ 必须 |
| `im:message.group_msg` | 接收群聊消息 | ✅ 必须 |
| `contact:contact:readonly` | 读取通讯录信息 | ✅ 获取 open_id 时需要 |

**⚠️ 关键提醒告诉用户：**
- `im:message:group_at_msg` **不是** `im:message` 的子权限，必须**手动额外添加**
- 添加权限后，必须**发布新版本**（应用版本管理 → 创建版本 → 申请发布）才能在生产环境生效
- 测试环境（沙箱模式）下权限即时生效，无需发布
- **以上权限对所有 Bot 框架（Hermes/OpenClaw/Claude-bridge）都一样**

### Step 2：飞书开发者后台 — 事件订阅

权限管理 → **事件与回调** → 订阅 `im.message.receive_v1`

订阅方式建议告诉用户选 **WebSocket（长连接）**：
- 无需公网 URL，直连飞书消息推送
- 框架自动管理 WebSocket 连接和心跳
- Webhook 模式仅当有公网服务器时才考虑

### Step 3：获取 Bot 的真实 open_id

**这是最常见的踩坑点。** 同一个 Bot 在不同 App 视角下 open_id 不同。**唯一正确获取方式**是从 Bot 自己的 API 获取：

```bash
# 1. 先获取 tenant_access_token
curl -X POST 'https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal' \
  -H 'Content-Type: application/json' \
  -d '{
    "app_id": "<FEISHU_APP_ID>",
    "app_secret": "<FEISHU_APP_SECRET>"
  }'

# 2. 用 token 获取 Bot 信息
curl -H "Authorization: Bearer <tenant_access_token>" \
  https://open.feishu.cn/open-apis/bot/v3/info

# 返回值中的 bot.open_id 才是对的
```

**❌ 不要**从其他 Bot 收到的 @mention 日志中提取 open_id——跨 App 提取的 ID 是错的。

如果用户无法执行 API 调用，可以在服务端帮用户执行（需要用户提供 App ID 和 App Secret）。

### Step 4：收集所有群成员的 open_id

需要群内**所有人类用户和所有 Bot** 的 open_id，缺少任何一个都会导致该成员的消息被策略拒绝（日志显示 `group_policy_rejected`）。

获取方式：

**方式 A（推荐）：从群消息记录提取**
```bash
curl -H "Authorization: Bearer <tenant_access_token>" \
  "https://open.feishu.cn/open-apis/im/v1/messages?container_id_type=chat&container_id=oc_<group_id>&page_size=50&sort_type=ByCreateTimeDesc"

# 从返回的 items[].sender.id 中提取所有 sender 的 open_id，去重
```

**方式 B：从群成员列表获取**
```bash
curl -H "Authorization: Bearer <tenant_access_token>" \
  "https://open.feishu.cn/open-apis/im/v1/chats/oc_<group_id>/members?page_size=50"
```

**⚠️ 重要提醒：** 群 open_id ≠ DM open_id。同一个人/Bot 在群聊和私聊中是两个不同的 ID。

### Step 5：配置 Bot 框架

根据用户使用的框架，执行对应的配置。

---

#### 选项 A：Hermes Gateway 配置

**`.env` 文件：**
```ini
FEISHU_APP_ID=cli_xxx...
FEISHU_APP_SECRET=xxx...
FEISHU_BOT_OPEN_ID=ou_xxx...              # Step 3 获取的值
FEISHU_ALLOWED_USERS=ou_xxx,ou_yyy,...    # Step 4 获取的全部 open_id（人和Bot）
GATEWAY_ALLOW_ALL_USERS=true               # 可选，配合白名单使用
```

**`config.yaml`：**
```yaml
channels:
  feishu:
    accounts:
      <your_account>:
        appId: cli_xxx...
        appSecret: xxx...
        group_chats:
          - oc_<group_chat_id>           # 群 ID 白名单
        group_collaboration:
          enabled: true                  # 开启群聊
          require_mention_in_group: true # 只在被 @ 时回复
          respond_to_bots: true          # ✅ 关键 A2A 设置——必须 true
          collaboration_mode: false      # 非自由回复模式
```

**关闭流式输出（多 Bot 防串台）：**
```yaml
streaming:
  enabled: false
features:
  streaming_cards: false
```

**重启：**
```bash
hermes gateway restart --profile <profile_name>
```

---

#### 选项 B：OpenClaw 配置

OpenClaw 使用 `config.yaml` 和 `.env` 配置飞书连接。OpenClaw v1.46+ 开始支持 WebSocket 飞书事件订阅。

**`.env` 文件：**
```ini
FEISHU_APP_ID=cli_xxx...
FEISHU_APP_SECRET=xxx...
```

**`config.yaml` 关键配置：**
```yaml
feishu:
  appId: cli_xxx...
  appSecret: xxx...
  eventType: websocket       # 使用 WebSocket 长连接（推荐）
  # 或
  # eventType: webhook      # 需要公网 URL
  # webhookUrl: https://...

  bot:
    openId: ou_xxx...        # Step 3 获取的 Bot open_id
    allowedUsers:            # Step 4 获取的全部 open_id
      - ou_xxx...            # 人
      - ou_yyy...            # Bot A
      - ou_zzz...            # Bot B

  groupChat:
    enabled: true
    requireMention: true     # 只在被 @ 时回复
    respondToBots: true      # ✅ 关键 A2A 设置
```

OpenClaw 的群策略位于 `config.yaml` 中的 `feishu.groupChat` 部分，配置项含义与 Hermes Gateway 一致。

**重启 OpenClaw：**
```bash
claw restart
# 或 pm2 restart claw
```

---

#### 选项 C：feishu-claude-bridge 配置

Feishu-Claude-Bridge 使用 `config.json` 或 `config.yaml` 配置：

```json
{
  "feishu": {
    "appId": "cli_xxx...",
    "appSecret": "xxx...",
    "botOpenId": "ou_xxx...",
    "allowedUsers": ["ou_xxx...", "ou_yyy..."],
    "groupChatEnabled": true,
    "requireMention": true,
    "respondToBots": true
  }
}
```

---

### Step 6：把 Bot 添加到目标群

群设置 → 群机器人 → 添加机器人，搜索 Bot 名称添加。

### Step 7：测试

在群里 @Bot 发送一条消息，观察日志确认消息到达。

---

## 快速排查（用户说"收不到消息"时按此顺序）

### 第 1 步：飞书 API 拉群最新消息

```bash
curl -H "Authorization: Bearer <tenant_access_token>" \
  "https://open.feishu.cn/open-apis/im/v1/messages?container_id_type=chat&container_id=oc_<group>&page_size=5&sort_type=ByCreateTimeDesc"
```

确认消息已到达飞书服务器。如果 API 返回空或报错，说明消息根本没发出来，和 Bot 配置无关。

### 第 2 步：飞书开发者后台检查权限
- `im:message:group_at_msg` 是否开启？
- `im.message.receive_v1` 事件是否订阅？
- 权限修改后是否**发布了新版本**？

### 第 3 步：查看目标 Bot 的框架日志

| 日志内容 | 含义 | 处理 |
|----------|------|------|
| `Inbound group message received`（Hermes）<br>或收到消息推送（OpenClaw） | ✅ 消息到了 | 检查下游响应 |
| `group_policy_rejected` 或类似策略拒绝 | ❌ 被策略拒绝 | 检查白名单和 open_id |
| 什么也没有 | ❌ WebSocket 哑死 | 走第 4 步 |

### 第 4 步：移除重加 Bot（治哑死）

群设置 → 移除该 Bot → 重新添加。这是治"TCP 连接在但消息不过来"的唯一有效解法。

**覆盖范围：** 这个问题对所有框架（Hermes/OpenClaw/Claude-bridge）都一样——是飞书侧的 WebSocket 事件路由冻结，不是框架的问题。

### 第 5 步：核对 Bot open_id
调 `bot/v3/info` API 获取实际值，和配置文件对比。跨 App 提取的 open_id 必然是错的。

### 第 6 步：核对 allowed users
拉群消息记录，提取所有 `sender.id`，确认人和 Bot 的 open_id 都在白名单中。

---

## 多个 Bot 协作时的架构

```
飞书群（群聊）
  │
  ├── 人类用户        open_id = ou_xxx_human_group
  │
  ├── Bot A（Hermes Gateway）
  │     └── 独立配置：.env + config.yaml
  │         ├── bot_open_id = ou_xxx_botA
  │         ├── allowed_users = [所有人 + 所有Bot]
  │         └── respond_to_bots: true
  │
  ├── Bot B（OpenClaw）
  │     └── 独立配置：config.yaml
  │         ├── bot.openId = ou_xxx_botB
  │         ├── bot.allowedUsers = [所有人 + 所有Bot]
  │         └── groupChat.respondToBots: true
  │
  └── Bot C（feishu-claude-bridge）
        └── 同理，独立配置
```

关键规则：
- **不管用什么框架**，每个 Bot 都需要独立的 App ID + App Secret（飞书应用）
- 所有 Bot 共享同一个群的 `oc_<group_chat_id>`
- 所有 Bot 的 respond_to_bots 都设为 true

---

## Common Pitfalls

1. **`im:message:group_at_msg` 没开。** 飞书侧直接过滤群 @消息，所有框架的日志都毫无记录。
   - 解法：权限管理 → 添加权限 → 搜 `group_at_msg` → 选中 → 发布新版本

2. **跨 App 提取 Bot open_id。** 从另一个 Bot 收到的 @mention 日志里提取，填到配置文件里，结果群消息全部被策略拒绝。
   - 解法：用 `bot/v3/info` API 获取真实 open_id

3. **只加了人的 open_id 到白名单，没加 Bot 的。** 群里有 3 个 Bot，只配了人的 ID，结果 Bot 之间 @ 消息全部被拒。
   - 解法：人和 Bot 的 open_id 都要加进白名单

4. **权限开了但没发布新版本。** 开发者后台权限开关全开了，但 Bot 还是收不到消息。
   - 解法：应用版本管理 → 创建版本 → 申请发布

5. **多次重启后 Bot 哑死。** TCP 连接正常，但群消息完全不推送。改配置、重启进程全无效。**所有框架都会遇到这个问题。**
   - 解法：群设置 → 移除 Bot → 重新添加

6. **流式输出开着导致串台（Hermes Gateway）。** 多个 Bot 同时回复时，A Bot 的回复片段出现在 B Bot 的回复中。
   - 解法：关闭 `streaming.enabled` 和 `features.streaming_cards`

7. **`respond_to_bots` 没设为 true。** Bot 收到人类 @ 消息正常，但其他 Bot @ 时完全不回应。
   - 解法：框架配置中找到响应 Bot 的开关，设为 true

8. **以为只有 Hermes 有这些问题。** OpenClaw 和 feishu-claude-bridge 一样会踩中飞书侧的权限和哑死坑——这些不是框架问题，是飞书 Bot 开发的共性问题。

---

## Verification Checklist

配置完成后按以下列表验证：

- [ ] 飞书开发者后台 → `im:message:group_at_msg` 已开启
- [ ] 飞书开发者后台 → 事件订阅 → `im.message.receive_v1` 已订阅
- [ ] 权限修改后已**发布新版本**
- [ ] Bot 已添加到目标群
- [ ] Bot open_id 来自 `bot/v3/info` API（非跨 App 提取）
- [ ] allowed_users 包含**所有人 + 所有 Bot** 的群 open_id
- [ ] `respond_to_bots`（或等效配置项）设为 `true`
- [ ] 流式输出已关闭（如框架支持）
- [ ] 发一条群 @消息测试 → 框架日志显示消息已收到
- [ ] 多个 Bot 之间互相 @ 能正常收到和回复

---

## 用户对话示例

**用户：** "我的 OpenClaw Bot 在群里不回消息了"
**Agent 操作：**
1. 查看框架日志 → 发现什么日志都没有
2. 让用户检查飞书开发者后台 → `group_at_msg` 权限是否已开
3. 权限已开，但没发布新版本 → 让用户发布版本
4. 依然收不到 → 让用户"移除 Bot → 重新添加"
5. 再 @ 测试 → 日志显示收到消息 → 修复

**用户：** "群里两个 Bot 互相 @ 没反应"
**Agent 操作：**
1. 查看 Gateay/OpenClaw 日志 → 发现 `group_policy_rejected`
2. 检查 allowed_users → Bot B 的 open_id 确实没在白名单
3. 补充 → 重启 → Bot 之间 @ 正常了

---

*文档版本: v2.1 · 2026-05-03*
