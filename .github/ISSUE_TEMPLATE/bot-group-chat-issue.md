---
name: Bot 群聊收不到消息 / A2A 配置问题
about: Bot 在群里收不到 @消息、群聊不回复、A2A 协作异常
title: "[群聊问题] "
labels: bug, group-chat
assignees: ''

---

## 描述问题

Bot 在群聊中有什么异常？收不到 @消息、收不到回复、还是串台？

## 你的环境

**Bot 框架：**
<!-- 删掉不适用的，留一个 -->
- [ ] Hermes Gateway
- [ ] OpenClaw
- [ ] feishu-claude-bridge
- [ ] 其他（请说明）

**框架版本：** （如 Hermes v0.11+ / OpenClaw v1.46+）

**部署方式：**
- [ ] Windows（WSL）
- [ ] Linux 服务器
- [ ] 其他

**Bot 数量：** 群里有几个 Bot？

## 飞书开发者后台确认

<!-- 请逐项确认 -->
- [ ] `im:message:group_at_msg` 权限已开启
- [ ] `im.message.receive_v1` 事件已订阅
- [ ] 权限修改后已**发布新版本**
- [ ] Bot 已添加到目标群

## 关键配置（脱敏后）

<!-- 请提供脱敏后的配置片段，隐去 app_secret -->

### FEISHU_BOT_OPEN_ID / bot.openId
```
来自哪个 API 获取的？  bot/v3/info / 其他 Bot 日志抄来的（选一个）
```

### allowed_users / FEISHU_ALLOWED_USERS
```
一共几个 ID？包含所有人和所有 Bot 吗？
```

### respond_to_bots 配置
```
true / false
```

## 框架日志片段

<!-- 请粘贴相关日志片段，尤其是包含以下关键词的行：
     inbound / group_policy_rejected / [WS] Connected / WebSocket disconnected -->

```
（粘贴日志片段）
```

## 排查步骤已试过

- [ ] 重启 Bot 进程
- [ ] 群设置中移除 Bot 重新添加
- [ ] 检查并确认 open_id 正确
- [ ] 检查 allowed_users 完整

## 补充说明

任何你觉得可能有帮助的信息。
