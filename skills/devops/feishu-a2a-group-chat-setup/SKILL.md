---
name: feishu-a2a-group-chat-setup
title: Feishu A2A Group Chat Setup
description: 飞书群聊 A2A（Agent 2 Agent）模式完整配置指南。当多个 AI Agent Bot 在同一飞书群协作时，从创建应用 → 配置事件 → 申请权限 → 网关对接，一条龙完成。
category: devops
---

# Feishu A2A Group Chat Setup（飞书群聊 A2A 配置指南）

> 让多个 AI Agent 在同一飞书群聊中互相协作、@彼此、自动响应。

## 适用场景

- 你有 **多个 AI Agent**（通过不同 Bot 接入），想让它们在同一个飞书群里协作
- 一个 Agent 产出结果后 @另一个 Agent 接力处理
- 需要在飞书群里看到 Agent 之间的完整对话链

## 前置条件

| 条件 | 说明 |
|------|------|
| 飞书企业版/旗舰版 | 个人版无法创建自建应用 |
| 飞书开发者后台权限 | 至少拥有一个应用的配置权限 |
| 网关服务 | 至少一个运行中的网关（Hermes Gateway / OpenClaw / feishu-claude-bridge） |

选择你的框架：

- **Hermes Gateway**（WSL/Linux） → [Step 1a](#step-1a-hermes-gateway-配置)
- **OpenClaw Gateway**（Windows） → [Step 1b](#step-1b-openclaw-gateway-配置)
- **feishu-claude-bridge**（Windows） → [Step 1c](#step-1c-feishu-claude-bridge-配置)

---

## Step 0：飞书开发者后台——创建应用 & 获取凭证

**所有框架都要做这一步。**

1. 打开 [飞书开发者后台](https://open.feishu.cn/app)
2. 点击「创建企业自建应用」，输入名称（如「龙虾暖暖」），上传头像
3. 创建成功后，左侧菜单 → **凭证与基础信息**
4. 记录以下值（后面会用到）：
   - `APP_ID`（格式如 `cli_a9f8b2c3d4e5`）
   - `APP_SECRET`（点击获取）

---

## Step 1：配置网关

### Step 1a：Hermes Gateway 配置

编辑 `~/.hermes/config.yaml`，在 gateway 段添加对应 profile：

```yaml
profiles:
  my-agent:  # profile 名称，可自定义
    model: claude-sonnet-4
    provider: anthropic
    slow_providers: []
    tools: true
    gateway:
      feishu:
        app_id: cli_a9f8b2c3d4e5
        app_secret: "你的 APP_SECRET"
```

然后在终端启动网关：

```bash
hermes gateway --profile my-agent
```

> 多 profile 需启动多个 gateway 实例（不同 tmux 窗口/会话）

### Step 1b：OpenClaw Gateway 配置

编辑 `~/.openclaw/openclaw.json`：

```json
{
  "feishu": {
    "appId": "cli_a9f8b2c3d4e5",
    "appSecret": "你的 APP_SECRET"
  }
}
```

启动：

```bash
openclaw gateway
```

### Step 1c：feishu-claude-bridge 配置

编辑 `bridge/config.yaml`：

```yaml
feishu:
  app_id: cli_a9f8b2c3d4e5
  app_secret: "你的 APP_SECRET"
```

启动：

```bash
node bridge.js
```

---

## Step 2：飞书开发者后台——订阅事件

> 让飞书把群里的新消息推送给你的 Bot。

1. **开启事件订阅**
   - 左侧菜单 → **事件与回调**
   - 开关切换为 **「启用」**
   - 配置回调地址（取决于你的框架）：

| 框架 | 回调地址（Callback URL） | 说明 |
|------|------------------------|------|
| Hermes Gateway | `http://<WSL_IP>:<port>/webhook/callback` | WSL 内网 IP，如 `http://172.x.x.x:7878/webhook/callback` |
| OpenClaw Gateway | `http://<本机IP>:<port>/` | Windows 本机 IP（非 `127.0.0.1`），如 `http://192.168.x.x:18789/` |
| feishu-claude-bridge | `http://<本机IP>:<port>/webhook/event` | 同上，端口自定义 |

> ⚠️ **顺序提醒**：建议先到 Step 3 申请权限并发布版本，权限审批通过后再回来添加事件。因为添加事件后飞书会立即验证回调地址，部分权限没通过可能导致验证失败。

2. **添加事件**
   - 点击 **「添加事件」**
   - 搜索并添加以下事件：

| 事件 | 用途 |
|------|------|
| `im.message.receive_v1` | 接收消息（必须） |
| `im.message.message_read_v1` | 消息已读（建议） |

3. **申请权限后重试**
   - 添加事件后，点击「发布版本」→ 申请权限 → 等待管理员通过
   - 通过后回到事件页面，点 **「重试」** 完成验证

---

## Step 3：飞书开发者后台——申请权限

> 你的 Bot 必须有权限才能读取群消息、发消息、查成员。

1. 左侧菜单 → **权限管理**
2. 在 **搜索框** 输入下面 **粗体关键词** 逐个搜索并添加

### IM（消息）权限

搜索 `im:message`：

| 权限全名 | 说明 |
|---------|------|
| `im:message` | 🔸 **根权限**：勾选后自动包含所有子权限，无需单独勾选下面的子项 |
| `im:message.group_msg` | 接收群聊消息 |
| `im:message.group_at_msg.include_bot:readonly` | 获取群里 Bot 被 @ 的消息（带 Bot 自身） |
| `im:message.group_at_msg:readonly` | 获取群里被 @ 的消息 |
| `im:message.group_msg:get_as_user` | 以用户身份获取群聊消息 |
| `im:message.p2p_msg:readonly` | 接收单聊消息 |
| `im:message:send_as_bot` | 以应用身份发送消息 |
| `im:message:readonly` | 读取消息 |

> 💡 **权限名格式提醒**：飞书后台权限名混用**点号**和**冒号**分隔，没有统一规则。
> - 子权限名可能是 `group_at_msg:readonly`（冒号）或 `group_msg`（点号），实际显示是什么就写什么
> - 搜不到时加`:readonly`后缀试试，很多只读权限名带这个后缀
> - 搜中文关键词兜底（`消息`、`群`、`@`）
>
> reference/feishu-permission-names.md

### 其他权限

| 搜索关键词 | 权限全名 | 说明 |
|-----------|---------|------|
| `im:chat` | `im:chat:readonly` | 读取群聊/单聊信息（群名、成员列表） |
| `im:resource` | `im:resource` | 获取消息中的图片、文件等资源 |
| `contact` | `contact:group:readonly` | 读取通讯录群组信息 |

---

## Step 4：飞书开发者后台——Bot 配置

1. 左侧菜单 → **应用功能** → **机器人**
2. 开启 **「启用机器人」**
3. 开启 **「开启机器人对话流」**（群聊 @ 时 Bot 才能看到消息上下文）
4. **关闭「群聊机器人的流式输出」**（否则群聊中消息会一段一段刷出来，看起来混乱）
5. 保存

---

## Step 5：创建飞书群 & 拉 Bot 入群

1. 在飞书客户端创建一个新群（或使用现有群）
2. 群设置 → **「群机器人」** → **「添加机器人」**
3. 搜索你的 Bot 名称并添加
4. **测试**：在群里 @Bot 发送一条消息，看 Bot 是否响应

---

## Step 6：多 Bot A2A 配置（关键步骤）

> 多个 Bot 在同一群聊协作的核心配置。

### 配置 Bot 允许交互的成员列表

**Hermes Gateway：** 在 profile 的 `.env` 或 `config.yaml` 中添加：

```yaml
gateway:
  feishu:
    allowed_users:
      - ou_xxxxx  # 用户1
      - ou_yyyyy  # 用户2（另一个 Bot）
```

> `allowed_users` 中的 `ou_xxx` 是飞书用户的 `open_id`。Bot 也是用户，有自己的 `open_id`。在飞书开发者后台 → 应用详情 → **基础信息** 可查看 Bot 的 `open_id`。

> ⚠️ 如果 `allowed_users` 为空或未配置，Bot 会响应所有消息（不推荐在生产环境这样做）。

**OpenClaw Gateway：** 在 `openclaw.json` 中配置：

```json
{
  "feishu": {
    "allowed_users": ["ou_xxxxx", "ou_yyyyy"]
  }
}
```

**feishu-claude-bridge：** 在 `config.yaml` 中：

```yaml
feishu:
  allowed_users:
    - ou_xxxxx
    - ou_yyyyy
```

### 获取 Bot 的 open_id

1. 在飞书群里 @Bot 发送一条任意消息
2. 查看 Bot 收到的消息日志，找到 `sender.open_id` 或 `open_id`
3. 将该值添加到其他 Bot 的 `allowed_users` 中

### 获取用户的 open_id

同样方法：用户在群里发一条消息，从日志中获得其 `open_id`。如果用户同时在私聊和群聊与 Bot 交互，注意：

> 飞书中同一用户的 **私聊 open_id** 和 **群聊 open_id** 可能不同。如果用户在群里 @Bot 不响应，检查 `allowed_users` 是否包含了该用户在群聊上下文中的 open_id。

---

## Step 7：启用长连接 WebSocket（可选）

> 替代公网回调地址，无需暴露端口。

**Hermes Gateway** 支持 WebSocket 长连接模式：

```bash
hermes gateway --profile my-agent --ws
```

飞书开发者后台 → **事件与回调** → 关闭 Callback URL，开启 **WebSocket** 模式即可。

**feishu-claude-bridge** 支持 WebSocket：

```yaml
# config.yaml
mode: websocket
```

> 长连接模式不需要公网 IP 和端口映射，适合内网环境。

---

## Step 8：飞书开发者后台——安全设置（可选）

> 如果你的回调地址是 HTTP（非 HTTPS），飞书默认会拒绝。在沙箱/测试环境可以关闭 IP 白名单验证。

1. 左侧菜单 → **安全设置**
2. 关闭 **「IP 白名单」**
3. 在 **「回调配置」** 中，如果使用 HTTP 而非 HTTPS，需要关闭 **「加密模式」** 或确保签名验证通过
4. （可选）添加你的服务器 IP 到白名单

---

## Step 9：发布上线

> 飞书自建应用默认只在「测试企业」或沙箱环境中运行。要让其在真实企业/群聊中生效，必须发布正式版本。

1. 飞书开发者后台 → **版本管理与发布**
2. 点击 **「创建版本」**
3. 填写版本号、更新说明
4. 确认已添加所有需要的权限和事件
5. 点击 **「发布」**
6. 等待管理员审核通过

---

## 完整配置示例

### Hermes Gateway（WSL）

```yaml
# ~/.hermes/config.yaml
profiles:
  research-assistant:
    model: claude-sonnet-4
    provider: anthropic
    slow_providers: []
    tools: true
    gateway:
      feishu:
        app_id: cli_a9f8b2c3d4e5
        app_secret: "your_app_secret"
        allowed_users:
          - ou_xxxxx  # 用户
          - ou_yyyyy  # 另一个 Bot
```

### OpenClaw Gateway（Windows）

```json
{
  "feishu": {
    "appId": "cli_a9f8b2c3d4e5",
    "appSecret": "your_app_secret",
    "allowedUsers": ["ou_xxxxx", "ou_yyyyy"]
  }
}
```

### feishu-claude-bridge（Windows）

```yaml
# config.yaml
feishu:
  app_id: cli_a9f8b2c3d4e5
  app_secret: "your_app_secret"
  allowed_users:
    - ou_xxxxx
    - ou_yyyyy
```

---

## 调试 & 排查

### Bot 不响应群 @消息

按顺序排查：

| # | 检查项 | 说明 |
|---|--------|------|
| 1 | 权限是否审批通过 | 飞书后台 → 权限管理 → 查看状态是否为 **「已通过」** |
| 2 | Bot 是否在群里 | 群设置 → 群机器人 → 确认机器人已添加 |
| 3 | 事件是否配置 | 事件与回调 → `im.message.receive_v1` 是否启用 |
| 4 | 回调地址是否正确 | 网关日志是否有飞书请求进入；用 `curl` 测试回调地址是否可达 |
| 5 | `allowed_users` 配置 | 用户的 **群聊 open_id** 是否在 `allowed_users` 列表中？用户私聊和群聊的 open_id 可能不同 |
| 6 | 长连接/Callback 模式 | 如果是 WebSocket 模式，确认飞书后台关掉了 Callback URL |
| 7 | IP 白名单 | 如果开启 IP 白名单，确认网关所在服务器 IP 已添加 |
| 8 | 版本已发布 | 开发者的修改必须「发布」后才会在生产环境生效 |

### 检查日志

```bash
# Hermes Gateway（WSL）
hermes gateway --profile my-agent --log-level debug

# OpenClaw Gateway（Windows）
查看 logs/ 目录下的日志文件

# feishu-claude-bridge
查看 bridge.log 或运行时的控制台输出
```

### 常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| `invalid signature` | APP_SECRET 或 APP_ID 配置错误 | 检查凭证与基础信息 |
| `permission denied` | 未申请对应权限 | 在权限管理添加后重新发布 |
| `event verification failed` | 回调地址无法访问 | 检查地址是否正确，是否在公网可达 |
| `invalid open_id` | 用户或 Bot 的 open_id 有误 | 从消息日志中获取准确的 open_id |
| `webhook timeout` | 网关处理时间超过飞书限制（3s） | 检查网关负载，考虑异步处理 |

---

## 验证清单

- [ ] 飞书开发者后台应用已创建
- [ ] 应用凭证（APP_ID, APP_SECRET）已记录
- [ ] 事件 `im.message.receive_v1` 已订阅
- [ ] 回调地址或 WebSocket 已配置
- [ ] 必要权限已申请并审批通过
  - [ ] `im:message`（根权限）或所有子权限
  - [ ] `im:chat:readonly`
  - [ ] `im:resource`
  - [ ] `contact:group:readonly`
- [ ] Bot 机器人功能已开启
- [ ] 「开启机器人对话流」已开启
- [ ] 「群聊机器人的流式输出」已关闭
- [ ] Bot 已拉入群聊
- [ ] 应用已发布上线
- [ ] Bot 在群聊中 @ 可正常响应
- [ ] 多 Bot A2A 模式下，每个 Bot 的 `allowed_users` 都包含其他 Bot 的 open_id

---

## 对话示例

### 单 Bot 对话

```
用户: @机器人 帮我查一下今天的天气
机器人: 今天北京天气晴，温度 22-28°C
```

### 多 Bot A2A 对话

```
用户: @研究助理 分析一下最近的股市
研究助理: 最近大盘震荡上行...
@报告生成 把分析整理成报告
报告生成: 报告已生成，要点如下...
```

---

## 附录：多 Bot 群聊行为协议

当多个 AI Agent 在同一飞书群聊中协作时，必须遵守以下规则。这些规则确保 Bot 间协作有序、不冲突、不遗漏。

### 核心规则

| # | 规则 | 说明 |
|---|------|------|
| 1 | **被@必须回应** | 无论被人类还是其他 Bot @，都必须回应 |
| 2 | **每次回应简短精炼** | 不啰嗦，直击要点，避免长段解释 |
| 3 | **群聊不能用话题回复** | 所有消息直接发在群聊中，不点「回复」创建话题/线程 |
| 4 | **@多人需逐一回应** | 一条消息 @ 了多个 Bot，每个被@的都必须回应，不遗漏 |
| 5 | **不重复其他 Bot 的话** | 做好分工，一个 Bot 已回答的，其他 Bot 不要重复相同内容 |
| 6 | **紧扣上下文不跑题** | 回应与当前对话密切相关 |
| 7 | **互相监督与检修** | 发现谁没正确回应，主动提醒对方 \[Hermes系统助理](#) |
| 8 | **未按规则会被提醒整改** | 接到提醒后立即整改 |

### 关于 @ 的注意事项

**✅ Bot 可以发出蓝色 @ 链接**

Hermes Gateway 的飞书 Bot 可以通过「post」富文本消息格式发送 `<at user_id="ou_xxx">名字</at>` 标签，飞书会将其渲染为蓝色 @ 链接。

**注意：** Hermes Gateway 目前默认将不含 Markdown 的内容以纯文本格式发送。当需要 @ 其他 Bot 时，回复内容中应包含能触发 Markdown 格式处理的格式化标记，或者等待 Hermes 更新支持显式的 `@` 语法注入。

**当前推荐做法：**
- **Bot 间相互指代时：** 可以直接用文字说出名字，其他 Bot 检查上下文主动回应
- **人类用户 @ Bot：** 在飞书客户端输入 `@` 并从弹出列表中选择
- **关键节点串联：** 由人类用户做蓝链 @ 的手动串联

**⚠️ 架构约束：每个 Bot 需要独立 Gateway**

每个 Bot 需要独立的 Gateway 进程连接飞书 WebSocket。Bot A 发出的消息只投递给 Bot A 的 Gateway，不会转发给 Bot B 的 Gateway。要让 Bot B 响应，必须有 Bot B 自己的 Gateway 进程在运行。因此 `allowed_users` 配置中必须互相包含对方的 open_id。

### 群内回应决策流程

```
收到群消息
  ↓
被@了吗？──否──→ 与自己的职责相关？──否──→ 不回应
  ↓ 是                    ↓ 是
必须回应                 可以回应
  ↓                       ↓
其他Bot已回应同类内容？
  ↓ 是                    ↓ 否
补充或不回应             回应
  ↓                       ↓
简短精炼
```

### 快速排查：Bot 不回应群 @

| # | 检查项 | 说明 |
|---|--------|------|
| 1 | 独立 Gateway 在运行？ | `ps aux \| grep -E "hermes.*gateway\|openclaw"` |
| 2 | 该 Bot 在群里吗？ | 群设置 → 群机器人 → 检查是否已添加 |
| 3 | `allowed_users` 互相包含？ | 每个 Bot 的 open_id 是否都在其他 Bot 的白名单里 |
| 4 | Bot 自身 open_id 配置正确？ | 对应的 `.env` 中 `FEISHU_BOT_OPEN_ID` 是否正确 |
| 5 | 版本已发布？ | 开发者后台的修改必须「发布」后才生效 |


## 相关资源

- [Hermes Gateway 文档](https://hermes-agent.nousresearch.com/docs)
- [openclaw-feishu](https://github.com/AlexAnys/openclaw-feishu) — 飞书消息转发到 OpenClaw 的网关
- [Claude-to-IM](https://github.com/op7418/Claude-to-IM) — Claude Desktop 消息转发到 IM（飞书/微信等）
