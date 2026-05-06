# 星标说明
# ☆ = 理论/可选（很稳但理论上仍可能不出事）
# ★ = 血泪教训（不出事不可能）

★ `im:message:group_at_msg` 是独立权限——不是 `im:message` 的子权限。群消息收不到第一件事去飞书开发者后台确认这个权限有没有开。开了之后还要发布新版本。
★ Bot 的群 open_id 不能跨 App 提取。必须用 `bot/v3/info` API 获取。从其他 Bot 的 @mention 日志里抄 open_id 必跪。
★ allowed_users 必须包含群里所有人 + 所有 Bot 的 open_id。人和 Bot 各算一个，缺谁谁的消息被静默丢弃。
★ 事件推送哑死——TCP 连接活着的但群消息不推送。唯一解法：群设置里移除 Bot 再重新添加。改配置/重启都无效。所有框架（Hermes/OpenClaw/Claude-bridge）都会遇到。
★ 多 Bot 协作必须关流式输出（如框架支持）。开着的后果：A Bot 的回复片段出现在 B Bot 的回复里，串台。
★ 以为"只有 Hermes Gateway 有这些问题"是最大的误区——OpenClaw 和 feishu-claude-bridge 一样会踩中飞书侧的权限和哑死坑。这不是框架问题，是飞书 Bot 开发的共性问题。
☆ 群 open_id ≠ DM open_id。同一个人/Bot 在群聊和私聊中是两个不同的 ID。
★ 飞书原生 @所有人 对 Bot 无效——不会触发消息事件。需用约定关键词「所有人」或检测 raw JSON 中的 `@_all`。Hermes 已内置支持（`FEISHU_GROUP_POLICY=open` + `require_mention=true`），Bridge 需加代码检测 `rawContent.includes('@_all')`。OpenClaw 暂不支持。
