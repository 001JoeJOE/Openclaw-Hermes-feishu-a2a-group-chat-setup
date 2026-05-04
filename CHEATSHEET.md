# 星标说明
# ☆ = 理论/可选（很稳但理论上仍可能不出事）
# ★ = 血泪教训（不出事不可能）

★ `im:message:group_at_msg` 是独立权限——不是 `im:message` 的子权限。群消息收不到第一件事去飞书开发者后台确认这个权限有没有开。开了之后还要发布新版本。
★ Bot-to-Bot 通信要两处都配，缺一不可：`.env` 里 `FEISHU_ALLOW_BOTS=mentions`（Gateway 入口过滤）+ `config.yaml` 里 `respond_to_bots: true`（飞书适配器层）。只配一处时日志可能不报错但不工作是静默的。而且所有 profile 都要配，漏一个 profile 则那个 gateway 进程仍然丢弃 Bot 消息。
★ 新加群要在两处白名单都加 chat_id：
   - `config.yaml` → `channels.feishu.accounts.*.group_chats`（群 ID 白名单）
   - `config.yaml` → `platforms.feishu.extra.group_chat_allowlist`（额外群聊允许列表）
   只加一处时，群内 @ 消息被静默丢弃，日志不报任何错误。
★ Bot 的 open_id 不能跨 App 互用。同一个 Bot 在 App A 视角下的 open_id ≠ App B 视角下的。A 的配置里要配 B 的 ID 时，必须从 A 自己的日志里取 B 的 sender.open_id，不能拿 B 自报的。
   - **场景A — 获取自身 Bot 的 open_id**：必须用 `bot/v3/info` API 获取（也是最终写入自己配置的那个 ID）
   - **场景B — 获取其他 Bot 的 open_id**：从自己的 Gateway 日志中，其他 Bot 发来的 @mention 消息里抄 `sender.open_id`（自己视角下的对方 ID 才有效）
   - **场景C — 获取用户的 open_id**：用户发消息时，从日志抄 `sender.open_id`
★ allowed_users 必须包含群里所有人 + 所有 Bot 的 open_id。人和 Bot 各算一个，缺谁谁的消息被静默丢弃。
★ 事件推送哑死——TCP 连接活着的但群消息不推送。唯一解法：群设置里移除 Bot 再重新添加。改配置/重启都无效。所有框架（Hermes/OpenClaw/Claude-bridge）都会遇到。
★ 多 Bot 协作必须关流式输出（如框架支持）。开着的后果：A Bot 的回复片段出现在 B Bot 的回复里，串台。
★ 以为"只有 Hermes Gateway 有这些问题"是最大的误区——OpenClaw 和 feishu-claude-bridge 一样会踩中飞书侧的权限和哑死坑。这不是框架问题，是飞书 Bot 开发的共性问题。
☆ 群 open_id ≠ DM open_id。同一个人/Bot 在群聊和私聊中是两个不同的 ID。
