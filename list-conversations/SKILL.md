---
name: list-conversations
description: 列出当前用户的历史会话列表。用法：/list-conversations [数量]
allowed-tools: Bash
---

获取资产查询服务（通过主后端代理）中的历史会话列表。

**配置**
- 服务地址：`http://112.74.61.192:3300`
- 用法：`/list-conversations <user_id> [数量]`，例如 `/list-conversations cqp 20`

## 执行查询

用 Bash 工具执行以下 Python 脚本：

```python
import json, urllib.request, urllib.parse

parts = """$ARGUMENTS""".strip().split()
user_id = parts[0] if parts else "default"
limit = int(parts[1]) if len(parts) > 1 and parts[1].isdigit() else 20

params = urllib.parse.urlencode({"user_id": user_id, "limit": limit})
req = urllib.request.Request(
    f"http://112.74.61.192:3300/api/langgraph/conversations?{params}",
    method="GET"
)
try:
    with urllib.request.urlopen(req, timeout=30) as r:
        print(r.read().decode("utf-8"))
except Exception as e:
    print(json.dumps({"error": str(e)}))
```

## 展示结果

将返回的 JSON 数组渲染为 Markdown 表格：

| # | 标题 | 状态 | 消息数 | 最近更新 |
|---|------|------|--------|---------|
| 1 | ...  | ...  | ...    | ...     |

字段映射：
- `title` → 标题（截取前 40 字）
- `last_status` 翻译：`success`=成功、`partial`=部分成功、`error`=出错、`running`=进行中、`needs_clarification`=待澄清
- `message_count` → 消息数
- `updated_at` → 最近更新（只显示日期+时间，不显示秒）
- `conversation_id` → 展示完整 ID，供用户在前端或后续命令中引用

如果列表为空，提示「暂无历史会话，使用 /query-data <问题> 开始第一次查询」。

如果请求失败（error 字段存在），提示用户检查网络是否可访问 `http://112.74.61.192:3300`，或联系管理员确认服务状态。
