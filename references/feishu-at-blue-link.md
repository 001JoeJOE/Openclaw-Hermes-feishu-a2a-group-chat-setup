# 飞书 @ 蓝链技术细节

## 问题

Bot 发消息中的 `@名字` 显示为纯文本，而非蓝色可点击的 @ 链接。

## 根因

Hermes Gateway 的 `feishu.py` 中 `_build_outbound_payload` 方法根据内容是否包含 Markdown 语法决定消息格式：

- 含 Markdown → `msg_type="post"`（富文本格式）
- 不含 Markdown → `msg_type="text"`（纯文本格式）

纯文本格式的 `@名字` 只是字符串，飞书 API 不会渲染为蓝链。

## 解决方案（Hermes v0.11.0+）

### 代码改动（`~/.hermes/hermes-agent/gateway/platforms/feishu.py`）

**1. 新增反向缓存 `_name_to_open_id_cache`**（`__init__` 中）
```python
self._name_to_open_id_cache: Dict[str, str] = {}  # name → open_id
```

**2. 缓存更新点同步反向映射**（`_resolve_sender_name_from_api` 中）
```python
# 在 _sender_name_cache[oid] = (name, expire_at) 之后
if name:
    self._name_to_open_id_cache[name] = oid
```

**3. 修改 `_build_outbound_payload`**
```python
def _build_outbound_payload(self, content: str) -> tuple[str, str]:
    if _MARKDOWN_HINT_RE.search(content):
        return "post", _build_markdown_post_payload(content)
    if "@" in content:  # ← 新增
        return "post", self._build_mention_post_payload(content)
    text_payload = {"text": content}
    return "text", json.dumps(text_payload, ensure_ascii=False)
```

**4. 新增 `_build_mention_post_payload` 方法**

实现逻辑：
1. 用正则 `@([^@\s,，。！？!?、；;：:]+)` 匹配所有 `@名字` 模式
2. 在 `_name_to_open_id_cache` 中查找名字 → open_id
3. 命中 → 在 post 消息中注入 `{"tag": "at", "user_id": "ou_xxx"}` 元素
4. 未命中 → 保留为 `{"tag": "md", "text": "@名字"}` 纯文本元素
5. 文本和 `@` 元素在同行内保持正确顺序（同一 row 内的 md 和 at 元素横向排列）

### 飞书 post 消息格式

```json
{
  "zh_cn": {
    "content": [
      [
        {"tag": "md", "text": "前面的话 "},
        {"tag": "at", "user_id": "ou_xxxxx"},
        {"tag": "md", "text": " 后面的话"}
      ]
    ]
  }
}
```

## 缓存生命周期

| 事件 | 缓存更新 |
|------|---------|
| Bot 收到群消息 | 发送者的 open_id → name 写入 `_sender_name_cache`；name → open_id 写入 `_name_to_open_id_cache` |
| Bot 查询用户信息 | 同上 |
| 缓存 TTL 过期（10分钟） | 两缓存同时清除 |

## 限制

- 只有**曾出现在缓存中**的名字才能被渲染为蓝链
- 新加入群聊的用户/Bot，在第一次发言前无法被蓝链 @
- 建议在 A2A 群聊建立初期，手动让每个 Bot 在群里发一条消息（或让人 @ 每个 Bot 一次）来预热缓存

## 部署

> ⚠️ 安装后的 `feishu.py` 可能为 root 所有，无法直接编辑。通过临时文件法绕过：
> 1. `cp /path/to/feishu.py /home/joe/feishu_tmp.py`
> 2. 编辑 `/home/joe/feishu_tmp.py` 添加上述 4 处改动
> 3. `cp /home/joe/feishu_tmp.py /path/to/feishu.py`（覆盖原文件，所有权不变，jo 可写 `/home/joe/` 下的临时文件）
>
> 行号参考（v0.11.0）：`__init__` ~1377, `_resolve_sender_name_from_api` ~3593/3619, `_build_outbound_payload` ~3981, 新增方法紧跟其后 ~3995。各版本行号可能偏移，搜索函数名定位。

修改后重启 Gateway：
```bash
# 在 tmux 中 Ctrl+C 停止，然后重新启动
tmux send-keys -t hermes-gw C-c
sleep 2
tmux send-keys -t hermes-gw "cd ~/.hermes && hermes gateway start" Enter
```
