---
name: query-data
description: 向资产数据库提问，用自然语言查询分析数据，返回 SQL 报告和图表。用法：/query-data <问题>
allowed-tools: Bash
---

用户想对资产数据库提问，问题是：**$ARGUMENTS**

调用资产查询服务（通过主后端代理）。

**配置**
- 服务地址：`http://112.74.61.192:3300`
- 用法：`/query-data <user_id> <问题>`，例如 `/query-data cqp 上个月各渠道销售额`

## 第一步：发起查询

用 Bash 工具执行以下 Python 脚本（整段复制执行，问题文本已内联）：

```python
import json, urllib.request

args = """$ARGUMENTS""".strip().split(None, 1)
user_id = args[0] if args else "default"
question = args[1] if len(args) > 1 else ""

payload = json.dumps({
    "query": question,
    "user_id": user_id,
    "response_mode": "blocking",
    "drill": 0
}, ensure_ascii=False).encode("utf-8")

req = urllib.request.Request(
    "http://112.74.61.192:3300/api/langgraph/query",
    data=payload,
    headers={"Content-Type": "application/json"},
    method="POST"
)
try:
    with urllib.request.urlopen(req, timeout=120) as r:
        print(r.read().decode("utf-8"))
except Exception as e:
    print(json.dumps({"error": str(e), "status": "connection_failed"}))
```

## 第二步：解析并展示结果

根据响应 JSON 中的 `status` 字段处理：

**success / partial**
- 直接渲染 `final_answer` 字段内容（Markdown 格式）
- 用 SQL 代码块展示 `sql_text`
- 如果 `charts` 列表非空，提示「共有 N 张图表，请在前端界面查看可交互版本」
- 最后一行告知：`会话 ID：{conversation_id}（可在前端继续追问，或下次传入此 ID 续问）`

**needs_clarification**
- 展示 `clarification.questions` 中每道题的 `question` 文本和 `options` 选项列表
- 让用户选择后，用 `/query-data` 重新提问，或告知用户在前端点选选项
- 不要自行替用户选择

**error**
- 展示 `error` 字段内容
- 如果 `status` 为 `connection_failed`，提示用户检查网络是否可访问 `http://112.74.61.192:3300`，或联系管理员确认服务状态。
