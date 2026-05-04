# 飞书权限名映射表

> 飞书开发者后台 → 权限管理，搜索对应关键词即可找到。

## IM 消息权限（搜 `im:message`）

| 权限全名（飞书后台显示） | API 参数名 | 说明 |
|--------------------------|-----------|------|
| `im:message` | im:message | 🔸 根权限，勾选后包含所有子权限 |
| `im:message.group_msg` | im:message.group_msg | 接收群聊消息 |
| `im:message.group_at_msg.include_bot:readonly` | im:message.group_at_msg.include_bot | 获取群里 Bot 被 @ 的消息（含 Bot 自身） |
| `im:message.group_at_msg:readonly` | im:message.group_at_msg | 获取群里被 @ 的消息 |
| `im:message.group_msg:get_as_user` | im:message.group_msg:get_as_user | 以用户身份获取群聊消息 |
| `im:message.p2p_msg:readonly` | im:message.p2p_msg | 接收单聊消息 |
| `im:message:send_as_bot` | im:message:send_as_bot | 以应用身份发送消息 |
| `im:message:readonly` | im:message:readonly | 读取消息 |

> ⚠️ **格式不统一**：飞书后台中权限名混用点号 `.` 和冒号 `:` 分隔，且有些带 `:readonly` 后缀有些不带。以上是实际搜到的显示名。

## 其他权限

| 搜索关键词 | 权限全名 | 说明 |
|-----------|---------|------|
| `im:chat` | `im:chat:readonly` | 读取群聊/单聊信息 |
| `im:resource` | `im:resource` | 获取消息中的图片、文件 |
| `contact` | `contact:group:readonly` | 读取通讯录群组信息 |

## 搜索技巧

- 搜不到的权限直接搜**中文**，如 `消息`、`群`、`@`、`通讯录`
- 权限必须先**发布版本**、**审批通过**后才会在事件验证中生效
- 根权限 `im:message` 勾选后不需要单独勾选子权限
