# AI Company OS - 缺陷修复与优化建议文档

> 基于阶段7完成后的代码审查，对照原始设计文档（DESIGN_DECISIONS.md）输出
> 版本：v1.0
> 日期：2026-04-04

---

## 一、缺陷列表总览

| 编号 | 类型 | 模块 | 严重程度 | 状态 |
|------|------|------|---------|------|
| BUG-01 | 功能缺陷 | hooks/pre_tool_use.py | 高 | 待修复 |
| BUG-02 | 功能缺陷 | hooks/on_notification.py | 高 | 待修复 |
| BUG-03 | 功能缺陷 | hooks/on_notification.py | 高 | 待修复 |
| BUG-04 | 逻辑错误 | services/scheduler.py | 中 | 待修复 |
| BUG-05 | 功能缺失 | frontend/js/scene/FloorScene.js | 中 | 待确认 |
| BUG-06 | 体验缺陷 | frontend/js/components/TopBar.js | 低 | 待修复 |

---

## 二、缺陷详情与修复方案

---

### BUG-01：pre_tool_use.py HIGH 消息中断机制无效

**问题描述**

当前 `hooks/pre_tool_use.py` 检测到 HIGH 优先级消息后，仅将信息打印到 stdout，Claude Code 不会因此中断当前工具流程。设计要求是：检测到 HIGH 消息时，必须能真正阻断 Claude Code 继续执行下一步工具调用。

**根本原因**

Claude Code Hooks 的 block 机制要求 `pre_tool_use` 输出特定 JSON 格式到 stdout，且退出码为非零。当前实现只有 `print()` 没有 `sys.exit(1)`，导致 Claude Code 忽略了中断信号。

**修复方案**

修改 `hooks/pre_tool_use.py` 中 HIGH 消息检测部分：

```python
import sys
import json

# 检测到 HIGH 消息时的输出格式
def block_for_high_message(msg_id, subject, agent_id, office_root):
    block_response = {
        "continue": False,
        "reason": f"收到高优先级消息需要立即处理：{subject}。请先处理消息文件：{office_root}/inbox/{agent_id}/{msg_id}.json，完成后继续当前工作。"
    }
    print(json.dumps(block_response, ensure_ascii=False))
    sys.exit(1)  # 非零退出码告知 Claude Code 中断

# 无 HIGH 消息时正常放行
def allow_continue():
    print(json.dumps({"continue": True}))
    sys.exit(0)
```

**验证流程**

```
步骤1：环境准备
  在 office/inbox/{任意agentId}/ 目录下手动创建一条测试消息：
  文件名：msg-test-001.json
  内容：
  {
    "msgId": "msg-test-001",
    "from": "ceo",
    "to": "{agentId}",
    "type": "ceo_directive",
    "priority": "HIGH",
    "subject": "测试中断消息",
    "content": "这是一条测试用HIGH消息",
    "createdAt": "2026-04-04T00:00:00Z",
    "status": "unread"
  }

步骤2：触发 Hook
  在对应 Agent 的工作目录下启动 Claude Code
  让 Agent 执行任意工具操作（如 ls 或 read_file）

步骤3：验证结果
  预期：Claude Code 在执行工具前停止，终端输出包含"收到高优先级消息"提示
  不预期：工具正常执行完成后才出现提示

步骤4：验证正常放行
  清空 inbox 中的 HIGH 消息（或将 priority 改为 NORMAL）
  重新触发工具操作
  预期：工具正常执行，无任何中断
```

---

### BUG-02：on_notification.py 未实现 nextAgent 自动触发

**问题描述**

设计要求 Agent 完成任务后，自动读取 `task.json` 中的 `nextAgent` 字段，向下游 Agent 发送 `task_notify` 消息，实现任务链式流转。当前 `on_notification.py` 的 task_complete 处理分支中没有这段逻辑，任务流转完全依赖 CEO 手动干预。

**根本原因**

功能未实现，需要新增代码。

**修复方案**

在 `hooks/on_notification.py` 的 task_complete 处理块末尾新增：

```python
import os
import json
import time
import random
import string
from datetime import datetime, timezone

def trigger_next_agent(office_root, current_agent_id, project_id, task_id):
    """读取任务的 nextAgent 字段，向下游 Agent 发送通知消息"""
    if not project_id or not task_id:
        return

    task_file = os.path.join(office_root, 'tasks', project_id, f'{task_id}.json')
    if not os.path.exists(task_file):
        return

    with open(task_file, 'r', encoding='utf-8') as f:
        task = json.load(f)

    next_agent = task.get('nextAgent')
    if not next_agent:
        return

    # 验证目标 Agent 存在
    agents_file = os.path.join(office_root, 'config', 'agents.json')
    with open(agents_file, 'r', encoding='utf-8') as f:
        agents_data = json.load(f)
    agents = agents_data if isinstance(agents_data, list) else agents_data.get('agents', [])
    agent_ids = [a['agentId'] for a in agents]
    if next_agent not in agent_ids:
        print(f"[on_notification] 警告：nextAgent {next_agent} 不存在，跳过触发")
        return

    # 构建 task_notify 消息
    random_suffix = ''.join(random.choices(string.ascii_lowercase + string.digits, k=3))
    msg_id = f"msg-{int(time.time())}-{random_suffix}"
    now = datetime.now(timezone.utc).isoformat().replace('+00:00', 'Z')
    timeout_ts = int(time.time()) + 3600
    timeout_dt = datetime.fromtimestamp(timeout_ts, tz=timezone.utc).isoformat().replace('+00:00', 'Z')

    msg = {
        "msgId": msg_id,
        "from": current_agent_id,
        "to": next_agent,
        "type": "task_notify",
        "priority": "NORMAL",
        "subject": f"上游任务已完成，请继续：{task.get('title', task_id)}",
        "content": f"任务 {task_id} 已由 {current_agent_id} 完成。请按照项目计划继续后续工作。",
        "context": {
            "projectId": project_id,
            "taskId": task_id,
            "handoffReason": "上游任务已完成，自动流转",
            "expectedOutput": task.get('nextAgentExpected', ''),
            "relatedFiles": task.get('output', {}).get('artifacts', [])
        },
        "createdAt": now,
        "status": "unread",
        "timeoutAt": timeout_dt
    }

    inbox_dir = os.path.join(office_root, 'inbox', next_agent)
    os.makedirs(inbox_dir, exist_ok=True)
    msg_file = os.path.join(inbox_dir, f'{msg_id}.json')
    with open(msg_file, 'w', encoding='utf-8') as f:
        json.dump(msg, f, ensure_ascii=False, indent=2)

    print(f"[on_notification] 已触发下游 Agent {next_agent}，消息：{msg_id}")
```

**验证流程**

```
步骤1：准备测试任务文件
  在 office/tasks/{projectId}/task-chain-test.json 创建：
  {
    "taskId": "task-chain-test",
    "projectId": "{projectId}",
    "title": "链式流转测试任务",
    "assignee": "{agentA_id}",
    "nextAgent": "{agentB_id}",
    "nextAgentExpected": "接收并确认上游任务完成",
    "status": "in_progress",
    "output": { "artifacts": [] }
  }

步骤2：模拟 task_complete 通知
  手动执行 on_notification.py，传入参数：
  NOTIFICATION_TYPE=task_complete PROJECT_ID={projectId} TASK_ID=task-chain-test python hooks/on_notification.py

步骤3：验证结果
  检查 office/inbox/{agentB_id}/ 目录
  预期：出现新消息文件，type=task_notify，from={agentA_id}，subject 包含"上游任务已完成"

步骤4：验证 nextAgent 不存在时的容错
  将 task.json 的 nextAgent 改为不存在的 agent ID
  重复步骤2
  预期：脚本输出警告日志，不崩溃，不写文件
```

---

### BUG-03：on_notification.py 未实现知识库摘要自动写入

**问题描述**

设计要求 Agent 每次完成任务后，自动将任务摘要写入 `office/knowledge/context/{projectId}/{taskId}-summary.md`，并更新 `_index.json`。当前实现中这段逻辑缺失，导致知识库上下文永远为空，后续 Agent 无法从知识库获取历史任务的可复用结论。

**根本原因**

功能未实现，需要新增代码。

**修复方案**

在 `hooks/on_notification.py` 的 task_complete 处理块中新增：

```python
def write_task_summary(office_root, agent_id, project_id, task_id, task_result=None):
    """将任务完成摘要写入知识库"""
    if not project_id or not task_id:
        return

    context_dir = os.path.join(office_root, 'knowledge', 'context', project_id)
    os.makedirs(context_dir, exist_ok=True)

    now_str = datetime.now(timezone.utc).strftime('%Y-%m-%d %H:%M UTC')
    date_str = datetime.now(timezone.utc).strftime('%Y-%m-%d')

    # 读取任务文件获取标题
    task_file = os.path.join(office_root, 'tasks', project_id, f'{task_id}.json')
    task_title = task_id
    if os.path.exists(task_file):
        with open(task_file, 'r', encoding='utf-8') as f:
            task_data = json.load(f)
        task_title = task_data.get('title', task_id)

    # 写摘要文件（如已存在则跳过，由 Agent 手动补充内容）
    summary_file = os.path.join(context_dir, f'{task_id}-summary.md')
    if not os.path.exists(summary_file):
        summary_content = f"""# 任务摘要 - {task_title}

- **任务ID**：{task_id}
- **负责Agent**：{agent_id}
- **项目**：{project_id}
- **完成时间**：{now_str}

## 产出

{task_result or '（Agent 请在此补充本次任务的实际产出描述）'}

## 可复用结论

（Agent 请在此补充后续 Agent 可直接引用的结论、方案或代码片段，每条不超过2行）

## 踩坑记录

（Agent 请在此补充本次任务遇到的问题和解决方案，避免后续重复踩坑）
"""
        with open(summary_file, 'w', encoding='utf-8') as f:
            f.write(summary_content)
        print(f"[on_notification] 已写入任务摘要：{summary_file}")

    # 更新 _index.json
    index_file = os.path.join(context_dir, '_index.json')
    if os.path.exists(index_file):
        with open(index_file, 'r', encoding='utf-8') as f:
            index = json.load(f)
    else:
        index = {"projectId": project_id, "summaries": []}

    existing_task_ids = [s.get('taskId') for s in index.get('summaries', [])]
    if task_id not in existing_task_ids:
        index['summaries'].append({
            "file": f"{task_id}-summary.md",
            "taskId": task_id,
            "title": task_title,
            "agentId": agent_id,
            "completedAt": date_str,
            "keywords": []
        })
        with open(index_file, 'w', encoding='utf-8') as f:
            json.dump(index, f, ensure_ascii=False, indent=2)
        print(f"[on_notification] 已更新知识库索引：{index_file}")
```

**验证流程**

```
步骤1：确保项目目录存在
  确认 office/knowledge/context/{projectId}/ 目录存在
  如不存在：mkdir -p office/knowledge/context/{projectId}/
  创建空的 _index.json：{"projectId": "{projectId}", "summaries": []}

步骤2：模拟 task_complete 通知
  手动执行 on_notification.py，传入 PROJECT_ID 和 TASK_ID

步骤3：验证摘要文件创建
  检查 office/knowledge/context/{projectId}/{taskId}-summary.md 是否存在
  预期：文件存在，包含任务ID、负责Agent、三个待填写的章节标题

步骤4：验证索引更新
  检查 office/knowledge/context/{projectId}/_index.json
  预期：summaries 数组中出现新条目，包含 taskId、agentId、completedAt 字段

步骤5：验证幂等性（重复执行不重复写入）
  再次执行步骤2
  预期：_index.json 中 summaries 仍只有一条该任务的记录（不重复添加）
  预期：已存在的摘要文件不被覆盖
```

---

### BUG-04：scheduler.py Agent 状态降级逻辑错误

**问题描述**

当前实现以"Agent 变为 working 状态的时刻"开始计时，超过 300 秒自动降级。正确逻辑应为：以**状态文件最后一次写入时间**为基准，超过 300 秒无任何更新时才降级。

两者区别：若 Agent 持续执行工具（每次工具调用都会触发 PostToolUse 更新状态文件），即使总耗时超过 300 秒也不应降级。只有在状态文件"停止更新"超过 300 秒（即 Agent 可能已崩溃或失联）时才降级。

**根本原因**

计时起点选错，应读取文件系统的 mtime 而非内存中记录的状态变更时间。

**修复方案**

修改 `services/scheduler.py` 的 Agent 状态降级任务：

```python
import os
import time
import json
from datetime import datetime, timezone

async def check_agent_status_degradation(office_root: str):
    """
    检查所有 working 状态的 Agent，
    若状态文件超过 300 秒未更新，自动降级为 idle
    """
    state_dir = os.path.join(office_root, 'state')
    if not os.path.exists(state_dir):
        return

    now = time.time()
    THRESHOLD_SECONDS = 300  # 5分钟

    for filename in os.listdir(state_dir):
        # 只处理 Agent 状态文件，跳过 receipts 子目录
        if not filename.endswith('.json'):
            continue
        if filename.startswith('.'):
            continue

        state_file = os.path.join(state_dir, filename)
        if not os.path.isfile(state_file):
            continue

        try:
            with open(state_file, 'r', encoding='utf-8') as f:
                state = json.load(f)
        except (json.JSONDecodeError, IOError):
            continue

        # 只检查 working 状态
        if state.get('status') != 'working':
            continue

        # 以文件系统 mtime 为基准（不依赖 state 内的 updatedAt 字段）
        file_mtime = os.path.getmtime(state_file)
        seconds_since_update = now - file_mtime

        if seconds_since_update > THRESHOLD_SECONDS:
            agent_id = state.get('agentId', filename.replace('.json', ''))
            state['status'] = 'idle'
            state['detail'] = f'长时间无状态更新（{int(seconds_since_update)}秒），自动降级'
            state['updatedAt'] = datetime.now(timezone.utc).isoformat().replace('+00:00', 'Z')

            with open(state_file, 'w', encoding='utf-8') as f:
                json.dump(state, f, ensure_ascii=False, indent=2)

            print(f"[scheduler] Agent {agent_id} 已自动降级为 idle（{int(seconds_since_update)}秒无更新）")
```

**验证流程**

```
步骤1：准备测试状态文件
  在 office/state/ 下创建测试文件 agent-test-degradation.json：
  {
    "agentId": "agent-test-degradation",
    "status": "working",
    "detail": "测试降级",
    "updatedAt": "2026-01-01T00:00:00Z"
  }
  （updatedAt 故意设为过去时间，但关键是文件 mtime）

步骤2：设置文件 mtime 为 10 分钟前（Unix）
  python3 -c "import os, time; os.utime('office/state/agent-test-degradation.json', (time.time()-600, time.time()-600))"

步骤3：手动触发降级检查
  临时将 scheduler.py 的 THRESHOLD_SECONDS 改为 30（加速测试）
  重启后端服务，等待 30 秒

步骤4：验证结果
  读取 office/state/agent-test-degradation.json
  预期：status 已变为 "idle"，detail 包含"自动降级"字样

步骤5：验证活跃 Agent 不被误降级
  创建另一个 working 状态文件，mtime 设为 10 秒前
  等待超过 30 秒
  预期：该文件 status 保持 working 不变

步骤6：恢复 THRESHOLD_SECONDS 为 300
```

---

### BUG-05：Agent 工位拖拽持久化未验证

**问题描述**

设计文档要求 Agent 工位支持拖拽，拖拽后坐标持久化到 `office/config/agents.json` 的 `uiPosition` 字段。当前文档未明确说明前端是否实现了拖拽后的 PATCH API 调用。

**需确认内容**

检查 `frontend/js/scene/FloorScene.js` 中是否存在：
1. `this.input.setDraggable(sprite)` 调用
2. `dragend` 事件监听
3. `dragend` 回调中调用 `ApiClient.patch('/agents/{agentId}', {uiPosition: {x, y}})`

**若未实现，修复方案**

在 FloorScene.js 的 `addAgentToScene()` 方法中补充：

```javascript
// 开启精灵拖拽
sprite.setInteractive({ draggable: true });
this.input.setDraggable(sprite);

// 拖拽结束时持久化坐标
sprite.on('dragend', (pointer, dragX, dragY) => {
    const roundedX = Math.round(dragX);
    const roundedY = Math.round(dragY);

    // 更新本地坐标
    sprite.x = roundedX;
    sprite.y = roundedY;

    // 持久化到后端
    ApiClient.patch(`/agents/${agentData.agentId}`, {
        uiPosition: { x: roundedX, y: roundedY }
    }).catch(err => {
        console.error('[FloorScene] 工位坐标保存失败:', err);
    });
});

// 拖拽中跟随鼠标
sprite.on('drag', (pointer, dragX, dragY) => {
    sprite.x = dragX;
    sprite.y = dragY;
    // 同步移动气泡和标签
    if (sprite.bubble) sprite.bubble.setPosition(dragX, dragY - 40);
    if (sprite.nameLabel) sprite.nameLabel.setPosition(dragX, dragY + 30);
});
```

**验证流程**

```
步骤1：确认功能存在
  打开浏览器，进入任意楼层，尝试拖拽 Agent 工位图标
  预期：Agent 图标可以被拖动

步骤2：验证持久化
  将一个 Agent 工位拖动到新位置
  打开浏览器开发者工具 Network 面板
  预期：出现一条 PATCH /api/v1/agents/{agentId} 请求
  请求体包含 { "uiPosition": { "x": xxx, "y": xxx } }
  响应为 200

步骤3：验证刷新后位置保持
  刷新页面（Ctrl+R）
  预期：Agent 工位出现在拖拽后的位置，不回到默认位置

步骤4：验证数据写入
  读取 office/config/agents.json
  找到对应 Agent 记录
  预期：uiPosition 字段值与拖拽后的坐标一致
```

---

### BUG-06：TopBar 楼层按钮缺少项目名称 Tooltip

**问题描述**

当系统有多个活跃项目时，F2、F3 等按钮无法区分对应哪个项目，用户只能逐一点击确认。设计要求按钮 hover 时显示项目名称。

**修复方案**

修改 `frontend/js/components/TopBar.js` 中动态生成楼层按钮的代码：

```javascript
// 生成项目楼层按钮时添加 title 属性
projects.forEach((project, index) => {
    const floorLabel = `F${index + 2}`;
    const btn = document.createElement('button');
    btn.className = 'floor-btn';
    btn.textContent = floorLabel;
    btn.title = `${project.name}（${project.status === 'active' ? '进行中' : '已归档'}）`;
    btn.dataset.floorId = floorLabel;
    btn.dataset.projectId = project.projectId;
    btn.addEventListener('click', () => this.switchFloor(floorLabel, project.projectId));
    this.floorBtnsContainer.appendChild(btn);
});
```

**验证流程**

```
步骤1：创建至少两个项目（使 F2、F3 按钮都出现）

步骤2：鼠标悬停到 F2 按钮
  预期：出现浏览器原生 tooltip，显示对应项目名称和状态
  例如："官网改版（进行中）"

步骤3：鼠标悬停到 F3 按钮
  预期：显示另一个项目的名称，与 F2 不同
```

---

## 三、优化建议

---

### OPT-01：Agent 走动动画联动区域点击（体验增强）

**当前状态**

点击区域只触发功能（如打开档案室模态框），Agent 像素角色不移动。

**设计原意**

点击某区域时，该楼层有 Agent 时应触发走路动画，角色移动到目标房间后功能触发。

**建议实现**

在 FloorScene.js 的 Zone `pointerdown` 回调中，先触发走路动画，300ms 后再触发功能：

```javascript
zone.on('pointerdown', () => {
    // 触发走路动画（如果有 Agent 在场景中）
    if (this.agentSprites && this.agentSprites.length > 0) {
        const targetX = area.x + area.w / 2;
        const targetY = area.y + area.h / 2;
        this.moveAgentToArea(this.agentSprites[0], targetX, targetY, () => {
            // 走到目标后再触发区域功能
            window.dispatchEvent(new CustomEvent('roomClick', {
                detail: { room: name, floor: floorKey, label: area.label }
            }));
        });
    } else {
        window.dispatchEvent(new CustomEvent('roomClick', {
            detail: { room: name, floor: floorKey, label: area.label }
        }));
    }
});
```

**优先级**：低（体验优化，不影响功能）

---

### OPT-02：广播消息在消息 Tab 中的展示

**当前状态**

广播消息写入 `inbox/broadcast/`，但前端消息 Tab 未说明如何区分和展示广播历史。

**建议实现**

在 SidePanel.js 的消息列表中，对 `type=broadcast` 的消息增加特殊样式标识：

```javascript
// 消息卡片渲染时判断类型
if (msg.type === 'broadcast') {
    card.classList.add('msg-broadcast');
    // 在发送者前加广播图标
    senderEl.textContent = '[广播] ' + msg.from;
}
```

```css
/* panel.css */
.msg-broadcast {
    border-left: 3px solid #ffd700;
    background: rgba(255, 215, 0, 0.05);
}
```

**优先级**：中

---

### OPT-03：消息列表分页

**当前状态**

`GET /messages/{agentId}` 返回全量消息摘要列表，随着使用时间增长性能会下降。

**建议实现**

在 API 层新增分页参数：

```python
@router.get("/messages/{agent_id}")
async def get_messages(
    agent_id: str,
    page: int = 1,
    page_size: int = 20,
    status: Optional[str] = None  # 可按状态过滤
):
```

前端消息列表底部增加"加载更多"按钮。

**优先级**：低（当前单用户场景消息量有限）

---

### OPT-04：常驻 Agent 并发任务计数展示

**当前状态**

设计文档提到 resident 类型 Agent 需要显示 `activeTasks` 计数，但当前 AgentTab 未实现。

**建议实现**

在 scheduler.py 中新增计数维护逻辑：

```python
async def update_resident_agent_task_count():
    """更新常驻 Agent 的活跃任务计数"""
    agents = load_agents()
    for agent in agents:
        if agent['type'] != 'resident':
            continue
        
        # 扫描所有项目，统计分配给该 Agent 的 in_progress 任务数
        active_count = count_in_progress_tasks(agent['agentId'])
        
        state_file = get_state_file(agent['agentId'])
        state = load_state(state_file)
        state['activeTasks'] = active_count
        save_state(state_file, state)
```

在 SidePanel.js 的 AgentTab 中，对 resident 类型 Agent 显示计数徽章：

```javascript
if (agent.type === 'resident' && state.activeTasks > 0) {
    const badge = document.createElement('span');
    badge.className = 'task-count-badge';
    badge.textContent = state.activeTasks;
    agentCard.appendChild(badge);
}
```

**优先级**：低

---

### OPT-05：知识库查询入口整合到指令 Tab

**当前状态**

档案室只能通过点击 F1 层的区域触发，无法在发指令时快速引用知识库内容。

**建议实现**

在 SidePanel.js 的指令 Tab 输入框下方增加"引用知识库"按钮，点击后弹出知识库搜索框，选中内容后自动插入到指令文本中。

**优先级**：中（提升 CEO 工作效率）

---

## 四、完整执行顺序建议

建议按以下顺序执行，先修复高严重度缺陷，再处理优化：

```
第一批（必须修复，影响核心功能）：
  BUG-01 → BUG-02 → BUG-03 → BUG-04

第二批（确认/修复，影响用户体验）：
  BUG-05 → BUG-06

第三批（优化，按优先级）：
  OPT-02 → OPT-05 → OPT-04 → OPT-01 → OPT-03
```

---

## 五、回归验证清单

完成所有修复后，执行以下完整回归验证：

```
[ ] 1. 创建新 Agent → 工位出现在 F1 层 → CLAUDE.md 和 settings.json 已生成
[ ] 2. 发送 HIGH 消息 → Agent 在工具调用前被中断 → 处理完成后恢复工作
[ ] 3. 触发 task_complete → nextAgent 收到通知消息 → 知识库出现摘要文件
[ ] 4. 让 Agent 进入 working 状态后停止更新 → 5分钟后自动降级为 idle
[ ] 5. 拖拽 Agent 工位 → 刷新页面 → 位置保持
[ ] 6. 创建多个项目 → F2/F3 按钮出现 → hover 时显示项目名称 tooltip
[ ] 7. 发送广播消息 → 消息 Tab 显示广播标识
[ ] 8. 消息超时 60 秒未回执 → CEO 收件箱出现 escalation → TopBar 告警数字增加
[ ] 9. 强制销毁有任务的 Agent → 出现二次确认 → 归档文件生成 → 工位消失
[ ] 10. 点击 F1 档案室区域 → 模态框打开 → 分配中断任务 → 目标 Agent 收到消息
```

---

**文档结束**
