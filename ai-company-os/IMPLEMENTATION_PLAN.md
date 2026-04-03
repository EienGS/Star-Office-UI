# AI Company OS - 实施路径文档

> 版本：v1.0
> 状态：待执行
> 设计决策参考：DESIGN_DECISIONS.md

---

## 执行总则（Claude Code 必读）

**在开始任何任务前，必须完整阅读本章节。**

### 项目边界（严格遵守）

- 本系统全部代码位于 `ai-company-os/` 目录下
- 原项目目录（`backend/`、`frontend/`、`docs/`）**禁止修改**，仅允许只读参考
- 风格继承方式：直接复制原项目的 CSS 变量、字体声明、Phaser.js 场景逻辑，不引用原项目文件路径

### 禁止事项

- 禁止修改原项目任何文件
- 禁止安装未在本文档明确列出的依赖
- 禁止自行设计数据结构（以 DESIGN_DECISIONS.md 第六章为准）
- 禁止在全局 `~/.claude/settings.json` 写入任何内容
- 禁止跳过验证步骤直接进入下一阶段

### 每个任务的执行规范

1. 完整阅读当前任务的所有要求
2. 阅读"前置确认"章节，逐项确认
3. 按照"实现步骤"逐步执行
4. 完成后逐项执行"验证步骤"，全部通过后填写"自检 Checklist"
5. Checklist 全部勾选后才可提交，不得遗漏任何一项

---

## 阶段总览

| 阶段 | 名称 | 核心产出 | 依赖阶段 |
|------|------|---------|---------|
| 阶段1 | 文件系统协议初始化 | `/office/` 目录结构 + 初始配置文件 | 无 |
| 阶段2 | FastAPI 后端核心 | 状态管理 + Agent/项目管理 + SSE + 文件监听 | 阶段1 |
| 阶段3 | Hook 脚本与 CLAUDE.md 生成器 | Agent 侧接入能力 | 阶段2 |
| 阶段4 | 前端主框架 | 楼层切换 + 像素场景 + 侧边面板 + 底部流水线 | 阶段2 |
| 阶段5 | CEO 指令输入 | 完整输入能力 + 附件传递 + 回执展示 | 阶段4 |
| 阶段6 | 知识库与归档系统 | 索引机制 + 归档触发 + 档案室 UI | 阶段4 |
| 阶段7 | 集成测试与联调 | 完整流程验证 | 全部阶段 |

---

## 目录结构规范（实施前确认）

```
/ai-company-os/
├── backend/
│   ├── main.py                  # FastAPI 入口
│   ├── routers/
│   │   ├── agents.py            # Agent 管理 API
│   │   ├── projects.py          # 项目管理 API
│   │   ├── messages.py          # 消息 API
│   │   ├── knowledge.py         # 知识库 API
│   │   └── system.py            # 健康检查 + SSE
│   ├── services/
│   │   ├── agent_service.py     # Agent 创建/销毁逻辑
│   │   ├── message_service.py   # 消息构建/发送逻辑
│   │   ├── scheduler.py         # 定时任务（超时检查等）
│   │   └── watcher.py           # 文件监听服务
│   ├── models/
│   │   └── schemas.py           # Pydantic 数据模型
│   └── requirements.txt
│
├── frontend/
│   ├── index.html               # 主入口
│   ├── css/
│   │   ├── base.css             # 继承自原项目的像素风格变量
│   │   ├── layout.css           # 整体布局
│   │   ├── floor.css            # 楼层/平面图样式
│   │   ├── panel.css            # 侧边面板样式
│   │   └── pipeline.css         # 底部流水线样式
│   └── js/
│       ├── main.js              # 主入口，初始化
│       ├── scene/
│       │   ├── FloorScene.js    # 楼层场景基类（继承原项目 Phaser 逻辑）
│       │   ├── F1Scene.js       # F1 公共层
│       │   ├── B1Scene.js       # B1 基础设施层
│       │   └── ProjectScene.js  # 项目楼层（动态生成）
│       ├── components/
│       │   ├── TopBar.js        # 顶部栏
│       │   ├── SidePanel.js     # 侧边面板
│       │   ├── Pipeline.js      # 底部流水线
│       │   └── DirectiveInput.js# CEO 指令输入
│       └── api/
│           └── client.js        # API 调用 + SSE 连接
│
├── hooks/                       # 共享 Hook 脚本
│   ├── pre_tool_use.py
│   ├── post_tool_use.py
│   └── on_notification.py
│
├── templates/
│   ├── CLAUDE.md.tpl            # CLAUDE.md 模板
│   └── settings.json.tpl        # .claude/settings.json 模板
│
├── office/                      # 文件系统协议目录（运行时数据）
│   ├── config/
│   ├── inbox/
│   ├── tasks/
│   ├── knowledge/
│   ├── resources/
│   ├── state/
│   └── attachments/
│
├── scripts/
│   └── init_office.py           # 初始化脚本
│
└── DESIGN_DECISIONS.md
```

---

## 阶段1：文件系统协议初始化

### 任务描述

创建 `/office/` 目录的完整结构，并写入所有初始配置文件。这是整个系统的数据基础，所有后续阶段都依赖此结构。

### 前置确认

- [ ] 当前工作目录为 `ai-company-os/`
- [ ] `office/` 目录不存在或为空
- [ ] 已阅读 DESIGN_DECISIONS.md 第六章（File System Protocol）

### 实现步骤

**步骤1：创建 init_office.py 脚本**

文件路径：`ai-company-os/scripts/init_office.py`

脚本必须完成以下操作：

1. 创建完整目录结构（参考目录规范）：
   ```
   office/config/
   office/inbox/ceo/
   office/tasks/_templates/
   office/knowledge/archive/
   office/knowledge/context/
   office/knowledge/skills/
   office/knowledge/sops/
   office/resources/mcp/
   office/resources/cloud/
   office/resources/db/
   office/state/receipts/
   office/attachments/
   ```

2. 写入初始配置文件，内容严格按照 DESIGN_DECISIONS.md 第六章的数据结构：

   `office/config/agents.json`：
   ```json
   { "agents": [] }
   ```

   `office/config/projects.json`：
   ```json
   { "projects": [] }
   ```

   `office/config/resources.json`：
   ```json
   {
     "mcp": [],
     "cloud": [],
     "db": []
   }
   ```

   `office/knowledge/archive/_index.json`：
   ```json
   { "archives": [] }
   ```

   `office/knowledge/skills/_index.json`：
   ```json
   { "skills": [] }
   ```

   `office/knowledge/sops/_index.json`：
   ```json
   { "sops": [] }
   ```

3. 脚本必须是幂等的（重复执行不报错，不覆盖已有数据文件，只创建缺失的目录和文件）

**步骤2：执行脚本**

```bash
cd ai-company-os
python scripts/init_office.py
```

### 验证步骤

执行以下检查，每项必须通过：

**V1-1**：验证目录结构完整性
```bash
find ai-company-os/office -type d | sort
```
预期输出必须包含：`config`、`inbox/ceo`、`tasks/_templates`、`knowledge/archive`、`knowledge/context`、`knowledge/skills`、`knowledge/sops`、`resources/mcp`、`resources/cloud`、`resources/db`、`state/receipts`、`attachments`

**V1-2**：验证 JSON 文件合法性
```bash
python -c "
import json, glob
for f in glob.glob('ai-company-os/office/**/*.json', recursive=True):
    json.load(open(f))
    print(f'OK: {f}')
"
```
预期：所有 .json 文件输出 OK，无报错

**V1-3**：验证幂等性
```bash
python scripts/init_office.py
python scripts/init_office.py
echo "exit code: $?"
```
预期：两次执行均无报错，exit code: 0

### 自检 Checklist

完成后逐项确认，全部勾选才可进入阶段2：

- [ ] V1-1 目录结构验证通过
- [ ] V1-2 JSON 文件合法性验证通过
- [ ] V1-3 幂等性验证通过
- [ ] 未修改原项目任何文件
- [ ] 未向 office/ 之外的目录写入任何文件

---

## 阶段2：FastAPI 后端核心

### 任务描述

实现 Office Server，提供 REST API、SSE 实时推送、文件监听、定时调度四个核心能力。

### 前置确认

- [ ] 阶段1 自检 Checklist 全部通过
- [ ] `office/config/agents.json` 存在且内容为 `{"agents":[]}`
- [ ] 已阅读 DESIGN_DECISIONS.md 第十二章（Office Server 设计）

### 依赖安装

`ai-company-os/backend/requirements.txt`：
```
fastapi==0.115.0
uvicorn[standard]==0.30.0
watchfiles==0.24.0
pydantic==2.9.0
python-multipart==0.0.12
aiofiles==24.1.0
```

### 数据模型（schemas.py）

严格按照 DESIGN_DECISIONS.md 第六章定义，使用 Pydantic v2：

```python
# 所有 ISO8601 时间字段使用 datetime 类型
# 所有 Optional 字段默认值为 None
# 枚举值使用 Literal 类型约束

AgentStatus = Literal["idle", "working", "waiting", "error"]
AgentType = Literal["project", "resident"]
MessageType = Literal["task_request", "task_notify", "shutdown", 
                       "ceo_directive", "broadcast", "context_update", "escalation"]
MessagePriority = Literal["HIGH", "NORMAL"]
MessageStatus = Literal["unread", "acknowledged", "done", "failed"]
TaskStatus = Literal["pending", "in_progress", "done", "failed", "interrupted"]
ReceiptStatus = Literal["acknowledged", "done", "failed"]
```

### API 实现要求

**agents.py 路由**

`GET /agents`
- 读取 `office/config/agents.json`
- 返回完整 agents 列表
- 如文件不存在返回 `{"agents": []}`

`POST /agents`
- 请求体字段：name, type, role, projectId(optional), workBaseDir, customInstructions(optional), targetRepos(optional)
- 生成 agentId：格式为 `agent-[name的拼音/英文]-[4位随机字符]`，全小写，连字符分隔
- 创建工作目录：`{workBaseDir}/{agentId}/`
- 创建 `.claude/` 子目录
- 调用 agent_service.generate_claude_md() 写入 CLAUDE.md
- 调用 agent_service.generate_settings_json() 写入 .claude/settings.json
- 注册到 agents.json
- 创建 `office/inbox/{agentId}/` 目录
- 写入 `office/state/{agentId}.json` 初始状态（status: "idle"）
- 通过 SSE 广播 `{"event": "agent_created", "agentId": agentId}`
- 返回创建成功的 Agent 完整数据

`PATCH /agents/{agentId}`
- 允许更新字段：name, role, projectId, customInstructions, targetRepos, uiPosition
- 更新 agents.json 对应条目
- 通过 SSE 广播 `{"event": "agent_updated", "agentId": agentId}`
- 不重新生成 CLAUDE.md（动态配置从 agents.json 读取）

`POST /agents/{agentId}/shutdown`
- 查询该 Agent 是否有 status 为 in_progress 的任务
- 如有：返回 `{"warning": true, "activeTasks": [...任务列表...]}` 并停止，等待前端二次确认
- 请求体携带 `{"force": true}` 时：向 `office/inbox/{agentId}/` 写入 shutdown 消息（priority: HIGH）
- 更新 agents.json 中该 Agent 的 status 为 "shutdown_pending"
- 通过 SSE 广播 `{"event": "agent_shutdown_pending", "agentId": agentId}`

**projects.py 路由**

`GET /projects`
- 读取 `office/config/projects.json` 返回列表

`POST /projects`
- 请求体字段：name, type(自由文本), stages(数组，至少2项), assignedAgents(optional), residentAgents(optional)
- 生成 projectId：格式为 `proj-[4位随机字符]`
- 写入 projects.json
- 创建 `office/tasks/{projectId}/` 目录
- 写入 `office/tasks/{projectId}/board.json`：`{"projectId": ..., "stages": [...], "tasks": []}`
- 创建 `office/knowledge/context/{projectId}/` 目录
- 写入 `office/knowledge/context/{projectId}/_index.json`：`{"projectId": ..., "summaries": []}`
- 通过 SSE 广播 `{"event": "project_created", "projectId": projectId}`

`GET /projects/{projectId}/board`
- 读取 `office/tasks/{projectId}/board.json`
- 合并每个 taskId 对应的 taskId.json 数据
- 返回按阶段分组的完整看板数据

**messages.py 路由**

`POST /messages`
- 请求体：to(agentId 或 "broadcast"), subject, content, priority(默认HIGH), projectId(optional), taskId(optional), attachments(文件列表)
- 生成 msgId：`msg-{timestamp}-{random3}`
- 保存附件到 `office/attachments/{msgId}/`
- 构建消息 JSON（严格按照第六章消息格式），from 固定为 "ceo"
- 写入 `office/inbox/{to}/{msgId}.json`（broadcast 写入 `office/inbox/broadcast/{msgId}.json`）
- 通过 SSE 广播 `{"event": "message_sent", "msgId": msgId, "to": to}`

`GET /messages/{agentId}`
- 读取 `office/inbox/{agentId}/` 目录所有 .json 文件
- 按 createdAt 倒序排列
- 返回消息列表（不返回消息正文，只返回 msgId, from, subject, priority, status, createdAt）
- 正文需要单独请求（节约 Token 原则）

`GET /messages/{agentId}/{msgId}`
- 读取具体消息完整内容

`POST /receipts/{msgId}`
- 请求体：status(acknowledged/done/failed), result(optional)
- 写入 `office/state/receipts/{msgId}.json`
- 更新对应消息文件的 status 字段
- 通过 SSE 广播 `{"event": "receipt_updated", "msgId": msgId, "status": status}`

`GET /receipts/{msgId}`
- 读取 `office/state/receipts/{msgId}.json`

**system.py 路由**

`GET /health`
- 检查 office/ 目录是否存在
- 检查 config/agents.json 是否可读
- 返回 `{"status": "ok", "officeRoot": "...", "agentCount": N}`

`GET /events`（SSE）
- 使用 FastAPI 的 StreamingResponse + asyncio.Queue 实现
- 客户端连接后保持长连接
- 所有后端事件通过此通道推送
- 心跳：每30秒发送一次 `{"event": "ping"}`

**state 路由（供 Hook 脚本调用）**

`POST /state/{agentId}`
- 请求体：status, detail(optional), currentTask(optional), area(optional)
- 写入 `office/state/{agentId}.json`
- 通过 SSE 广播 `{"event": "state_updated", "agentId": agentId, "state": {...}}`

`GET /state`
- 读取 `office/state/` 目录所有 agentId.json（排除 receipts/ 子目录）
- 返回所有 Agent 当前状态的汇总

**knowledge.py 路由**

`GET /knowledge/archive`
- 读取 `office/knowledge/archive/_index.json`
- 返回归档索引列表（不返回归档正文）

`GET /knowledge/archive/{filename}`
- 读取具体归档 Markdown 文件内容

`GET /knowledge/context/{projectId}`
- 读取 `office/knowledge/context/{projectId}/_index.json`

`GET /resources`
- 读取 `office/config/resources.json`

### 文件监听服务（watcher.py）

使用 `watchfiles` 库监听 `office/` 目录，处理以下事件：

```python
监听路径: office/state/  （排除 receipts/ 子目录）
事件类型: 文件修改
触发动作: SSE 广播 state_updated

监听路径: office/inbox/  （所有子目录）
事件类型: 文件新增
触发动作: 
  1. 读取消息文件，检查 priority
  2. 记录消息 createdAt 和 timeoutAt 到内存字典（用于超时检查）
  3. SSE 广播 new_message

监听路径: office/state/receipts/
事件类型: 文件新增
触发动作:
  1. 读取回执文件，获取 msgId
  2. 从超时检查字典中移除该 msgId
  3. SSE 广播 receipt_updated

监听路径: office/tasks/
事件类型: 文件修改
触发动作: SSE 广播 task_updated

监听路径: office/knowledge/archive/
事件类型: 文件新增（排除 _index.json）
触发动作:
  1. 读取对应的 shutdown 回执确认
  2. 从 agents.json 移除该 Agent
  3. 删除 office/state/{agentId}.json
  4. SSE 广播 agent_removed
```

### 定时调度（scheduler.py）

使用 `asyncio` 实现两个定时任务：

**任务1：消息超时检查（每60秒）**
```
遍历内存中记录的待回执消息
对每条消息：
  if 当前时间 > timeoutAt and 无对应回执文件:
    构建 escalation 消息
    写入 office/inbox/ceo/{msgId}-escalation.json
    SSE 广播 escalation 事件
    从超时检查字典中移除（避免重复告警）
```

**任务2：Agent 状态超时检查（每30秒）**
```
遍历 office/state/ 目录所有 agentId.json
对每个 Agent：
  if Agent.status != "idle" and 当前时间 - updatedAt > 300秒:
    更新 state 文件：status = "idle", detail = "状态超时，自动降级"
    SSE 广播 state_updated
```

### 主入口（main.py）

```python
# 启动时：
# 1. 验证 office/ 目录存在，否则打印错误提示并退出
# 2. 启动文件监听服务（后台任务）
# 3. 启动定时调度（后台任务）
# 4. 监听端口：18792（避免与原项目 18791 冲突）

# CORS：允许所有来源（本地开发环境）
# 所有路由前缀：/api/v1
```

### 验证步骤

**V2-1**：依赖安装验证
```bash
cd ai-company-os/backend
pip install -r requirements.txt
echo "exit: $?"
```
预期：exit: 0，无报错

**V2-2**：服务启动验证
```bash
cd ai-company-os/backend
uvicorn main:app --port 18792 &
sleep 3
curl http://localhost:18792/api/v1/health
```
预期返回：`{"status": "ok", ...}`

**V2-3**：Agent 创建流程验证
```bash
curl -X POST http://localhost:18792/api/v1/agents \
  -H "Content-Type: application/json" \
  -d '{
    "name": "测试Agent",
    "type": "project",
    "role": "测试用途",
    "workBaseDir": "/tmp/office-test-agents"
  }'
```
预期：
- 返回包含 agentId 的 JSON
- `/tmp/office-test-agents/{agentId}/CLAUDE.md` 文件存在
- `/tmp/office-test-agents/{agentId}/.claude/settings.json` 文件存在
- `office/config/agents.json` 中包含该 Agent
- `office/inbox/{agentId}/` 目录存在
- `office/state/{agentId}.json` 存在且 status 为 "idle"

**V2-4**：项目创建流程验证
```bash
curl -X POST http://localhost:18792/api/v1/projects \
  -H "Content-Type: application/json" \
  -d '{
    "name": "测试项目",
    "type": "测试",
    "stages": ["待认领", "进行中", "已完成"]
  }'
```
预期：
- 返回包含 projectId 的 JSON
- `office/tasks/{projectId}/board.json` 存在
- `office/knowledge/context/{projectId}/_index.json` 存在

**V2-5**：SSE 连接验证
```bash
curl -N http://localhost:18792/api/v1/events &
sleep 2
curl -X POST http://localhost:18792/api/v1/state/agent-test-0001 \
  -H "Content-Type: application/json" \
  -d '{"status": "working", "detail": "测试中"}'
sleep 2
```
预期：SSE 流中出现包含 `state_updated` 的事件数据

**V2-6**：shutdown 警告机制验证
```bash
# 先手动写入一个 in_progress 任务
# 然后调用 shutdown，验证返回 warning
curl -X POST http://localhost:18792/api/v1/agents/{agentId}/shutdown
```
预期：返回 `{"warning": true, "activeTasks": [...]}`，不发送 shutdown 消息

**V2-7**：定时任务验证（状态超时）
```bash
# 写入一个超过5分钟前的 state 文件
python -c "
import json
from datetime import datetime, timedelta
state = {'agentId': 'test', 'status': 'working', 'detail': 'test', 
         'updatedAt': (datetime.now() - timedelta(minutes=6)).isoformat()}
json.dump(state, open('ai-company-os/office/state/test.json', 'w'))
"
sleep 35
cat ai-company-os/office/state/test.json | python -c "import sys,json; d=json.load(sys.stdin); assert d['status']=='idle', 'FAIL: status not idle'"
echo "V2-7 PASS"
```
预期：输出 "V2-7 PASS"

### 自检 Checklist

- [ ] V2-1 依赖安装验证通过
- [ ] V2-2 服务启动验证通过
- [ ] V2-3 Agent 创建流程验证通过（包括目录、文件、注册）
- [ ] V2-4 项目创建流程验证通过
- [ ] V2-5 SSE 连接验证通过
- [ ] V2-6 shutdown 警告机制验证通过
- [ ] V2-7 定时任务验证通过
- [ ] 所有 API 返回的 JSON 字段与 DESIGN_DECISIONS.md 第六章数据结构一致
- [ ] 端口为 18792，未占用原项目 18791 端口
- [ ] 未修改原项目任何文件

---

## 阶段3：Hook 脚本与 CLAUDE.md 生成器

### 任务描述

实现共享 Hook 脚本和 CLAUDE.md/settings.json 生成器，这是 Agent 侧接入系统的完整实现。

### 前置确认

- [ ] 阶段2 自检 Checklist 全部通过
- [ ] `http://localhost:18792/api/v1/health` 可正常访问
- [ ] 已阅读 DESIGN_DECISIONS.md 第十一章、第十四章

### 实现步骤

**步骤1：pre_tool_use.py**

文件路径：`ai-company-os/hooks/pre_tool_use.py`

逻辑：
```
读取环境变量 OFFICE_AGENT_ID 和 OFFICE_ROOT
如果环境变量缺失：静默退出（不影响 Claude Code 正常工作）

推送状态：
  POST {OFFICE_SERVER_URL}/api/v1/state/{OFFICE_AGENT_ID}
  body: {"status": "working", "detail": "使用工具: {tool_name}"}
  如果请求失败：静默忽略，不抛出异常（Agent 工作不能因系统不可用而中断）

检查 HIGH 优先级消息：
  扫描 {OFFICE_ROOT}/inbox/{OFFICE_AGENT_ID}/ 目录
  读取所有 status=unread 且 priority=HIGH 的消息
  如有：将消息内容写入标准输出（Claude Code 会读取此输出）
  格式：
  ---[OFFICE-HIGH-PRIORITY-MESSAGE]---
  来自: {from}
  主题: {subject}
  内容: {完整消息JSON}
  ---[END]---
  
  将消息 status 更新为 acknowledged

OFFICE_SERVER_URL 默认值：http://localhost:18792
可通过环境变量 OFFICE_SERVER_URL 覆盖
```

**步骤2：post_tool_use.py**

文件路径：`ai-company-os/hooks/post_tool_use.py`

逻辑：
```
读取环境变量 OFFICE_AGENT_ID 和 OFFICE_ROOT
如果环境变量缺失：静默退出

推送状态更新：
  POST {OFFICE_SERVER_URL}/api/v1/state/{OFFICE_AGENT_ID}
  body: {"status": "working", "detail": "工具调用完成", "updatedAt": 当前时间ISO8601}
  如果请求失败：静默忽略
```

**步骤3：on_notification.py**

文件路径：`ai-company-os/hooks/on_notification.py`

逻辑：
```
读取环境变量 OFFICE_AGENT_ID 和 OFFICE_ROOT
如果环境变量缺失：静默退出

读取通知类型（从 stdin 或环境变量获取）

如果通知类型为 task_complete：
  1. 处理 NORMAL 优先级收件箱：
     扫描 {OFFICE_ROOT}/inbox/{OFFICE_AGENT_ID}/ 目录
     读取所有 status=unread 且 priority=NORMAL 的消息
     将消息写入标准输出（同 pre_tool_use 格式）
     更新消息 status 为 acknowledged
  
  2. 推送 idle 状态：
     POST {OFFICE_SERVER_URL}/api/v1/state/{OFFICE_AGENT_ID}
     body: {"status": "idle", "detail": "任务完成，等待新指令"}

如果通知类型为 error：
  1. 读取错误信息
  2. 推送 error 状态：
     POST {OFFICE_SERVER_URL}/api/v1/state/{OFFICE_AGENT_ID}
     body: {"status": "error", "detail": "发生错误: {error_info前100字符}"}
  3. 写入 escalation 消息：
     构建 escalation 类型消息，写入 {OFFICE_ROOT}/inbox/ceo/
```

**步骤4：CLAUDE.md 模板（templates/CLAUDE.md.tpl）**

```markdown
# Agent 操作规范

## 身份标识
- Agent ID: {{AGENT_ID}}
- 系统配置文件: {{OFFICE_ROOT}}/config/agents.json

**每次开始新任务时，必须先读取系统配置文件，获取当前角色、项目绑定等最新信息。**

---

## 任务开始前（强制执行）

1. 读取 {{OFFICE_ROOT}}/config/agents.json 找到本 Agent 的配置
2. 读取 {{OFFICE_ROOT}}/inbox/{{AGENT_ID}}/ 处理未读消息
3. 如配置中有 projectId，读取 {{OFFICE_ROOT}}/tasks/{projectId}/board.json
4. 如需项目背景知识：
   - **先读** {{OFFICE_ROOT}}/knowledge/context/{projectId}/_index.json
   - 根据关键词过滤，**只读** 命中的 1~2 个摘要文件
   - **禁止**一次性读取整个目录

---

## 任务执行中

- 收到 priority=HIGH 消息：当前工具执行完毕后立即处理，不得推迟
- 状态推送由 Hook 脚本自动完成，无需手动处理

---

## 任务完成后（强制执行）

1. 更新任务文件：{{OFFICE_ROOT}}/tasks/{projectId}/{taskId}.json
   - status 改为 "done"
   - output.result 填写产出摘要
2. 写任务摘要到 {{OFFICE_ROOT}}/knowledge/context/{projectId}/{taskId}-summary.md
   - 格式：只写"产出"、"可复用结论"、"踩坑"三部分
   - 每条不超过2行，只写结论不写过程
3. 更新上下文索引 {{OFFICE_ROOT}}/knowledge/context/{projectId}/_index.json
4. 检查任务的 nextAgent 字段，有则发送 task_request 消息

---

## 跨 Agent 消息发送规范

消息文件路径：{{OFFICE_ROOT}}/inbox/{targetAgentId}/msg-{timestamp}-{random3}.json

必须包含的字段：
- from: "{{AGENT_ID}}"
- context.projectBrief: 项目背景摘要（不超过200字）
- context.taskId: 关联任务ID
- context.handoffReason: 交接原因
- context.expectedOutput: 期望产出
- timeoutAt: 当前时间 +1小时（ISO8601格式）

发送后：轮询 {{OFFICE_ROOT}}/state/receipts/{msgId}.json
超过 timeoutAt 无回执：写入 escalation 消息到 {{OFFICE_ROOT}}/inbox/ceo/

---

## 收到 shutdown 消息时（强制执行）

1. 完成当前最小工作单元（不得强行中断到一半）
2. 将所有 in_progress 任务的 status 改为 "interrupted"
3. 在每个中断任务的 interruptedContext 字段填写：
   - 当前进度描述
   - 已产出文件列表
   - 待完成内容
   - 建议接手方要求
4. 生成归档文件：{{OFFICE_ROOT}}/knowledge/archive/{{AGENT_ID}}-{today}.md
5. 更新归档索引：{{OFFICE_ROOT}}/knowledge/archive/_index.json
6. 写回执：{{OFFICE_ROOT}}/state/receipts/{shutdown_msgId}.json
   内容：{"status": "done", "respondedAt": "当前时间ISO8601"}
7. **不得直接退出 Claude Code，等待回执确认**

---

## 知识库查询禁令（Token 节约原则）

- **禁止**不读 _index.json 直接扫描目录
- **禁止**一次性读取整个知识库目录
- **禁止**将归档全文保留在对话上下文中
- 查询步骤：读 _index.json → 过滤关键词 → 只读 1~2 个目标文件

---

## 公共资源（只读）

- MCP 工具列表：{{OFFICE_ROOT}}/resources/mcp/registry.json
- 知识库入口：{{OFFICE_ROOT}}/knowledge/{skills|sops}/_index.json
- 云资源配置：{{OFFICE_ROOT}}/resources/cloud/（只读，不得修改）
- 数据库连接：{{OFFICE_ROOT}}/resources/db/（只读，不得修改）

---

## 目标代码仓库
{{TARGET_REPOS}}

## 角色补充说明
{{CUSTOM_INSTRUCTIONS}}
```

**步骤5：settings.json 模板（templates/settings.json.tpl）**

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "python {{OFFICE_ROOT}}/.hooks/pre_tool_use.py"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "python {{OFFICE_ROOT}}/.hooks/post_tool_use.py"
          }
        ]
      }
    ],
    "Notification": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "python {{OFFICE_ROOT}}/.hooks/on_notification.py"
          }
        ]
      }
    ]
  },
  "env": {
    "OFFICE_AGENT_ID": "{{AGENT_ID}}",
    "OFFICE_ROOT": "{{OFFICE_ROOT}}",
    "OFFICE_SERVER_URL": "{{OFFICE_SERVER_URL}}"
  }
}
```

**步骤6：将 hooks/ 目录复制到 office/.hooks/**

```python
# 在 init_office.py 中增加：
# 将 ai-company-os/hooks/ 软链接或复制到 ai-company-os/office/.hooks/
# 使用复制而非软链接，避免路径问题
```

**步骤7：在 agent_service.py 中实现生成逻辑**

```python
def generate_claude_md(agent_id, office_root, office_server_url, 
                       target_repos, custom_instructions):
    # 读取 templates/CLAUDE.md.tpl
    # 替换所有 {{变量}}
    # target_repos 格式化为 "- /path/to/repo" 列表
    # 如 custom_instructions 为空，删除该章节
    # 返回字符串

def generate_settings_json(agent_id, office_root, office_server_url):
    # 读取 templates/settings.json.tpl
    # 替换所有 {{变量}}
    # 返回字符串（合法 JSON）
```

### 验证步骤

**V3-1**：Hook 脚本环境变量缺失时静默退出
```bash
python ai-company-os/hooks/pre_tool_use.py
echo "exit: $?"
```
预期：exit: 0，无报错输出

**V3-2**：Hook 脚本正常工作验证
```bash
export OFFICE_AGENT_ID="test-agent"
export OFFICE_ROOT="$(pwd)/ai-company-os/office"
export OFFICE_SERVER_URL="http://localhost:18792"
python ai-company-os/hooks/pre_tool_use.py
```
预期：
- 无报错
- `office/state/test-agent.json` 被更新（如果后端服务在运行）

**V3-3**：CLAUDE.md 生成验证
```bash
curl -X POST http://localhost:18792/api/v1/agents \
  -H "Content-Type: application/json" \
  -d '{
    "name": "验证Agent",
    "type": "project", 
    "role": "验证Hook脚本",
    "workBaseDir": "/tmp/verify-agents",
    "customInstructions": "这是测试补充说明",
    "targetRepos": ["/tmp/test-repo"]
  }'
```
验证生成的 CLAUDE.md 必须包含：
```bash
AGENT_ID=$(curl ... | python -c "import sys,json; print(json.load(sys.stdin)['agentId'])")
grep -q "{{" /tmp/verify-agents/${AGENT_ID}/CLAUDE.md && echo "FAIL: 模板变量未替换" || echo "PASS"
grep -q "这是测试补充说明" /tmp/verify-agents/${AGENT_ID}/CLAUDE.md && echo "PASS: customInstructions" || echo "FAIL"
grep -q "/tmp/test-repo" /tmp/verify-agents/${AGENT_ID}/CLAUDE.md && echo "PASS: targetRepos" || echo "FAIL"
```

**V3-4**：settings.json 合法性验证
```bash
python -c "import json; json.load(open('/tmp/verify-agents/${AGENT_ID}/.claude/settings.json')); print('PASS: valid JSON')"
```
预期：输出 "PASS: valid JSON"

**V3-5**：settings.json 无模板变量残留
```bash
grep -q "{{" /tmp/verify-agents/${AGENT_ID}/.claude/settings.json && echo "FAIL: 模板变量未替换" || echo "PASS"
```
预期：输出 PASS

### 自检 Checklist

- [ ] V3-1 Hook 静默退出验证通过
- [ ] V3-2 Hook 正常工作验证通过
- [ ] V3-3 CLAUDE.md 生成验证通过（无残留模板变量，customInstructions 和 targetRepos 正确注入）
- [ ] V3-4 settings.json 合法性验证通过
- [ ] V3-5 settings.json 无模板变量残留验证通过
- [ ] hooks/ 已复制到 office/.hooks/
- [ ] 全局 `~/.claude/settings.json` 未被修改（执行 `cat ~/.claude/settings.json` 确认）

---

## 阶段4：前端主框架

### 任务描述

实现像素风格办公室 UI，包含楼层切换、Phaser 场景、侧边面板、底部流水线。必须直接复用原项目的渲染代码，不得自行重写。

### 前置确认

- [ ] 阶段2 自检 Checklist 全部通过
- [ ] **已完整阅读原项目 `Star-Office-UI/frontend/index.html` 全部代码**（必须实际读取文件，不得依赖记忆）
- [ ] 已阅读 DESIGN_DECISIONS.md 第十六章（UI 详细设计）
- [ ] 确认原项目 `Star-Office-UI/static/` 目录下的所有静态资源文件名（sprite、字体、tilemap等）

---

### ⚠️ 强制代码复用清单（执行前必读）

**以下代码块必须从原项目直接搬运，禁止重写、禁止简化：**

**1. Phaser 游戏配置（必须搬运）**

从原项目搬运以下配置到 `FloorScene.js`，一字不改：
- `const config = { type: Phaser.AUTO, width: 1280, height: 720, pixelArt: true, scale: {...} }` 完整配置
- `IS_TOUCH_DEVICE` 检测逻辑
- `checkWebPSupport()` 和 `checkWebPSupportFallback()` 两个函数
- `getExt()` 函数（WebP 兼容处理）

**2. 静态资源路径（必须搬运）**

原项目的所有 sprite/字体/tilemap 资源复制到 `ai-company-os/frontend/static/`，路径保持一致：
- `static/fonts/ark-pixel-12px-proportional-zh_cn.ttf.woff2`
- `static/` 下所有 `.png`、`.webp`、`.json`（tilemap）文件

**命令：**
```bash
cp -r Star-Office-UI/static ai-company-os/frontend/static
```

**3. preload() 函数（必须搬运）**

从原项目完整搬运 `preload()` 函数到 `FloorScene.js`，包括：
- 所有 `this.load.image()`、`this.load.spritesheet()`、`this.load.tilemapTiledJSON()` 调用
- 加载进度回调逻辑

**4. 像素角色渲染逻辑（必须搬运）**

从原项目搬运以下到 `FloorScene.js`：
- `star`（主角色）的 sprite 创建代码，含 anims.create 动画定义
- `guestSprites` 的创建逻辑（访客像素角色）
- 气泡（bubble）的创建和 typewriter 打字机动画
- 角色行走 `waypoints` 逻辑和 tween 动画

**5. CSS（必须搬运）**

从原项目 `<style>` 块中直接提取以下内容到 `ai-company-os/frontend/css/base.css`：
- `@font-face` 声明（ArkPixel 字体）
- `body` 基础样式（`background: #1a1a2e`，`font-family: 'ArkPixel'`）
- `#game-container` 样式（含 `image-rendering: pixelated`、`border: 4px solid #e94560`）
- `#loading-overlay`、`#loading-progress-bar` 样式（含 `background: linear-gradient(90deg, #e94560, #ffd700)`）
- 所有面板样式（`background: #2c2f3a`、`border: 2px solid #e94560`）
- 滚动条样式（`#guest-agent-list::-webkit-scrollbar` 等）

**6. 气泡文字内容（必须搬运）**

将原项目的 `BUBBLE_TEXTS` 对象完整搬运到新系统，新增 Agent 状态对应的文字。

**7. 加载动画（必须搬运）**

完整搬运 `#loading-overlay` HTML 结构和 `updateLoadingProgress()`、`hideLoadingOverlay()` 函数。

---

### 重要提示

- 原项目用的是 Phaser.js 渲染 Canvas，**不是 CSS 矩形**。房间必须渲染在 Canvas 上，通过 Phaser 的图形 API 绘制，而不是 HTML div。
- Agent 角色必须使用原项目现有的 sprite 图片（`guest_role_1` 到 `guest_role_6` 等），不得用 CSS 色块替代。
- Tilemap 背景必须加载原项目的 `.json` tilemap 文件，渲染像素地图纹理，不得用纯色背景替代。
- 如果原项目 `static/` 目录中没有某个资源，执行 `ls Star-Office-UI/static/` 确认实际有哪些文件，按实际文件名加载。

---

### 实现要求

**base.css**（从原项目提取，禁止自行编写）

执行步骤：
1. 读取 `Star-Office-UI/frontend/index.html` 的 `<style>` 块
2. 将其中全部 CSS 完整复制到 `ai-company-os/frontend/css/base.css`
3. 不得删除任何样式，不得修改任何颜色值

核心颜色值（仅供验证用，实际以原文件为准）：
- 背景：`#1a1a2e`
- 面板背景：`#2c2f3a`
- 强调色/边框：`#e94560`
- 标题色：`#ffd700`
- 次要背景：`#3a3f4f`
- 字体：ArkPixel

**layout.css**

整体三区域布局（TopBar + 主视图 + 底部流水线）：
- TopBar：高度 40px，背景 `var(--bg-panel)`，border-bottom `2px solid var(--accent)`
- 主视图区：flex 布局，Canvas 区域 + 侧边面板（340px固定宽）
- 底部流水线：高度 200px，背景 `var(--bg-panel)`，border-top `2px solid var(--accent)`

**TopBar.js**

DOM 结构：
```
[公司名：金色像素字] [楼层按钮组：B1/F1/F2/F3...] [项目定位下拉] [告警通知图标+数量] [设置图标]
```

交互：
- 楼层按钮：点击切换当前楼层，激活状态用 `var(--accent)` 背景
- 楼层按钮动态生成：F1 固定，F2+ 根据 projects 数量生成
- 项目定位：选择项目后触发 `selectProject(projectId)` 事件
- 告警图标：有 escalation 消息时数字角标变红闪烁

**FloorScene.js（Phaser 场景基类）**

继承原项目的 Phaser 配置，扩展��
- `addAgent(agentData)` 方法：在指定 area 渲染像素角色
- `removeAgent(agentId)` 方法：移除工位和角色
- `updateAgentState(agentId, state)` 方法：更新角色状态和气泡
- `highlightAgent(agentId)` 方法：边框闪烁高亮
- Agent 像素角色使用原项目现有的 sprite，不新增图片资源

Agent 状态气泡规范：
- idle：灰色气泡，角色随机小动作
- working：`#00ff88` 绿色气泡，打字动画
- waiting：`#ffd700` 金色气泡，等待动画
- error：`#e94560` 红色气泡，感叹号动画

工位拖拽：
- 使用 Phaser 的 drag 事件
- 拖拽结束后调用 `PATCH /api/v1/agents/{agentId}` 更新 `uiPosition`

**F1Scene.js**

固定房间：会议室（左上）、档案室（右上）、休息区（右下）、常驻 Agent 工位区（左下）
点击房间触发对应面板弹出（由前端 JS 处理，非 Phaser 内）

**B1Scene.js**

固定房间：服务器室、数据库室、MCP 工具架、知识库走廊
点击房间显示对应资源内容

**ProjectScene.js**

接受 projectId 参数动态渲染：
- 顶部金色像素字显示项目名
- 上方工位区：Team Lead Agent 工位，自动排列，最多4列
- 下方 Worker 区：更小的像素角色，动态显示子 Task

**SidePanel.js**

三个 Tab（消息/Agent/指令），每次只渲染当前激活的 Tab 内容：

消息 Tab：
- 显示 CEO 收件箱消息列表（`GET /api/v1/messages/ceo`）
- 每条消息显示：发件方、主题、优先级标签（HIGH 用红色）、状态、时间
- 点击展开完整内容

Agent Tab：
- 显示所有 Agent 列表
- 每行：状态圆点（颜色对应状态）、名称、当前任务、所在楼层
- 点击跳转到对应楼层并高亮工位
- 支持按楼层筛选

指令 Tab：
- 留空，阶段5实现

**Pipeline.js**

- 读取 `GET /api/v1/projects/{projectId}/board` 数据
- 按阶段渲染泳道，每个泳道显示该阶段的任务卡片
- 任务卡片：标题、负责 Agent、状态
- 顶部项目选择器切换显示不同项目的流水线

**SSE 连接（client.js）**

```javascript
const eventSource = new EventSource('/api/v1/events');

eventSource.onmessage = (e) => {
  const data = JSON.parse(e.data);
  switch(data.event) {
    case 'state_updated': updateAgentState(data); break;
    case 'agent_created': addAgentToFloor(data); break;
    case 'agent_removed': removeAgentFromFloor(data); break;
    case 'task_updated': refreshPipeline(data); break;
    case 'new_message': refreshSidePanel(data); break;
    case 'escalation': showAlert(data); break;
    case 'ping': break; // 心跳，忽略
  }
};

// 断线重连：3秒后自动重连
eventSource.onerror = () => {
  setTimeout(() => reconnectSSE(), 3000);
};
```

### 验证步骤

**V4-1**：页面正常加载
```
打开浏览器访问 http://localhost:18792（后端同时服务静态文件）
预期：无控制台报错，页面正常显示，TopBar/主视图/底部流水线三区域可见
```

**V4-2**：楼层切换
```
点击 TopBar 的 B1/F1 按钮
预期：Phaser 场景切换，当前激活楼层按钮背景变为 var(--accent)
```

**V4-3**：SSE 实时更新
```
在��一个终端执行：
  curl -X POST http://localhost:18792/api/v1/state/{某agentId} \
    -H "Content-Type: application/json" \
    -d '{"status": "working", "detail": "实时测试"}'
预期：UI 中对应 Agent 的状态气泡颜色和文字立即更新，无需刷新页面
```

**V4-4**：Agent 列表面板
```
点击侧边面板 "Agent" Tab
预期：显示已注册的所有 Agent，点击某个 Agent 跳转到对应楼层并高亮工位
```

**V4-5**：像素渲染一致性逐项检查（最重要的验证）
```
打开浏览器，同时打开原项目（Star-Office-UI）和新系统，逐项对比：

1. 背景：新系统背景色必须是 #1a1a2e（深蓝黑），不得是其他颜色
2. 字体：所有文字必须是 ArkPixel 像素字体，不得是普通系统字体
3. Canvas 渲染：主区域必须是 Phaser Canvas，不得是 HTML div 矩形
4. Tilemap：F1/B1 场景必须显示像素地图纹理背景，不得是纯色背景
5. Agent 角色：工位区必须显示像素 sprite 角色，不得是色块或方块
6. 气泡：Agent 状态气泡必须使用打字机动画，文字逐字出现
7. 边框：所有面板边框必须是 2-4px solid #e94560，不得是其他颜色
8. 加载动画：页面加载时必须显示进度条（#e94560→#ffd700渐变）

上述8项必须全部通过，任意一项不通过即视为验证失败，必须修复后重新验证。
```

**V4-6**：静态资源加载检查
```
打开浏览器开发者工具 Network 面板，刷新页面
预期：无 404 错误，所有 .png/.webp/.woff2/.json 资源加载成功
如有 404：检查 ai-company-os/frontend/static/ 目录，确认文件已从原项目复制
```

### 自检 Checklist

- [ ] V4-1 页面正常加载验证通过
- [ ] V4-2 楼层切换验证通过
- [ ] V4-3 SSE 实时更新验证通过
- [ ] V4-4 Agent 列表面板验证通过
- [ ] V4-5 像素渲染一致性8项全部通过
- [ ] V4-6 无静态资源 404 错误
- [ ] Canvas 区域使用 Phaser 渲染，而非 HTML div 矩形
- [ ] Agent 角色使用 sprite 图片，而非 CSS 色块
- [ ] static/ 目录已从原项目完整复制
- [ ] 未引用原项目文件路径（独立复制，非软链接）

---

## 阶段5：CEO 指令输入

### 任务描述

实现侧边面板"指令"Tab 的完整输入能力。

### 前置确认

- [ ] 阶段4 自检 Checklist 全部通过
- [ ] 已阅读 DESIGN_DECISIONS.md 第十五章（CEO 指令输入）

### 实现要求

**DirectiveInput.js**

UI 结构：
```
[目标 Agent 选择器（下拉，支持多选+广播选项）]
[多行文本输入框（支持 Markdown 代码块语法高亮）]
[附件区域：拖拽或点击上传，支持文件+图片+URL识别]
[展开/收起：结构化补充（projectId 选择器 + taskId 输入）]
[发送按钮]
```

文件上传：
- 使用 `FormData` 上传到 `POST /api/v1/messages`（multipart/form-data）
- 图片上传后在输入框内显示缩略图预览
- 文件上传后显示文件名和大小标签

URL 识别：
- 监听输入框 input 事件
- 正则检测 URL 格式，检测到后自动添加蓝色下划线标记

发送后：
- 清空输入框和附件区域
- 在消息 Tab 中立即显示刚发送的消息（status: 已发送）
- 自动切换到消息 Tab

**后端支持（messages.py 补充）**

`POST /api/v1/messages` 支持 multipart/form-data：
- 文件保存到 `office/attachments/{msgId}/`
- 返回 `attachmentPaths` 列表

### 验证步骤

**V5-1**：文字指令发送
```
在指令 Tab 选择一个 Agent，输入文字，点击发送
预期：消息 Tab 出现该消息，status 为已发送
预期：office/inbox/{agentId}/ 目录出现新消息文件
```

**V5-2**：文件附件发送
```
拖拽一个文件到输入框，发送
预期：office/attachments/{msgId}/ 目录存在该文件
预期：消息文件的 context.relatedFiles 包含该文件的绝对路径
```

**V5-3**：广播发送
```
选择"全体广播"，发送消息
预期：office/inbox/broadcast/ 出现消息文件
```

**V5-4**：结构化补充
```
展开结构化补充，选择 projectId，填写 taskId，发送
预期：消息文件的 context.projectId 和 context.taskId 字段正确填写
```

### 自检 Checklist

- [ ] V5-1 文字指令发送验证通过
- [ ] V5-2 文件附件发送验证通过
- [ ] V5-3 广播发送验证通过
- [ ] V5-4 结构化补充验证通过
- [ ] 附件路径为绝对路径，不是相对路径
- [ ] 消息文件严格符合第六章消息格式（无多余字段，无缺失必填字段）

---

## 阶段6：知识库与归档系统

### 任务描述

实现档案室 UI 和归档触发流程，完成知识库索引的完整可视化。

### 前置确认

- [ ] 阶段4 自检 Checklist 全部通过
- [ ] 已阅读 DESIGN_DECISIONS.md 第十三章（知识归档格式）

### 实现要求

**档案室 UI（点击 F1 层档案室房间触发）**

弹出模态面板，包含三个 Tab：

归档 Tab：
- 调用 `GET /api/v1/knowledge/archive`
- 显示归档卡片列表：agentName、projectName、归档时间、归档原因、tasksDone/tasksInterrupted
- 支持按 projectId 和关键词筛选
- 点击卡片：弹出完整 Markdown 渲染（调用 `GET /api/v1/knowledge/archive/{filename}`）
- "分配中断任务"按钮：仅在 reason=force_shutdown 且有 tasksInterrupted > 0 时显示

上下文 Tab：
- 按项目分组显示 context 索引
- 点击摘要卡片展开内容

资源 Tab：
- 显示 `GET /api/v1/resources` 数据
- MCP 工具列表、云资源、DB 连接，只读展示

**"分配中断任务"流程（后端支持）**

`POST /api/v1/knowledge/archive/{archiveFile}/assign`
- 请求体：`{"targetAgentId": "agent-xxx"}`
- 读取归档文件，提取 interruptedContext
- 构建 ceo_directive 消息（主题："接手中断任务"，携带完整 interruptedContext 作为 context）
- 写入 `office/inbox/{targetAgentId}/`

### 验证步骤

**V6-1**：归档室正常打开
```
点击 F1 层档案室房间
预期：弹出模态面板，三个 Tab 正常显示
```

**V6-2**：中断任务分配
```
如果存在归档（reason=force_shutdown），点击"分配中断任务"
选择目标 Agent，确认
预期：目标 Agent 的 inbox 出现新消息，context 包含 interruptedContext 数据
```

**V6-3**：资源展示
```
切换到"资源"Tab
预期：显示 office/config/resources.json 中的内容（初始为空，正常显示空状态）
```

### 自检 Checklist

- [ ] V6-1 归档室打开验证通过
- [ ] V6-2 中断任务分配验证通过
- [ ] V6-3 资源展示验证通过

---

## 阶段7：集成测试与联调

### 任务描述

模拟完整业务流程，验证系统端到端正确运行。

### 测试场景1：完整 Agent 生命周期

```
步骤1: 通过 UI 创建一个 project 类型 Agent（名称：集成测试Agent）
验证: 工位出现在对应楼层，状态为 idle

步骤2: 手动写入一个 in_progress 任务到该 Agent
验证: 底部流水线出现任务卡片

步骤3: 尝试销毁该 Agent（不强制）
验证: UI 弹出警告，显示有1个任务在执行

步骤4: 强制销毁
验证: 
  - shutdown 消息出现在 office/inbox/{agentId}/
  - 模拟 Agent 写入归档文件（手动创建归档 .md 文件）
  - 模拟 Agent 写入回执文件（status: done）
  - 工位从 UI 消失
  - agents.json 中该 Agent 被移除
```

### 测试场景2：消息收发与回执

```
步骤1: CEO 通过 UI 向某 Agent 发送指令
验证: 消息文件出现在 inbox，UI 消息 Tab 显示状态为"已发送"

步骤2: 手动写入回执文件（status: done）
验证: UI 消息 Tab 状态更新为"已完成"，无需刷新页面

步骤3: 不写回执，等待超时
验证: 超过 timeoutAt 后，CEO inbox 出现 escalation 消息，UI 告警图标数字增加
```

### 测试场景3：楼层与项目定位

```
步骤1: 创建两个项目（不同阶段定义）
验证: TopBar 出现 F2/F3 楼层按钮

步骤2: 为每个项目分配不同 Agent
验证: 对应楼层显示对应 Agent 工位

步骤3: 使用项目定位下拉框选择项目1
验证: 自动跳转到 F2 楼层，项目1的 Agent 高亮，底部流水线切换为项目1的阶段

步骤4: 选择项目2
验证: 跳转到 F3，Agent 高亮正确，流水线阶段正确
```

### 自检 Checklist

- [ ] 测试场景1全部步骤验证通过
- [ ] 测试场景2全部步骤验证通过
- [ ] 测试场景3全部步骤验证通过
- [ ] 整个测试过程中原项目目录无任何文件被修改（`git diff backend/ frontend/ docs/` 输出为空）
- [ ] 系统在 18792 端口运行，原项目 18791 端口不受影响
- [ ] 全局 `~/.claude/settings.json` 未被修改

---

## 附录A：常见问题处理

**Q: Hook 脚本调用后端失败怎么办？**
A: 所有 Hook 脚本对后端请求失败必须静默忽略，不能让系统不可用影响 Claude Code 正常工作。Agent 工作永远优先于状态推送。

**Q: office/ 目录中的 JSON 文件被损坏怎么办？**
A: 后端所有读取操作必须捕获 JSON 解析异常，返回空数据而不是 500 错误。同时记录错误日志。

**Q: 两个 Agent 同时写同一个文件怎么办？**
A: 本系统当前阶段不处理写冲突（单人使用，并发写的概率极低）。后续版本可引入文件锁。

**Q: Agent 工作目录在不同操作系统上路径格式不同怎么办？**
A: 所有路径使用 Python 的 `pathlib.Path` 处理，不硬编码路径分隔符。

---

## 附录B：风格参考快查

原项目风格变量（实施时直接从 `frontend/index.html` 读取最新值）：

```css
背景色: #1a1a2e
面板背景: #2c2f3a
强调色: #e94560
标题金色: #ffd700
次要背景: #3a3f4f
成功绿: #00ff88
字体: 'ArkPixel', 'Courier New', monospace
边框: 2px solid var(--accent)
阴影: 0 0 10px rgba(233, 69, 96, 0.3)
图形渲染: image-rendering: pixelated
```
