# AI Company OS - 设计决策文档

> 版本：v0.3（讨论中）
> 最后更新：2025-04-02
> 状态：方案确认阶段，尚未进入实施

---

## 一、项目定位

**目标：** 一个人管理多团队、多任务、多项目的 AI 公司操作系统（AI Company OS）。

**核心能力：**
- 多个 Claude Code 窗口（Agent）的统一可视化管理
- Agent 之间的任务流转与信息传递
- 任务全生命周期跟踪
- 公共资源（MCP、知识库、云资源）统一管理
- CEO 视角的全局调度与指令下发

---

## 二、系统分层架构

```
┌─────────────────────────────────────────┐
│  你（CEO）                               │
│  - 创建/销毁 Agent                       │
│  - 下发指令（完整 Claude Code 输入能力）  │
│  - 全局监控                              │
└──────────────┬──────────────────────────┘
               │
   ┌───────────▼────────────┐
   │   Office Server（后端） │
   │   - 消息总线 API        │
   │   - 状态管理            │
   │   - 生命周期管理        │
   │   - 归档触发            │
   └───────────┬────────────┘
               │
   ┌───────────▼────────────┐
   │  File System Protocol  │  ← 所有模块的契约基础
   │  /office/ 目录结构      │
   └───────────┬────────────┘
               │
   ┌───────────▼────────────┐
   │  Claude Code（Agent 侧）│
   │  - CLAUDE.md 规范       │
   │  - Hook 脚本            │
   │  - 收件箱监听           │
   └────────────────────────┘
```

---

## 三、UI 设计方案

**选定方案：C（混合结构）**

- 主视图：像素风格办公室平面图（展示 Agent 位置与状态）
- 底部：任务流水线泳道（动态阶段，非固定软件开发流程）
- 侧边：消息面板 + Agent 列表

**任务流水线阶段：**
- 由 CEO 创建项目时自定义，完全领域无关
- 示例：软件开发（开发→测试→部署）/ 内容创作（选题→撰写→审核）/ 运营策划（立项→执行→复盘）

---

## 四、Agent 体系

### 4.1 角色类型

| 类型 | 对应实体 | 项目绑定 | 并发任务 |
|------|---------|---------|---------|
| CEO | 人类操作者 | - | - |
| Team Lead | Claude Code 主进程 | 严格 1:1 | 单项目 |
| Worker | Claude Code 子 Task | 继承 Team Lead | 进程内并发 |
| 常驻 Agent | Claude Code 主进程 | 不绑定（参与多项目） | 同类任务并发 |

### 4.2 常驻 Agent 定位

- 处理固定化、通用化任务（如：服务部署、运维、测试）
- 可同时接收多个项目的同类任务
- 通过通知型消息触发（不强制调用）

### 4.3 Agent 生命周期

```
创建
  → 系统自动生成 CLAUDE.md（通用规范 + 角色身份）
  → 注册到 /office/config/agents.json
  → UI 生成工位

销毁（正常）
  → 系统发送 shutdown 指令
  → Agent 执行归档：任务摘要 + 知识产出 + 踩坑记录
  → 写入 /office/knowledge/archive/[agentId]-[date].md
  → Agent 回执 "ready_to_shutdown"
  → UI 移除工位

销毁（强制中断，有任务在跑）
  → UI 弹出提示：当前有 N 个任务正在执行
  → 用户确认强制中断
  → 将中断状态打包写入 interruptedContext
  → 一并归档后移除工位
```

---

## 五、Agent 间通信机制

### 5.1 通信类型

| 场景 | 类型 | 技术实现 |
|------|------|---------|
| Team Lead → Worker | 调用型 | Claude Code 进程内原生 Task |
| Team Lead A → Team Lead B | 调用型（模拟） | HIGH 优先级消息 + Hook 立即触发 |
| Team Lead → 常驻 Agent | 通知型 | 写入收件箱，B 自行处理 |
| CEO → 任意 Agent | 调用型 | UI 指令 → HIGH 优先级消息 |
| A 等待 B 结果 | 回执轮询 | A 轮询 /office/state/receipts/[msgId].json |

### 5.2 收件箱触发时机

- **HIGH 优先级消息**：PreToolUse Hook 触发，立即处理
- **NORMAL 优先级消息**：Task 结束后检查全量收件箱
- **超时未回执**：系统自动升级为 CEO 通知（escalation）

### 5.3 上下文隔离保障

每条跨 Agent 消息必须携带完整上下文快照，包含：
- projectBrief（项目背景摘要）
- taskHistory（相关任务历史）
- handoffReason（交接原因）
- expectedOutput（期望产出）

不依赖对方"记得"之前的通信内容。

### 5.4 回执闭环

```
发送方写消息 → status: "unread"
接收方读取   → status: "acknowledged"
接收方完成   → status: "done" + result
发送方确认   → 继续下一步

超时未回执   → system 写 escalation 消息给 CEO
```

---

## 六、File System Protocol v1.0

### 6.1 目录结构

```
/office/
├── config/
│   ├── agents.json           # Agent 注册表
│   ├── projects.json         # 项目注册表
│   └── resources.json        # 公共资源注册表
│
├── inbox/
│   ├── [agentId]/            # 每个 Agent 的收件箱
│   │   └── [msgId].json
│   └── broadcast/            # 广播频道
│
├── tasks/
│   ├── [projectId]/
│   │   ├── board.json        # 看板状态（阶段定义 + 任务索引）
│   │   └── [taskId].json     # 任务详情
│   └── _templates/
│
├── knowledge/
│   ├── skills/               # SKILL 库
│   ├── sops/                 # 标准操作流程
│   ├── context/              # 项目上下文（Agent 写入）
│   │   └── [projectId]/
│   └── archive/              # 销毁 Agent 时的归档
│       └── [agentId]-[date].md
│
├── resources/                # 服务器室（只读）
│   ├── mcp/
│   ├── cloud/
│   └── db/
│
└── state/
    ├── [agentId].json        # Agent 实时状态
    └── receipts/
        └── [msgId].json      # 消息回执
```

### 6.2 核心数据结构

**Agent 注册（agents.json 单条）**
```json
{
  "agentId": "agent-frontend-001",
  "name": "前端团队",
  "type": "project",
  "projectId": "proj-001",
  "role": "前端开发",
  "claudeWorkDir": "/path/to/workdir",
  "createdAt": "ISO8601",
  "status": "active"
}
```

**项目注册（projects.json 单条）**
```json
{
  "projectId": "proj-001",
  "name": "官网改版",
  "type": "自由定义",
  "stages": ["待认领", "设计中", "开发中", "测试中", "已完成"],
  "assignedAgents": ["agent-frontend-001"],
  "residentAgents": ["agent-devops-001"],
  "createdAt": "ISO8601",
  "status": "active"
}
```

**消息格式（msgId.json）**
```json
{
  "msgId": "msg-[timestamp]-[random]",
  "from": "agentId | ceo",
  "to": "agentId",
  "type": "task_request | task_notify | shutdown | ceo_directive | broadcast | context_update | escalation",
  "priority": "HIGH | NORMAL",
  "subject": "消息标题",
  "context": {
    "projectId": "",
    "projectBrief": "",
    "taskId": "",
    "handoffReason": "",
    "expectedOutput": "",
    "relatedFiles": []
  },
  "createdAt": "ISO8601",
  "status": "unread | acknowledged | done | failed",
  "replyTo": "agentId | ceo",
  "timeoutAt": "ISO8601"
}
```

**任务详情（taskId.json）**
```json
{
  "taskId": "",
  "projectId": "",
  "title": "",
  "stage": "对应 project.stages 中的某一项",
  "assignee": "agentId",
  "createdBy": "agentId | ceo",
  "createdAt": "ISO8601",
  "updatedAt": "ISO8601",
  "input": {
    "description": "",
    "attachments": [],
    "referenceFiles": []
  },
  "output": {
    "result": null,
    "artifacts": []
  },
  "status": "pending | in_progress | done | failed | interrupted",
  "interruptedContext": null
}
```

**Agent 实时状态（state/agentId.json）**
```json
{
  "agentId": "",
  "status": "idle | working | waiting | error",
  "currentTask": "taskId | null",
  "detail": "展示在 UI 气泡的文字",
  "updatedAt": "ISO8601",
  "area": "desk | meeting | archive | server | idle_zone"
}
```

**消息回执（receipts/msgId.json）**
```json
{
  "msgId": "",
  "from": "agentId",
  "status": "acknowledged | done | failed",
  "result": "",
  "respondedAt": "ISO8601"
}
```

### 6.3 消息类型说明

| type | 发起方 | 含义 | 需要回执 |
|------|--------|------|---------|
| `task_request` | Agent / CEO | 调用型，要求对方执行任务 | 是 |
| `task_notify` | Agent | 通知型，告知完成/进展 | 可选 |
| `shutdown` | CEO / System | 触发归档销毁流程 | 是 |
| `ceo_directive` | CEO | CEO 直接指令 | 是 |
| `broadcast` | 任意 | 全体通知 | 否 |
| `context_update` | Agent | 更新共享上下文到知识库 | 否 |
| `escalation` | System | 超时/异常自动升级通知 CEO | 否 |

---

## 七、Hook 触发规则（Agent 侧）

```
PreToolUse
  → 推送 status: "working" + 当前工具名 到 state/[agentId].json
  → 检查 inbox 中 priority=HIGH 的消息
    → 有则优先处理高优先级消息

PostToolUse
  → 推送状态更新到 state/[agentId].json

Task 结束（Notification: task_complete）
  → 检查全量收件箱（NORMAL 消息）
  → 写 context_update 到 /office/knowledge/context/[projectId]/
  → 发出待发送的 task_notify 消息
```

---

## 八、公共设施（服务器室 / 档案室）

| 设施 | 路径 | 内容 | Agent 使用方式 |
|------|------|------|--------------|
| 服务器室 | `/office/resources/` | MCP 工具配置、云资源、DB 连接 | 只读，按需取用 |
| 档案室 | `/office/knowledge/` | SKILL 库、SOP、项目上下文、归档 | 任务开始时查阅，完成后写入 |
| 会议室 | `/office/tasks/[proj]/board.json` | 看板状态、任务交接记录 | 任务移交时更新 |
| 部署室 | 常驻 devops Agent 工位 | 部署脚本、环境配置 | 其他 Agent 发 task_request 触发 |

---

## 九、尚未讨论的模块

以下模块已识别，尚未深入设计：

- [x] **Agent 侧设计**：CLAUDE.md 模板完整内容、Hook 脚本实现（见第十一章）
- [ ] **Office Server 设计**：API 定义、文件监听、自动化逻辑（超时/升级/归档触发）
- [ ] **CEO 指令输入**：完整输入能力的技术实现（文字+文件+图片+URL）
- [ ] **UI 详细设计**：平面图房间划分、Agent 动画、任务卡片交互
- [ ] **CLAUDE.md 模板生成器**：创建 Agent 时的自动化脚本
- [ ] **知识归档格式**：Agent 销毁时归档内容的标准结构

---

---

## 十一、Agent 侧设计（方向A 确认版）

### 11.1 CLAUDE.md 结构原则

**采用方式 B（动态配置分离）：**
- CLAUDE.md 只写静态通用规范，内容稳定，无需重启生效
- 动态配置（projectId、role、customInstructions 等）全部存放在 /office/config/agents.json
- Agent 每次任务开始时主动读取 agents.json 获取最新配置
- 好处：CEO 在 UI 修改 Agent 配置无需重启 Claude Code 窗口

### 11.2 CLAUDE.md 模板结构

两层组成：
- 通用规范层（所有 Agent 共享，系统统一维护）
- 角色身份层（创建时注入 agentId，其余运行时从 agents.json 读取）

**强制行为规范（5条）：**
1. 任务开始前：读收件箱 + 读看板 + 读项目上下文
2. 执行中：HIGH 消息当前工具完毕后立即处理
3. 任务完成后：更新 task 状态 + 写上下文摘要 + 触发下游
4. 跨 Agent 消息：必须携带完整 context 快照，设置 timeoutAt
5. 收到 shutdown：归档后回执，不得直接退出

### 11.3 Hook 脚本分工

| Hook 挂载点 | 脚本 | 职责 |
|------------|------|------|
| PreToolUse | pre_tool_use.py | 推送 working 状态 + 检查 HIGH 消息 |
| PostToolUse | post_tool_use.py | 推送状态更新 |
| Notification: task_complete | on_task_complete.py | 处理 NORMAL 收件箱 + 写上下文摘要 + 触发下游 |
| Notification: error | on_error.py | 推送 error 状态 + 写 escalation 给 CEO |

### 11.4 环境变量（每个 Claude Code 窗口必须设置）

```
OFFICE_AGENT_ID=agent-xxx-001
OFFICE_ROOT=/path/to/office
```

CLAUDE.md 中的所有文件操作都通过这两个变量定位，不硬编码路径。

### 11.5 原生 Claude Code 隔离保障

**核心原则：Hook 和 CLAUDE.md 只在 Agent 专属工作目录生效，不影响原生使用。**

- Claude Code Hooks 采用**项目级配置**，写入 `[agentWorkDir]/.claude/settings.json`
- CLAUDE.md 同为项目级，只对该目录下启动的 Claude Code 有效
- 全局配置文件 `~/.claude/settings.json` 不写入任何系统内容
- 日常随手开启的 Claude Code 窗口（在其他目录）原生行为 100% 保留

**Agent 工作目录规范（创建时强制执行）：**
- 系统为每个 Agent 创建独立的专属工作目录（如 `/office-agents/team-frontend/`）
- 不允许将已有代码仓库目录设为 Agent 工作目录
- Agent 通过配置读取目标代码目录路径，而不是直接住在代码仓库内
- 好处：代码仓库中随时开 Claude Code 做小任务，不触发任何系统逻辑

---

## 十、实施优先级（待最终确认）

建议顺序（基于依赖关系）：
1. File System Protocol（已完成设计）
2. Agent 侧：CLAUDE.md 模板 + Hook 脚本
3. Office Server：核心 API + 文件监听
4. CEO 指令输入实现
5. UI：平面图 + 流水线 + 消息面板
6. 集成测试与调优
