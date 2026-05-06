# OpenClaw-Hermes-feishu-a2a-group-chat-setup

[![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

飞书群聊 A2A（Agent 2 Agent）模式完整配置指南 + 可执行 Hermes Agent Skill。

当多个 AI Agent Bot 在同一飞书群协作时，本仓库提供了从飞书开发者后台配置到 Bot 框架部署的完整方案。

## 支持的框架

| 框架 | 配置章节 | 说明 |
|------|----------|------|
| Hermes Gateway | Step 5 选项 A | `config.yaml` + `.env` |
| OpenClaw | Step 5 选项 B | `config.yaml` |
| feishu-claude-bridge | Step 5 选项 C | `config.json` / `config.yaml` |

不管用什么框架，**飞书开发者后台的权限配置是通用的**——本指南覆盖了最常踩的坑。

## 内容

| 路径 | 说明 |
|------|------|
| `LICENSE` | MIT 许可证 |
| `CHEATSHEET.md` | 星标知识点速查卡 |
| `skills/devops/feishu-a2a-group-chat-setup/SKILL.md` | Hermes Agent 可加载 skill（v2.2），含触发条件、分步操作、「所有人」全员唤醒、全框架配置、快速排查、常见坑点 |
| `.github/ISSUE_TEMPLATE/` | Issue 模板（提问题时自动收集环境信息） |

## 快速开始

**Hermes Agent 用户**——加载 skill 后直接问：
- "帮我配飞书 A2A"
- "OpenClaw Bot 在群里不回消息了"
- "群聊 A2A 收不到消息"

## 核心配置（3 分钟速览）

```
飞书开发者后台                  Bot 框架配置                   群设置
┌─────────────────┐          ┌────────────────────┐         ┌──────────┐
│ 权限:            │          │ bot_open_id        │         │ 添加 Bot │
│ group_at_msg ✅  │ ───────→ │ allowed_users      │ ─────→ │          │
│ 事件订阅 ✅       │          │ respond_to_bots: true│        └──────────┘
│ 已发布新版本 ✅    │          │ groupChat: enabled │
└─────────────────┘          └────────────────────┘
```

## 适用场景

- 任何飞书 Bot 框架的多 Bot 群聊协作
- Bot 在群里收不到 @消息 / 群聊不回 / A2A 配不起来

## 贡献

提 Issue 或 PR 前请先看：

- [Bot 群聊问题模板](.github/ISSUE_TEMPLATE/bot-group-chat-issue.md) — 报 Bug 时填写，自动收集环境信息和关键配置
- [功能建议模板](.github/ISSUE_TEMPLATE/feature-request.md) — 补充新框架支持或改进建议

## 许可证

[MIT](LICENSE) © 2026 Joe
