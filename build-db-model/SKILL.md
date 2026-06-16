---
name: build-db-model
description: 对指定数据库启动语义建模流程（L1 剖析→L2 问答→L3 装配），生成数据库交接卡。用法：/build-db-model <数据库名>
allowed-tools: Bash
---

用户要对数据库 **$ARGUMENTS** 启动语义建模，调用资产查询服务（通过主后端代理）。

**配置**
- 服务地址：`http://112.74.61.192:3300`
- 会话隔离：用数据库名作为 `conversation_id`，各库互不干扰

整个流程分三阶段：
- **L1**：连库抽样，提炼库结构摘要
- **L2**：AI 规划问题，逐题一问一答收集业务语义（本 skill 负责启动并呈现第一题）
- **L3**：装配语义模型 YAML + 渲染交接卡 Markdown（所有问题回答完后自动触发）

## 第一步：启动建模会话

用 Bash 工具执行以下 Python 脚本，读取 SSE 流直到第一道问题出现：

```python
import json, subprocess

db_name = "$ARGUMENTS".strip()
result = subprocess.run(
    ["curl", "-s", "-N", "-X", "POST", "http://112.74.61.192:3300/api/dbmodel/chat/start",
     "-H", "Content-Type: application/json",
     "-d", json.dumps({"database_name": db_name, "conversation_id": db_name}, ensure_ascii=False)],
    capture_output=True, text=True, timeout=300
)

# 解析 SSE 事件
events = []
current = {}
for line in result.stdout.splitlines():
    if line.startswith("event:"):
        current["event"] = line[6:].strip()
    elif line.startswith("data:"):
        try:
            current["data"] = json.loads(line[5:].strip())
        except Exception:
            current["data"] = line[5:].strip()
    elif line == "" and current:
        events.append(current)
        current = {}
if current:
    events.append(current)

print(json.dumps(events, ensure_ascii=False, indent=2))
```

## 第二步：解析并展示进度与第一道题

遍历事件列表：

**progress 事件** (`event == "progress"`)：
- 展示为进度提示，格式：`[{data.stage}] {data.msg}`
- 例：`[L1] 连接数据库 lycx-asset-dev，开始抽样剖析…`

**question 事件** (`event == "question"`)：
这是需要用户回答的题目，按以下格式展示：

```
━━━ 问题 {data.id} ━━━
{data.text}

选项：
  A. {options[0].label}
  B. {options[1].label}
  C. {options[2].label}
  ...（如有更多选项）

请输入选项字母，或直接输入文字描述（输入 skip 跳过此题）
```

然后**等待用户回答**，不要替用户选择，不要继续执行。

**error 事件** (`event == "error"`)：
- 展示 `data.detail`
- 如果是连接失败，提示用户检查网络是否可访问 `http://112.74.61.192:3300`，或联系管理员确认服务状态。

## 第三步：用户回答后继续

用户给出答案后（字母选项或自由文字），调用 `/chat/reply` 接口继续流程：

```python
import json, subprocess

db_name = "$ARGUMENTS".strip()  # 用数据库名做会话隔离，与 /chat/start 保持一致
# 根据用户回答填写 choice_id（字母→选项序号）或 text（自由输入）
payload = {
    "conversation_id": db_name,
    # 如果用户选了字母（A/B/C...），转换为对应选项的 id（整数或字符串）
    "choice_id": "<选项id>",
    # 如果用户直接输入文字，则改用 text 字段，去掉 choice_id
    # "text": "<用户输入的文字>",
}
result = subprocess.run(
    ["curl", "-s", "-N", "-X", "POST", "http://112.74.61.192:3300/api/dbmodel/chat/reply",
     "-H", "Content-Type: application/json",
     "-d", json.dumps(payload, ensure_ascii=False)],
    capture_output=True, text=True, timeout=120
)
# 同第一步的方式解析 SSE 事件并打印
events = []
current = {}
for line in result.stdout.splitlines():
    if line.startswith("event:"):
        current["event"] = line[6:].strip()
    elif line.startswith("data:"):
        try:
            current["data"] = json.loads(line[5:].strip())
        except Exception:
            current["data"] = line[5:].strip()
    elif line == "" and current:
        events.append(current)
        current = {}
if current:
    events.append(current)
print(json.dumps(events, ensure_ascii=False, indent=2))
```

收到 reply 事件后：
- **ack 事件**：确认回答已记录，展示 `data.echo`（如「选项：直连业务库」）
- **question 事件**：展示下一道题，重复第二步格式，继续等待用户回答
- **card 事件**：所有问题回答完毕，L3 装配完成。展示 `data.content`（Markdown 格式的数据库交接卡）
- **done 事件**：展示建模完成摘要：`已回答 {data.answered} 题，待确认枚举项 {data.open_issues} 个，模型完成度 {data.completeness:.0%}`

如果用户输入 `skip`，将 `choice_id` 设为字符串 `"skip"`。
