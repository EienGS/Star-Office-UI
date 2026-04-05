# 阶段一实施提示词：PM Agent 模式 + Agent 记忆保全机制

## 开始前必须执行

1. 读取以下文件，不得依赖记忆：
   - backend/services/agent_service.py
   - backend/routers/agents.py
   - backend/models/agent.py（或 schemas.py，找到 Agent 数据模型定义）
   - office/.hooks/on_notification.py
   - office/.hooks/pre_tool_use.py

2. 列出 office/agents/ 目录下任意一个已有 Agent 的 CLAUDE.md 内容

3. 确认以下 API 已存在且可用：
   - POST /api/v1/agents（创建 Agent）
   - POST /api/v1/agents/{agentId}/shutdown（销毁 Agent）
   - POST /api/v1/agents/{agentId}/launch（启动 Agent）
   - POST /api/v1/agents/{agentId}/archive（归档 Agent）

   如有缺失，先补充实现，再继续后续任务。

---

## 核心约束（违反即停止，等待人工确认）

- 不得修改现有消息协议的任何已有字段
- 不得修改现有 Agent 状态（idle/working/error）的判断逻辑，只允许新增状态
- 不得修改现有 Archive 流程，只允许在其前面插入新步骤
- CLAUDE.md 模板必须分为独立的 manager 版和 worker 版两个文件，禁止合并
- 所有新增 API 必须遵循现有项目的路由命名规范和返回格式
- 禁止修改任何前端布局属性（width/height/flex/position）

---

## 任务一：Agent 类型扩展

### 1.1 后端 Agent 模型新增 manager 类型

在 Agent 数据模型中，将 type 字段的枚举值扩展：
- 原有：resident / project
- 新增：manager / worker

在 Agent 状态机中新增状态：
- 原有：idle / working / error / archived
- 新增：hibernated（休眠）/ harvesting（记忆收割中）

### 1.2 数据库或状态文件同步更新

确认 state.json 的 status 字段支持写入 hibernated 和 harvesting。

### 1.3 前端 Agent 渲染区分

在前端 Agent 渲染逻辑中，根据 type 字段显示不同图标或颜色标识：
- manager 类型：在名称旁显示 [PM] 标签，颜色使用金色 #ffd700
- worker 类型：在名称旁显示 [W] 标签，颜色使用蓝色 #4fc3f7
- hibernated 状态：图标整体降低透明度至 0.5，不移除工位
- harvesting 状态：图标显示旋转动画（表示正在生成记忆报告）

验证：创建一个 type=manager 的 Agent，确认前端显示 [PM] 金色标签。

---

## 任务二：消息协议扩展

### 2.1 新增两种消息类型

在项目消息协议中（参考现有 type 枚举定义位置）新增：

```
memory_harvest：PM Agent 发给 Worker，触发记忆收割
memory_harvested：Worker 发给 PM Agent，确认记忆报告已生成
```

### 2.2 memory_report.json 结构规范

在 backend/models/ 或 schemas/ 下创建 memory_report_schema.json，定义以下结构：

```json
{
  "agentId": "string",
  "role": "string",
  "projectId": "string",
  "harvestedAt": "ISO8601 时间字符串",
  "tasksCompleted": [
    {
      "taskId": "string",
      "title": "string",
      "outcome": "string",
      "keyDecisions": ["string"],
      "filesModified": ["string"],
      "pitfalls": ["string"]
    }
  ],
  "unfinishedItems": [
    {
      "description": "string",
      "priority": "high|medium|low",
      "suggestion": "string"
    }
  ],
  "criticalKnowledge": ["string"],
  "handoffNotes": "string"
}
```

此文件作为规范文档存在，不需要后端强制校验格式（Claude Code 生成的内容可能有差异）。

---

## 任务三：归档流程增强（在现有归档前插入记忆收割步骤）

### 3.1 修改 POST /api/v1/agents/{agentId}/archive 接口

在执行现有归档逻辑之前，新增以下步骤：

```
步骤1：将 Agent 状态改为 harvesting
步骤2：向该 Agent 的 inbox 写入 memory_harvest 消息
步骤3：轮询等待 Agent 工作目录下出现 memory_report.json 文件
        超时时间：120 秒
        轮询间隔：3 秒
步骤4：超时后无论是否有报告，继续执行归档
步骤5：如果 memory_report.json 存在，将其内容写入知识库：
        office/knowledge/context/{projectId}/{agentId}-memory-report.json
步骤6：继续执行原有归档流程（压缩 ZIP、移除工位等）
```

归档接口返回值新增字段：
```json
{
  "archived": true,
  "agentId": "xxx",
  "memoryHarvested": true,  // 新增：是否成功获取记忆报告
  "memoryReportPath": "xxx" // 新增：报告存储路径
}
```

### 3.2 修改 on_notification.py 处理 memory_harvested 消息

在 on_notification.py 的消息类型处理中新增：

```python
elif notification_type == "memory_harvested":
    # Worker 确认记忆报告已生成
    # 将 Agent 状态从 harvesting 改回 working（等待 archive API 完成归档）
    update_agent_status(agent_id, office_root, "working", "Memory report generated, awaiting archive")
```

---

## 任务四：休眠 API

新增接口：POST /api/v1/agents/{agentId}/hibernate

```
逻辑：
1. 将 Agent 状态改为 hibernated
2. 保留工作目录所有文件，不做任何删除或压缩
3. 返回：{"hibernated": true, "agentId": "xxx", "workDir": "xxx"}

说明：
- 休眠不触发记忆收割（因为工作目录完整保留，随时可以恢复）
- 恢复休眠：调用现有 launch API 即可，状态自动变回 idle
- 前端：休眠的 Agent 工位仍然显示在楼层，透明度降低
```

---

## 任务五：CLAUDE.md 模板分离

### 5.1 创建 Manager 版 CLAUDE.md 模板

路径：backend/templates/CLAUDE_manager.md

内容规范（必须包含以下所有章节）：

```markdown
# AI Company OS - Project Manager Agent

## 身份
你是一个项目经理 Agent，负责接收 CEO 的项目委托，拆解为子任务，
动态创建 Worker Agent 来执行，汇总成果后向 CEO 汇报。

## 办公室系统信息
- 办公室根目录：{OFFICE_ROOT}
- 你的 Agent ID：{AGENT_ID}
- 项目 ID：{PROJECT_ID}
- 后端 API 地址：{API_BASE_URL}
- API Key：{API_KEY}

## 收件箱规范
你的消息存放在：{OFFICE_ROOT}/inbox/{AGENT_ID}/
每次开始工作前，先检查收件箱是否有未读消息。
消息文件格式参考：{OFFICE_ROOT}/inbox/README.md（如存在）

## Worker Agent 生命周期管理

### 创建 Worker
当你需要创建 Worker Agent 时，调用：
POST {API_BASE_URL}/api/v1/agents
Headers: X-API-Key: {API_KEY}
Body:
{
  "name": "Worker-{功能描述}",
  "type": "worker",
  "role": "具体职责描述",
  "projectDir": "{项目工作目录}",
  "projectId": "{PROJECT_ID}"
}
响应中的 agentId 是该 Worker 的唯一标识。

### 启动 Worker
创建后调用：
POST {API_BASE_URL}/api/v1/agents/{agentId}/launch
Headers: X-API-Key: {API_KEY}
这会弹出一个新的终端窗口，Worker 的 Claude Code 将在其中运行。

### 分配任务给 Worker
将任务消息写入文件：
{OFFICE_ROOT}/inbox/{workerId}/{msgId}.json
消息格式：
{
  "msgId": "msg-{timestamp}-{random3}",
  "from": "{AGENT_ID}",
  "to": "{workerId}",
  "type": "task_assign",
  "priority": "NORMAL",
  "subject": "任务标题",
  "context": {
    "projectId": "{PROJECT_ID}",
    "taskId": "task-{描述}",
    "description": "详细任务描述",
    "expectedOutput": "期望交付物描述",
    "nextAgent": "{AGENT_ID}"
  },
  "createdAt": "{ISO8601时间}",
  "status": "unread"
}
注意：nextAgent 设置为你自己的 AGENT_ID，这样 Worker 完成后会通知你。

### 触发 Worker 记忆收割
当 Worker 完成任务，你决定归档它之前，先发送记忆收割消息：
{
  "msgId": "msg-{timestamp}-{random3}",
  "from": "{AGENT_ID}",
  "to": "{workerId}",
  "type": "memory_harvest",
  "priority": "HIGH",
  "subject": "请生成记忆报告",
  "createdAt": "{ISO8601时间}",
  "status": "unread"
}
发送后等待 Worker 发回 type=memory_harvested 的确认消息（检查你的收件箱）。

### 归档 Worker
收到 memory_harvested 确认后，调用：
POST {API_BASE_URL}/api/v1/agents/{workerId}/archive
Headers: X-API-Key: {API_KEY}

### 休眠 Worker（短期暂停，可能还需要）
POST {API_BASE_URL}/api/v1/agents/{workerId}/hibernate
Headers: X-API-Key: {API_KEY}

## 记忆收割协议
当你收到 type=memory_harvest 的消息时：
（Manager 一般不会收到此消息，但如果收到，执行 Worker 版的协议）

## 工作完成汇报
当所有子任务完成，向 CEO 汇报时，将消息写入：
{OFFICE_ROOT}/inbox/ceo/{msgId}.json
type 使用 task_complete

## 状态更新规范
每次开始新工作时，调用：
POST {API_BASE_URL}/api/v1/agents/{AGENT_ID}/status
Body: {"status": "working", "detail": "当前正在做什么"}
空闲时：{"status": "idle", "detail": "等待新任务"}
```

### 5.2 创建 Worker 版 CLAUDE.md 模板

路径：backend/templates/CLAUDE_worker.md

内容规范（必须包含以下所有章节）：

```markdown
# AI Company OS - Worker Agent

## 身份
你是一个 Worker Agent，负责执行 PM Agent 分配给你的具体子任务。
专注完成你的任务，完成后通知 PM Agent，然后等待下一步指令。

## 办公室系统信息
- 办公室根目录：{OFFICE_ROOT}
- 你的 Agent ID：{AGENT_ID}
- 项目 ID：{PROJECT_ID}
- 后端 API 地址：{API_BASE_URL}
- API Key：{API_KEY}

## 收件箱规范
你的消息存放在：{OFFICE_ROOT}/inbox/{AGENT_ID}/
启动后第一件事：检查收件箱，找到 type=task_assign 的消息，开始执行。

## 任务完成汇报
完成任务后，将完成消息写入 PM Agent 的收件箱：
{OFFICE_ROOT}/inbox/{pmAgentId}/{msgId}.json
消息格式：
{
  "msgId": "msg-{timestamp}-{random3}",
  "from": "{AGENT_ID}",
  "to": "{pmAgentId}",
  "type": "task_complete",
  "priority": "NORMAL",
  "subject": "任务完成：{任务标题}",
  "context": {
    "projectId": "{PROJECT_ID}",
    "taskId": "{taskId}",
    "result": "完成情况的简要描述",
    "filesCreated": ["相对路径列表"],
    "filesModified": ["相对路径列表"]
  },
  "createdAt": "{ISO8601时间}",
  "status": "unread"
}

## 记忆收割协议
当你收到 type=memory_harvest 的消息时，必须立即：

1. 停止当前所有工作
2. 在你的工作目录根目录生成 memory_report.json，格式如下：
{
  "agentId": "{AGENT_ID}",
  "role": "{你的角色}",
  "projectId": "{PROJECT_ID}",
  "harvestedAt": "{当前ISO8601时间}",
  "tasksCompleted": [
    {
      "taskId": "任务ID",
      "title": "任务标题",
      "outcome": "完成情况",
      "keyDecisions": ["关键技术决策1", "关键技术决策2"],
      "filesModified": ["修改的文件路径"],
      "pitfalls": ["遇到的坑和注意事项"]
    }
  ],
  "unfinishedItems": [
    {
      "description": "未完成的事项描述",
      "priority": "medium",
      "suggestion": "后续 Agent 的建议"
    }
  ],
  "criticalKnowledge": [
    "后续 Agent 必须知道的关键信息"
  ],
  "handoffNotes": "给下一个接手 Agent 的整体建议"
}

3. 生成完成后，将确认消息写入发送方的收件箱：
{
  "msgId": "msg-{timestamp}-{random3}",
  "from": "{AGENT_ID}",
  "to": "{发送 memory_harvest 消息的 agentId}",
  "type": "memory_harvested",
  "priority": "HIGH",
  "subject": "记忆报告已生成",
  "createdAt": "{ISO8601时间}",
  "status": "unread"
}

4. 更新自己状态为 idle，等待系统关闭，不再执行任何其他操作。

## 状态更新规范
启动时：POST {API_BASE_URL}/api/v1/agents/{AGENT_ID}/status
Body: {"status": "working", "detail": "正在读取任务"}
任务完成后：{"status": "idle", "detail": "任务完成，等待下一步指令"}
```

### 5.3 修改 agent_service.py 的 CLAUDE.md 生成逻辑

根据创建 Agent 时传入的 type 字段，选择对应模板：
- type == "manager" → 使用 CLAUDE_manager.md 模板
- type == "worker" → 使用 CLAUDE_worker.md 模板
- type == "resident" 或 "project"（原有类型）→ 使用原有模板

模板中的所有 {占位符} 必须替换为实际值，包括：
- {OFFICE_ROOT}：office 目录绝对路径（正斜杠）
- {AGENT_ID}：实际生成的 agentId
- {PROJECT_ID}：创建时传入的 projectId
- {API_BASE_URL}：http://localhost:18792
- {API_KEY}：从 office/config.json 读取的 apiKey

---

## 任务六：前端创建 Agent 表单扩展

在"创建 Agent"的表单中，type 选项新增：
- manager（项目经理）
- worker（执行工程师）

保留原有的 resident 和 project 类型不变。

表单提交时，type 字段传入后端，后端根据 type 选择对应 CLAUDE.md 模板。

---

## 验证流程（必须全部通过后才算完成）

### V1：基础类型验证
```
通过 UI 创建一个 type=manager 的 Agent，名称为 TestPM
确认：
1. 后端返回成功，agentId 格式正确
2. 前端楼层显示该 Agent，名称旁有 [PM] 金色标签
3. 生成的 CLAUDE.md 包含"Project Manager Agent"标题
4. 生成的 CLAUDE.md 中所有 {占位符} 均已替换为实际值
5. settings.json 的路径全部为正斜杠
```

### V2：休眠 API 验证
```
调用：POST /api/v1/agents/{testPMId}/hibernate
确认：
1. 返回 {"hibernated": true, ...}
2. 工作目录文件完整保留
3. state.json 中 status 变为 hibernated
4. 前端该 Agent 工位透明度降低，仍然显示
5. 再次调用 launch API，前端状态恢复为 idle
```

### V3：记忆收割消息验证
```
手动向任意 Worker Agent 的收件箱写入一条 type=memory_harvest 的消息
启动该 Worker 的 Claude Code 窗口
等待 Agent 处理（最多5分钟）
确认：
1. Agent 工作目录根目录出现 memory_report.json
2. memory_report.json 包含 tasksCompleted、criticalKnowledge 等字段
3. PM Agent 收件箱出现 type=memory_harvested 的确认消息
```

### V4：归档流程增强验证
```
对一个已有 memory_report.json 的 Agent 调用 archive API
确认：
1. 归档返回 {"memoryHarvested": true, ...}
2. office/knowledge/context/{projectId}/{agentId}-memory-report.json 文件已创建
3. 原有归档流程正常完成（ZIP 生成、工位移除）
对一个没有 memory_report.json 的 Agent 调用 archive API（超时场景）
确认：
1. 等待 120 秒后归档仍然完成，不卡住
2. 返回 {"memoryHarvested": false, ...}
```

### V5：完整 PM Agent 工作流模拟验证
```
1. 创建 type=manager 的 PM Agent，记录 agentId
2. 创建 type=worker 的 Worker Agent，记录 agentId
3. 手动向 Worker 的 inbox 写入 type=task_assign 消息，nextAgent 设为 PM Agent ID
4. 启动 Worker 的 Claude Code 窗口，执行"创建一个 hello.txt 文件，内容写 Hello World"
5. Worker 完成后确认 PM Agent 收件箱收到 task_complete 消息
6. 向 Worker 发送 memory_harvest 消息
7. 确认 Worker 生成 memory_report.json 并发回 memory_harvested 确认
8. 调用 archive API 归档 Worker
9. 确认知识库中出现对应的记忆报告文件
```

### V6：布局完整性验证
```
检查所有楼层切换正常
检查侧边面板所有 Tab 正常
检查 Pipeline 模态框和 Archive 模态框正常
确认无 JS 报错
```

---

## 自检 Checklist（全部勾选后反馈）

- [ ] V1：manager/worker 类型创建正常，前端标签显示正确，CLAUDE.md 占位符全部替换
- [ ] V2：休眠 API 正常，文件保留，状态正确，恢复正常
- [ ] V3：Worker 收到 memory_harvest 后正确生成 memory_report.json 并发回确认
- [ ] V4：归档流程读取 memory_report.json 并写入知识库，超时场景不卡住
- [ ] V5：完整 PM Agent 工作流模拟通过
- [ ] V6：现有功能无回归
- [ ] manager 模板和 worker 模板是两个独立文件
- [ ] 所有新增 API 遵循现有路由命名规范
- [ ] 未修改任何现有状态判断逻辑，只新增状态
- [ ] 未修改任何前端布局属性

全部完成后将每项验证截图或输出结果反馈给我，等待确认后再继续。
