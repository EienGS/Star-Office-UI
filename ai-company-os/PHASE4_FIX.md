# 阶段4 UI 修复任务书

## 问题描述

当前阶段4实现的前端使用了 HTML div 矩形模拟房间，CSS 色块模拟 Agent 角色。
这与原项目 `Star-Office-UI/frontend/index.html` 的像素风格差距极大。

**必须修复，修复前不得进入阶段5。**

---

## 核查当前实现的问题

在开始修复前，先逐项确认以下问题是否存在于当前代码中：

- [ ] 房间是否用 HTML div 绘制（而非 Phaser Canvas）
- [ ] Agent 角色是否用 CSS 色块表示（而非 sprite 图片）
- [ ] 背景是否是纯色（而非像素地图纹理 office_bg）
- [ ] 字体是否不是 ArkPixel（而是系统默认字体）
- [ ] 是否缺少加载进度遮罩动画

上述任何一项为"是"，均需按本文档修复。

---

## 根本原因

当前实现没有复用原项目的 Phaser.js 渲染逻辑，而是自行用 CSS 重写了界面。
正确做法是：**直接搬运原项目 `Star-Office-UI/frontend/index.html` 中的 Phaser 渲染代码**，在此基础上扩展多楼层功能，而不是从零重写。

---

## 第一步：复制静态资源（必须先执行）

原项目的所有像素图片、字体、spritesheet 必须复制到新系统，否则 Phaser 无法加载任何素材。

**执行以下命令：**

```bash
# 在项目根目录执行
cp -r Star-Office-UI/static ai-company-os/frontend/static
```

**验证：**
```bash
ls ai-company-os/frontend/static/
```
预期输出中必须包含以下文件（列举关键文件，不限于此）：
- `office_bg_small.png` 或 `office_bg_small.webp`（地图背景）
- `star-idle-spritesheet.png` 或对应 `.webp`（主角色 idle 动画）
- `star-working-spritesheet-grid.png` 或对应 `.webp`（主角色工作动画）
- `guest_anim_1.webp` 至 `guest_anim_6.webp`（访客角色动画）
- `fonts/ark-pixel-12px-proportional-zh_cn.ttf.woff2`（像素字体）
- `sofa-busy-spritesheet.png` 或对应 `.webp`（沙发动画）
- `coffee-machine-spritesheet.png` 或对应 `.webp`（咖啡机动画）
- `serverroom-spritesheet.png` 或对应 `.webp`（服务器动画）
- `error-bug-spritesheet-grid.png` 或对应 `.webp`（报错 Bug 动画）
- `sync-animation-spritesheet-grid.png` 或对应 `.webp`（同步动画）
- `plants-spritesheet.png`、`posters-spritesheet.png`、`cats-spritesheet.png`（装饰）
- `memo-bg.png` 或对应 `.webp`

如有缺失文件，执行 `ls Star-Office-UI/static/` 确认原项目实际有哪些文件，按实际文件名处理。

---

## 第二步：重写 CSS（从原项目直接提取）

**操作：**
1. 读取 `Star-Office-UI/frontend/index.html` 中 `<style>` 标签内的全部 CSS
2. 将以下内容完整复制到 `ai-company-os/frontend/css/base.css`，**一字不改**：

必须包含的 CSS 块（以原文件为准，以下仅为核查清单）：

```
@font-face { font-family: 'ArkPixel'; src: url('/static/fonts/ark-pixel-...woff2') ... }
body { background: #1a1a2e; font-family: 'ArkPixel', 'Courier New', monospace; ... }
#game-container { border: 4px solid #e94560; image-rendering: pixelated; ... }
#game-container canvas { image-rendering: pixelated; ... }
#loading-overlay { background: #1a1a2e; ... }
#loading-progress-bar { background: linear-gradient(90deg, #e94560, #ffd700); ... }
面板类（background: #2c2f3a; border: 2px solid #e94560; ...）
滚动条样式（::-webkit-scrollbar ...）
```

**禁止：**
- 不得修改任何颜色值
- 不得删除 `@font-face` 声明
- 不得用 `px` 替换原始单位
- 不得添加任何新的颜色（紫色 `#7c3aed` 等非原项目颜色一律删除）

---

## 第三步：重写 Phaser 主场景

当前实现的主要错误是用 HTML div 画房间。必须改为 Phaser Canvas 渲染。

### 3.1 搬运核心工具函数

从 `Star-Office-UI/frontend/index.html` 直接复制以下函数到新系统的主 JS 文件，**不得修改函数体**：

```javascript
// 1. WebP 检测
function checkWebPSupport() { ... }
function checkWebPSupportFallback() { ... }
function getExt(pngFile) { ... }  // 注意：star-working-spritesheet.png 始终用 PNG

// 2. 加载进度控制
function updateLoadingProgress() { ... }
function hideLoadingOverlay() { ... }

// 3. 区域坐标
function getAreaRect(area) { ... }  // 含 breakroom/writing/error 的像素坐标
function getAreaPoint(area, idx) { ... }
function randomPointInRect(rect) { ... }
function randomInt(min, max) { ... }
```

### 3.2 Phaser 配置

从原项目搬运完整 config 对象：

```javascript
const IS_TOUCH_DEVICE = ('ontouchstart' in window) || (navigator.maxTouchPoints > 0) || window.matchMedia('(pointer: coarse)').matches;

const config = {
    type: Phaser.AUTO,
    width: 1280,
    height: 720,
    parent: 'game-container',
    pixelArt: true,
    scale: {
        mode: IS_TOUCH_DEVICE ? Phaser.Scale.RESIZE : Phaser.Scale.FIT,
        autoCenter: Phaser.Scale.CENTER_BOTH,
        width: 1280,
        height: 720
    },
    physics: { default: 'arcade', arcade: { gravity: { y: 0 }, debug: false } },
    scene: { preload: preload, create: create, update: update }
};
```

### 3.3 preload() 函数

从原项目完整搬运 `preload()` 函数，包括所有资源加载调用：

必须加载的资源（以原项目实际文件为准）：
- `office_bg`：地图背景图片
- `star_idle`：主角色 idle spritesheet（frameWidth: 128, frameHeight: 128）
- `star_researching`：主角色 researching spritesheet（frameWidth: 128, frameHeight: 105）
- `sofa_busy`：沙发动画 spritesheet（frameWidth: 256, frameHeight: 256）
- `plants`：植物 spritesheet（frameWidth: 160, frameHeight: 160）
- `posters`：海报 spritesheet（frameWidth: 160, frameHeight: 160）
- `coffee_machine`：咖啡机 spritesheet（frameWidth: 230, frameHeight: 230）
- `serverroom`：服务器 spritesheet（frameWidth: 180, frameHeight: 251）
- `error_bug`：报错 Bug spritesheet（frameWidth: 180, frameHeight: 180）
- `cats`：猫咪 spritesheet（frameWidth: 160, frameHeight: 160）
- `star_working`：主角色工作 spritesheet（frameWidth: 230, frameHeight: 144）
- `sync_anim`：同步动画 spritesheet（frameWidth: 256, frameHeight: 256）
- `desk_v2`：桌子图片
- `flowers`：花盆 spritesheet（frameWidth: 65, frameHeight: 65）
- `guest_anim_1` 至 `guest_anim_6`：访客角色 spritesheet（frameWidth: 32, frameHeight: 32）

### 3.4 create() 函数

从原项目搬运 `create()` 函数的以下关键部分：

**背景和家具（必须）：**
```javascript
this.add.image(640, 360, 'office_bg');  // 像素地图背景
sofa = this.add.sprite(670, 144, 'sofa_busy').setOrigin(0, 0);
// 沙发动画
this.anims.create({ key: 'sofa_busy', frames: ..., frameRate: 12, repeat: -1 });
```

**主角色动画（必须）：**
```javascript
this.anims.create({ key: 'star_idle', frames: ..., frameRate: 12, repeat: -1 });
this.anims.create({ key: 'star_working', frames: ..., frameRate: 12, repeat: -1 });
this.anims.create({ key: 'star_researching', frames: ..., frameRate: 12, repeat: -1 });
this.anims.create({ key: 'error_bug', frames: ..., frameRate: 12, repeat: -1 });
this.anims.create({ key: 'sync_anim', frames: ..., frameRate: 12, repeat: -1 });
this.anims.create({ key: 'coffee_machine', frames: ..., frameRate: 12.5, repeat: -1 });
this.anims.create({ key: 'serverroom_on', frames: ..., frameRate: 6, repeat: -1 });
```

**访客角色动画（必须）：**
```javascript
for (let i = 1; i <= 6; i++) {
    this.anims.create({
        key: `guest_anim_${i}_idle`,
        frames: this.anims.generateFrameNumbers(`guest_anim_${i}`, { start: 0, end: 7 }),
        frameRate: 8,
        repeat: -1
    });
}
```

**装饰物（必须）：**
- 植物（3株，坐标：565,178 / 230,185 / 977,496）
- 海报（252,66）
- 猫咪（94,557）
- 咖啡机（659,397）
- 桌子（218,417）
- 花盆（310,405）
- 服务器（1021,142）
- 错误 Bug 角色（1007,221，默认隐藏）
- 同步动画精灵（1157,592，默认停在第0帧）
- 牌匾文字（画面底部居中）

**主角色 star（必须）：**
```javascript
star = game.physics.add.sprite(areas.breakroom.x, areas.breakroom.y, 'star_idle');
star.setOrigin(0.5);
star.setScale(1.4);
star.setAlpha(0.95);
star.setDepth(20);
star.setVisible(false);  // 默认隐藏，idle 状态用 sofa 表示
```

### 3.5 访客 Agent 渲染

从原项目完整搬运 `renderGuestAgentsInScene()` 函数：
- 访客角色使用 `guest_anim_x` spritesheet，播放 idle 动画，scale: 4.0
- 名字文字使用 ArkPixel 字体，白色，带黑色描边
- 位置根据 Agent 状态映射到对应区域坐标

---

## 第四步：楼层扩展（在原项目渲染基础上扩展）

上述修复完成后，楼层切换功能需要在 Phaser 场景基础上扩展，而不是替换：

### 楼层切换实现方式

**不要**为每个楼层创建独立的 Phaser Game 实例（代价极高）。

**正确方式**：使用 Phaser 的 Scene 管理器，每个楼层是一个独立 Scene：

```javascript
// 注册多个场景
const config = {
    ...
    scene: [F1Scene, B1Scene, ProjectFloorScene]  // 多场景注册
};

// 楼层切换
function switchFloor(floorKey) {
    game.scene.stop(currentFloor);
    game.scene.start(floorKey);
    currentFloor = floorKey;
}
```

**F1Scene（公共层）**：包含原项目完整的 create() 内容（家具、装饰、访客渲染）
**B1Scene（基础设施层）**：复用同一套背景 `office_bg`，替换家具为服务器室布局
**ProjectFloorScene（项目层）**：复用同一套背景，工位动态生成

### 各楼层共享的内容（不重复写，用继承或工具函数）

- `preload()`：所有场景共享，只加载一次（放在 Boot Scene 或第一个 Scene）
- `renderGuestAgentsInScene()`：每个场景各自调用，传入该楼层对应的 Agent 列表
- `BUBBLE_TEXTS`：全局共享
- 状态轮询逻辑：全局单例，各场景监听同一个状态源

---

## 第五步：HTML 结构修复

当前 HTML 中如果有用于画房间的 div（如 `.room`、`.floor-container` 等），全部删除。

主区域结构应为：

```html
<div id="loading-overlay">
    <div id="loading-text">正在加载...</div>
    <div id="loading-progress-container">
        <div id="loading-progress-bar"></div>
    </div>
</div>

<div id="app-header">
    <!-- 顶部栏：楼层切换按钮、项目定位、告警 -->
</div>

<div id="game-container">
    <!-- Phaser Canvas 自动注入此处，不放任何子元素 -->
</div>

<div id="sidebar">
    <!-- 侧边面板：消息/Agent/指令 Tab -->
</div>

<div id="pipeline-bar">
    <!-- 底部流水线 -->
</div>
```

`#game-container` 内部不放任何 HTML 子元素，Phaser 自动注入 canvas。

---

## 验证步骤（全部通过才算修复完成）

### V-FIX-1：静态资源加载
```
打开浏览器开发者工具 → Network 面板 → 刷新页面
预期：无任何 .png / .webp / .woff2 文件报 404
如有 404：检查 ai-company-os/frontend/static/ 目录，确认文件是否复制成功
```

### V-FIX-2：字体渲染
```
打开页面，观察所有文字
预期：所有文字（标题、按钮、气泡、名字）均使用像素字体（ArkPixel）
判断方法：文字应呈现明显的像素点阵风格，而不是普通的圆润系统字体
如不对：检查 @font-face 是否正确加载，字体文件路径是否 /static/fonts/ark-pixel-...woff2
```

### V-FIX-3：地图背景
```
打开 F1 楼层
预期：主区域（Phaser Canvas）显示像素风格办公室地图（office_bg 图片），不是纯色背景或 CSS 网格
如不对：检查 create() 中是否有 this.add.image(640, 360, 'office_bg')
```

### V-FIX-4：Agent 角色渲染
```
创建一个 Agent 并设置为 active 状态
预期：在 Phaser Canvas 中出现像素 sprite 角色（guest_anim_x），带 idle 动画，角色下方有名字文字
如不对：检查 renderGuestAgentsInScene() 是否被调用，guest_anim_x 资源是否加载成功
```

### V-FIX-5：动画效果
```
打开页面，静置10秒观察
预期：沙发上有角色动画（sofa_busy 循环）、咖啡机有烹煮动画、植物/猫咪可点击切换外观
如不对：检查 anims.create() 是否正确定义，对应 sprite 是否调用了 anims.play()
```

### V-FIX-6：加载遮罩
```
硬刷新页面（Ctrl+Shift+R / Cmd+Shift+R）
预期：
  1. 出现深蓝色（#1a1a2e）全屏遮罩
  2. 遮罩中央有 ArkPixel 字体的加载文字
  3. 下方有进度条，颜色为 #e94560 → #ffd700 渐变
  4. 资源加载完成后，遮罩淡出消失
如不对：检查 #loading-overlay HTML 结构和 updateLoadingProgress() / hideLoadingOverlay() 函数
```

### V-FIX-7：楼层切换
```
点击顶部 B1 / F1 / F2 按钮
预期：
  1. Canvas 内容切换（不同楼层有不同的房间布局）
  2. 切换有过渡效果（淡入淡出或即时切换均可）
  3. 切换后地图背景仍是像素风格，不是纯色
如不对：检查 Phaser Scene 管理器配置，确认每个 Scene 的 create() 中都有 this.add.image(640, 360, 'office_bg')
```

### V-FIX-8：视觉对比
```
同时打开原项目（Star-Office-UI 目录下运行）和新系统
对比以下元素，视觉效果必须一致或更好，不得更差：
  1. 背景色（深蓝黑 #1a1a2e）✓/✗
  2. 面板边框颜色（#e94560 红色）✓/✗
  3. 标题金色文字（#ffd700）✓/✗
  4. 地图像素纹理（家具、地板纹理清晰可见）✓/✗
  5. Agent 像素 sprite 角色（有动画，不是色块）✓/✗
  6. 气泡打字机效果（文字逐字出现）✓/✗

上述6项必须全部 ✓，任意一项 ✗ 须修复后重新验证。
```

---

## 自检 Checklist（修复完成后逐项勾选）

- [ ] V-FIX-1 无静态资源 404 通过
- [ ] V-FIX-2 ArkPixel 字体渲染通过
- [ ] V-FIX-3 地图背景（office_bg 像素纹理）通过
- [ ] V-FIX-4 Agent 像素 sprite 角色通过
- [ ] V-FIX-5 动画效果（沙发/咖啡机/植物）通过
- [ ] V-FIX-6 加载遮罩（进度条渐变色）通过
- [ ] V-FIX-7 楼层切换通过
- [ ] V-FIX-8 视觉对比6项全部 ✓ 通过
- [ ] HTML 中无用于画房间的 div 残留
- [ ] Phaser Canvas 是主区域唯一的渲染容器
- [ ] 未引入任何非原项目的颜色（如紫色 #7c3aed）

**全部勾选后，向用户报告修复完成，等待确认后再进入阶段5。**

---

## 禁止事项

- 禁止用 HTML div + CSS 边框模拟房间轮廓
- 禁止用 CSS 色块替代 sprite 角色
- 禁止用纯色背景替代 office_bg 地图
- 禁止重新设计颜色系统（必须使用原项目颜色）
- 禁止删除或修改 `Star-Office-UI/` 目录下的任何文件
- 禁止在未通过所有验证步骤的情况下继续阶段5
