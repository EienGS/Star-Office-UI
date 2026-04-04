# AI Company OS — 问题解决执行计划

---

## 执行总原则

- 每个阶段完成后，你反馈结果给我，我再给下一阶段的提示词
- 标注【Claude Code执行】的，给 Claude Code 提示词让它做
- 标注【你执行】的，需要你手动操作并截图/粘贴结果反馈
- 标注【你+Claude Code】的，需要你先做一部分，再让 Claude Code 处理结果

---

## 阶段顺序总览

```
阶段1：端对端验证（最优先，决定系统是否真正可用）
  └── 你执行：手动跑通第一个真实 Agent

阶段2：区域坐标校准（视觉修正，阶段1完成后做）
  └── 你执行：用 ?dev=1 标注坐标
  └── Claude Code执行：写入 areas.js

阶段3：Agent 自动启动机制（降低使用门槛）
  └── Claude Code执行：生成启动脚本

阶段4：知识库检索能力（让知识库真正有价值）
  └── Claude Code执行：关键词索引 + 检索 API

阶段5：安全基础（扩展部署场景的前提）
  └── Claude Code执行：API Key 认证

阶段6：Pipeline 和 Archive 完善（功能完整性）
  └── Claude Code执行：完善模态框内容
```

---

## 阶段1：端对端验证 Hook 真实触发

**目标**：确认 Hook 脚本在真实 Claude Code 窗口中是否实际触发，这是整个系统的生死线。

**执行者：你**

按以下步骤操作，每步记录结果：

### 步骤1-1：确认 settings.json 格式

打开 Claude Code 创建的任意一个 Agent 工作目录，找到 `.claude/settings.json`，
把文件内容粘贴给我。我需要确认格式是否符合 Claude Code 的 Hooks 规范。

### 步骤1-2：创建一个最小测试 Agent

在系统 UI 中创建一个测试 Agent，配置如下：
- 名称：hook-test-001
- 类型：project
- 角色：Hook验证测试
- 工作目录：任意空目录

创建完成后，把以下内容粘贴给我：
1. 生成的 CLAUDE.md 内容（完整）
2. 生成的 .claude/settings.json 内容（完整）
3. 工作目录的完整路径

### 步骤1-3：在测试 Agent 的工作目录启动 Claude Code

打开终端，执行：
```bash
cd [Agent工作目录]
claude
```

在 Claude Code 里输入任意简单指令，例如：
```
列出当前目录下的文件
```

观察并截图以下内容：
1. Claude Code 执行工具调用前，终端是否有 Hook 脚本输出
2. `office/state/hook-test-001.json` 文件是否被自动更新
3. 前端 UI 中该 Agent 的状态是否实时变化

### 步骤1-4：测试 HIGH 消息中断

在 `office/inbox/hook-test-001/` 目录下手动创建文件 `msg-test-high.json`：
```json
{
  "msgId": "msg-test-high",
  "from": "ceo",
  "to": "hook-test-001",
  "type": "ceo_directive",
  "priority": "HIGH",
  "subject": "测试高优先级中断",
  "context": { "projectId": null },
  "createdAt": "2025-04-04T00:00:00Z",
  "status": "unread"
}
```

然后在 Claude Code 里再执行一个工具调用，观察是否出现中断提示。

**完成后**：把以上4步的截图和终端输出粘贴给我，我根据结果决定是否需要修复 Hook 配置。

---

## 阶段2：区域坐标校准

**前置条件**：阶段1完成后执行

**执行者：你**

### 步骤2-1：进入调试模式

在浏览器中打开系统，URL 末尾加 `?dev=1`：
```
http://localhost:18792?dev=1
```

此时画面上应出现绿色坐标显示和半透明网格参考线。

### 步骤2-2：逐楼层记录区域坐标

对每个楼层，记录每个功能区域的四个角坐标（左上角和右下角即可）。

**F1 楼层需要标注的区域：**
- 会议室（左上区域）
- 档案室（右上区域）
- 常驻工位区（左下区域）
- 休息区（右下区域）

**B1 楼层需要标注的区域：**
- 服务器室（左侧机柜区）
- 数据库室（右上区域）
- MCP工具架（右侧工具台）
- 知识库走廊（底部区域）

**PROJECT 楼层需要标注的区域：**
- Team Lead 工位区（上半部分桌子区）
- Worker 任务区（下半部分桌子区）

记录格式（每个区域记录两个点）：
```
会议室：左上角 x:___ y:___   右下角 x:___ y:___
档案室：左上角 x:___ y:___   右下角 x:___ y:___
（以此类推）
```

**完成后**：把所有坐标数据粘贴给我，我给出让 Claude Code 更新 areas.js 的提示词。

---

## 阶段3：Agent 自动启动机制

**前置条件**：阶段1验证通过后执行

**执行者：Claude Code**

完成阶段1反馈后，我会给出这一阶段的完整提示词。

核心内容：
- 创建 Agent 时，后端额外生成一个 `start.sh`（macOS/Linux）和 `start.bat`（Windows）启动脚本
- 前端 Agent 详情面板新增"启动命令"区域，显示可复制的一键启动命令
- 命令内容：`cd [workDir] && claude`

---

## 阶段4：知识库关键词检索

**前置条件**：阶段3完成后执行

**执行者：Claude Code**

核心内容：
- 新增 `GET /api/v1/knowledge/search?q=关键词&projectId=xxx` API
- 搜索逻辑：读取各 `_index.json` 的 `keywords` 字段做关键词匹配，返回命中的文件列表
- 前端档案室面板新增搜索框，支持关键词检索历史归档和任务摘要

---

## 阶段5：API 认证

**前置条件**：阶段4完成后执行

**执行者：Claude Code**

核心内容：
- 在 `office/config/` 下生成一个 `api_key.txt`（随机32位字符串）
- 所有 API 请求 Header 必须携带 `X-API-Key: [key]`
- Hook 脚本从环境变量 `OFFICE_API_KEY` 读取 key 并在请求时携带
- 前端 localStorage 存储 key，首次使用时弹窗要求输入

---

## 阶段6：Pipeline 和 Archive 模态框完善

**前置条件**：阶段5完成后执行

**执行者：Claude Code**

核心内容：
- Pipeline 模态框：展示当前项目所有任务卡片，按阶段分列，每张卡片显示标题/负责人/状态/创建时间，支持拖拽移动阶段
- Archive 模态框：展示归档列表，支持按项目/关键词筛选，点击查看归档详情，提供"分配给新 Agent"按钮

---

## 当前状态

- [ ] 阶段1：等待你执行并反馈
- [ ] 阶段2：等待阶段1完成
- [ ] 阶段3：等待阶段1完成
- [ ] 阶段4：等待阶段3完成
- [ ] 阶段5：等待阶段4完成
- [ ] 阶段6：等待阶段5完成
