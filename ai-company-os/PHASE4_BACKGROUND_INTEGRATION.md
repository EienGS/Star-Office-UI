请执行以下任务：将三张新背景图接入当前项目，替换原有的 office_bg，并实现区域定位系统。

---

## 前置准备（开始前必须完成）

1. 使用 Read 工具读取以下文件，不得依赖记忆：
   - Star-Office-UI/frontend/index.html（了解原项目资源加载方式）
   - ai-company-os/frontend/js/scene/F1Scene.js
   - ai-company-os/frontend/js/scene/B1Scene.js
   - ai-company-os/frontend/js/scene/ProjectScene.js
   - ai-company-os/frontend/js/scene/FloorScene.js（如存在）

2. 确认以下三张图片文件已存在于项目目录：
   - F1_BG.png（F1公共层背景）
   - B1_BG.png（B1基础设施层背景）
   - PROJECT_BG.png（项目楼层背景）

   如不在 ai-company-os/frontend/static/ 目录下，先将其复制到该目录。

---

## 核心约束（违反即停止，等待人工确认）

- 不得修改任何 HTML 布局结构（顶部栏/侧边面板/底部流水线）
- 不得修改任何 CSS 布局属性（width/height/flex/grid/position）
- 不得修改楼层切换按钮逻辑
- 所有区域坐标必须从独立配置文件读取，禁止硬编码在场景文件内
- 不得删除任何现有的 sprite 资源加载代码（装饰物 sofa/plants/cats 等仍需保留叠加在背景上）

---

## 任务一：替换背景图

### 1.1 在 PreloadScene.js 中加载三张新背景图

在现有 preload() 函数中，新增以下加载（不删除原有加载代码）：

```javascript
this.load.image('f1_bg', 'static/F1_BG.png');
this.load.image('b1_bg', 'static/B1_BG.png');
this.load.image('project_bg', 'static/PROJECT_BG.png');
```

### 1.2 替换各场景的背景图引用

- F1Scene.js：将 `this.add.image(640, 360, 'office_bg')` 改为 `this.add.image(640, 360, 'f1_bg')`
- B1Scene.js：将背景改为 `this.add.image(640, 360, 'b1_bg')`
- ProjectScene.js：将背景改为 `this.add.image(640, 360, 'project_bg')`

背景图必须设置为场景最底层：`bg.setDepth(0)`，其余所有元素 depth >= 1。

---

## 任务二：区域配置文件

创建 `ai-company-os/frontend/js/config/areas.js`，内容如下：

```javascript
// 区域坐标配置文件
// 所有坐标基于 1280x720 画布
// x/y 为区域左上角坐标，w/h 为宽高
// 注意：这是初始估算坐标，需要通过调试工具校准后更新

export const FLOOR_AREAS = {
  F1: {
    meeting_room: {
      x: 30, y: 30, w: 390, h: 380,
      label: '会议室',
      color: 0xe94560,
      alpha: 0.15
    },
    archive_room: {
      x: 820, y: 30, w: 430, h: 330,
      label: '档案室',
      color: 0xffd700,
      alpha: 0.15
    },
    resident_desk: {
      x: 30, y: 420, w: 430, h: 280,
      label: '常驻工位',
      color: 0x00ff88,
      alpha: 0.15
    },
    idle_zone: {
      x: 820, y: 370, w: 430, h: 330,
      label: '休息区',
      color: 0x4fc3f7,
      alpha: 0.15
    }
  },
  B1: {
    server_room: {
      x: 30, y: 30, w: 500, h: 450,
      label: '服务器室',
      color: 0xe94560,
      alpha: 0.15
    },
    database_room: {
      x: 560, y: 30, w: 360, h: 280,
      label: '数据库室',
      color: 0xffd700,
      alpha: 0.15
    },
    mcp_rack: {
      x: 930, y: 30, w: 320, h: 280,
      label: 'MCP工具架',
      color: 0x00ff88,
      alpha: 0.15
    },
    knowledge_hall: {
      x: 560, y: 500, w: 690, h: 200,
      label: '知识库走廊',
      color: 0x4fc3f7,
      alpha: 0.15
    }
  },
  PROJECT: {
    teamlead_zone: {
      x: 200, y: 30, w: 880, h: 320,
      label: 'Team Lead 工位区',
      color: 0xe94560,
      alpha: 0.15
    },
    worker_zone: {
      x: 200, y: 420, w: 880, h: 270,
      label: 'Worker 任务区',
      color: 0x00ff88,
      alpha: 0.15
    }
  }
};
```

---

## 任务三：区域渲染系统

在 FloorScene.js（基类）中实现以下方法：

### 3.1 drawAreas(floorKey)

```javascript
drawAreas(floorKey) {
  const areas = FLOOR_AREAS[floorKey];
  if (!areas) return;

  this.areaZones = {};

  Object.entries(areas).forEach(([name, area]) => {
    // 半透明填充层（鼠标悬停时变亮）
    const fill = this.add.rectangle(
      area.x + area.w / 2,
      area.y + area.h / 2,
      area.w,
      area.h,
      area.color,
      area.alpha
    ).setDepth(2);

    // 边框
    const border = this.add.rectangle(
      area.x + area.w / 2,
      area.y + area.h / 2,
      area.w,
      area.h
    ).setStrokeStyle(2, area.color, 0.8)
     .setFillStyle()
     .setDepth(2);

    // 区域标签（金色像素字）
    const label = this.add.text(
      area.x + area.w / 2,
      area.y + 16,
      area.label,
      {
        fontFamily: "'ArkPixel', 'Courier New', monospace",
        fontSize: '14px',
        fill: '#ffd700',
        stroke: '#000000',
        strokeThickness: 3
      }
    ).setOrigin(0.5, 0).setDepth(3);

    // 可交互 Zone
    const zone = this.add.zone(
      area.x + area.w / 2,
      area.y + area.h / 2,
      area.w,
      area.h
    ).setInteractive({ useHandCursor: true }).setDepth(4);

    zone.on('pointerover', () => {
      fill.setAlpha(area.alpha * 2.5);
    });
    zone.on('pointerout', () => {
      fill.setAlpha(area.alpha);
    });
    zone.on('pointerdown', () => {
      window.dispatchEvent(new CustomEvent('roomClick', {
        detail: { room: name, floor: floorKey, label: area.label }
      }));
    });

    this.areaZones[name] = { fill, border, label, zone };
  });
}
```

### 3.2 各场景的 create() 中调用

- F1Scene.js 的 create() 末尾加：`this.drawAreas('F1')`
- B1Scene.js 的 create() 末尾加：`this.drawAreas('B1')`
- ProjectScene.js 的 create() 末尾加：`this.drawAreas('PROJECT')`

---

## 任务四：坐标调试工具

在 FloorScene.js 基类中加入调试模式（通过 URL 参数 ?dev=1 开启）：

```javascript
initDevTools() {
  const isDev = new URLSearchParams(window.location.search).get('dev') === '1';
  if (!isDev) return;

  // 坐标显示文字
  const coordText = this.add.text(10, 690, 'x:0 y:0', {
    fontFamily: "'ArkPixel', 'Courier New', monospace",
    fontSize: '12px',
    fill: '#00ff88',
    backgroundColor: '#000000',
    padding: { x: 4, y: 2 }
  }).setDepth(100).setScrollFactor(0);

  // 鼠标移动时更新坐标
  this.input.on('pointermove', (pointer) => {
    coordText.setText(`x:${Math.round(pointer.x)} y:${Math.round(pointer.y)}`);
  });

  // 点击时在控制台输出坐标，方便记录区域四个角
  this.input.on('pointerdown', (pointer) => {
    console.log(`[DEV] click x:${Math.round(pointer.x)}, y:${Math.round(pointer.y)}`);
  });

  // 在画布上显示网格参考线（每100px一条）
  const grid = this.add.graphics().setDepth(99).setAlpha(0.15);
  grid.lineStyle(1, 0xffffff);
  for (let x = 0; x <= 1280; x += 100) {
    grid.moveTo(x, 0); grid.lineTo(x, 720);
    this.add.text(x + 2, 2, String(x), {
      fontFamily: 'monospace', fontSize: '10px', fill: '#ffffff'
    }).setDepth(100).setAlpha(0.5);
  }
  for (let y = 0; y <= 720; y += 100) {
    grid.moveTo(0, y); grid.lineTo(1280, y);
    this.add.text(2, y + 2, String(y), {
      fontFamily: 'monospace', fontSize: '10px', fill: '#ffffff'
    }).setDepth(100).setAlpha(0.5);
  }
  grid.strokePath();
}
```

在每个场景的 create() 最后调用：`this.initDevTools()`

---

## 验证步骤

**V1：背景图加载**
```
打开浏览器开发者工具 Network 面板，刷新页面
预期：F1_BG.png / B1_BG.png / PROJECT_BG.png 全部加载成功，无 404
切换楼层，每个楼层显示对应背景图
```

**V2：区域边框显示**
```
切换到 F1 楼层
预期：画面上出现 4 个半透明彩色矩形区域，带金色 ArkPixel 标签文字（会议室/档案室/常驻工位/休息区）
鼠标悬停到某区域，该区域填充变亮
```

**V3：区域点击事件**
```
点击任意区域
打开浏览器控制台
预期：无 JS 报错，CustomEvent roomClick 正常触发
```

**V4：调试工具**
```
在 URL 末尾加 ?dev=1，刷新页面
预期：画面左下角出现绿色坐标显示，背景出现半透明网格参考线
鼠标移动时坐标实时更新，点击时控制台输出 [DEV] click x:xxx y:xxx
```

**V5：布局完整性**
```
检查顶部栏/侧边面板/底部流水线位置和尺寸与之前完全一致
楼层切换按钮正常工作
```

---

## 自检 Checklist（全部勾选后反馈完成）

- [ ] V1：三张背景图全部加载成功，无 404
- [ ] V2：区域边框和标签正常显示，hover 效果正常
- [ ] V3：区域点击事件正常触发，无 JS 报错
- [ ] V4：调试工具正常工作（坐标显示+网格+控制台输出）
- [ ] V5：布局与之前完全一致，楼层切换正常
- [ ] 区域坐标来自 areas.js 配置文件，场景文件内无硬编码坐标
- [ ] 背景图 depth=0，区域填充 depth=2，标签 depth=3，Zone depth=4
- [ ] 未修改任何布局属性（width/height/flex/grid/position）

全部完成后反馈结果，不要继续执行后续阶段。
