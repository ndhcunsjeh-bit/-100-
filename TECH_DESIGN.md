# TECH_DESIGN.md — 技术设计文档

> 项目：是男人就下100层（HTML5 Canvas 网页小游戏）
> 阶段：Vibe Coding 工作流 · 第 3 步「技术设计」
> 版本：v1.0　｜　日期：2026-09-24　｜　上游文档：PRD.md
> 状态：与当前 `index.html` 实现一致（行号引用以 v0.8 版本为准）

---

## 1. 技术选型

| 决策项 | 选择 | 理由 |
|---|---|---|
| 渲染 | **原生 HTML5 Canvas 2D** | 全代码绘制像素图形，不需要任何图片素材；`fillRect` 拼像素比 DOM 便宜得多 |
| 语言 | **原生 JavaScript（ES6，无 TypeScript）** | 单文件、无构建、无类型系统负担；AI 改动时"改完即刷新可用" |
| 结构 | **单文件 `index.html`** | 作业要求可交付、双击即玩；也避免 AI 在多个文件间改错位置 |
| 架构 | **面向过程 + 数据驱动**（不用类、不用组件框架） | 状态量不到 20 个，用全局状态 + 纯函数最直观，AI 也最不容易改乱 |
| 循环 | `requestAnimationFrame` + **`dt` 钳位** | 帧率无关的物理积分，同时防止切后台回来时"一帧跳半秒" |
| 音效 | **WebAudio API 合成方波**（不使用音频文件） | 0 素材依赖，方波 = 8-bit 音色，正好对味 |
| 存储 | `localStorage` | 最高纪录持久化，无需后端 |
| 性能技巧 | 预生成纹理 `createPattern` 平铺 | 斑驳底纹只画一次 72×72 离屏画布，之后平铺 |
| 明确不用 | 任何框架（React/Vue）、构建工具（Vite/Webpack）、图片/音频资源 | 违背"零依赖单文件"的产品基调 |

---

## 2. 项目结构

单文件内用注释分区，共 8 个区块（改代码前按区块定位）：

```
index.html
├── <head>
│   └── <style>            像素风 UI：居中容器、开始按钮、触屏虚拟按键
├── <body>
│   ├── <canvas 420×640>
│   ├── <div id="diff"> 简单/困难/地狱 三格难度按钮
│   ├── <button> 开始/再来一次
│   └── <div class="pad"> ×2  触屏左右键（pointer:coarse 时才显示）
└── <script>  "use strict"   —— 8 个分区 ——
    ├── [1] 常量与调色板       W/H/FIELD/WALL_W、G/MOVE、PITCH/PLAT_W/PLAT_H、DEPTH_UNIT、FLIP_TIME/FLIP_DROP、SPRING_LOCK、BELT_SPEED、DIFF 难度表、C 调色板
    ├── [2] 音效              beep(freq, dur, type, vol) 方波合成
    ├── [3] 底纹贴图          texNoise：先画积木小纹理，再 createPattern 平铺
    ├── [4] 绘制小工具        px() 像素描边矩形 / pxText() 等宽像素文字
    ├── [5] 状态与初始化      reset() / mkPlat() / deepestY() / nextPlatX() / pickType() / spawnPlat()
    ├── [6] 输入              keydown/keyup + bindHold() 触屏绑定 + selectDiff() / startGame()
    ├── [7] 逻辑              update(dt) 物理、碰撞、相机 / decayFx(dt) 特效衰减 / showBanner()
    └── [8] 渲染              loop() → render() → drawBackground/Walls/Ceiling/Plat/Man/HUD/neon
```

### 2.1 文件规模

| 项 | 数值 |
|---|---|
| 总行数 | 797 行（含 CSS/HTML） |
| JS 行数 | 约 738 行 |
| 外部依赖 | 0 |
| 资源文件 | 0 |

---

## 3. 数据模型

### 3.1 全局状态变量

```js
let player, plats, camY, wallScroll, floor, hp, state, shake, genY, nextId, particles,
    flashT, overReason, best, startDepth, bannerT, passed100,
    bannerTitle, bannerSub, bannerColor, diffKey, D;
```

| 变量 | 类型 | 含义 |
|---|---|---|
| `player` | object | 玩家实体（见 3.2） |
| `plats` | array | 全部平台实体（含尖刺/弹跳/翻转铁板/传送带） |
| `camY` | number | **相机在世界坐标系中的 y 偏移**（核心，决定所有绘制坐标） |
| `wallScroll` | number | 砖墙滚动累计量（独立于 camY，仅用于墙体纹理） |
| `floor` | number | 当前层数（= 踩到的最深平台 id） |
| `hp` | number | 生命值，初始 100 |
| `state` | string | `'ready'` \| `'play'` \| `'over'` |
| `shake` | number | 受击抖动强度（像素），随时间衰减 |
| `genY` | number | 生成高度的**兜底值**（仅在无任何平台时使用） |
| `nextId` | number | 平台自增 id，同时也是"层号"的来源 |
| `particles` | array | 粒子（踩踏/受击） |
| `flashT` | number | 闪红计时 |
| `overReason` | string | 死亡原因 `'spike'` \| `'top'` |
| `best` | number | 当前难度的最高层数（`localStorage['down100_best_' + diffKey]`，**按难度分键**） |
| `startDepth` | number | 出生板的世界 y，用于计算"深度因子"决定陷阱概率 |
| `bannerT` | number | 横幅剩余显示时间（开局难度提示 / 100 层达成共用） |
| `bannerTitle` `bannerSub` `bannerColor` | string | 当前横幅的标题 / 副标题 / 颜色 |
| `passed100` | boolean | 100 层里程碑是否已触发（防重复弹窗） |
| `diffKey` | string | 当前难度 `'easy'` \| `'hard'` \| `'hell'`，持久化于 `localStorage['down100_diff']` |
| `D` | object | 当前难度的参数对象（`DIFF[diffKey]` 的引用），逻辑里一切难度相关数值都从 `D.xxx` 取 |

### 3.2 玩家实体

```js
player = {
  x, y,        // 世界坐标（左上角）
  w: 20, h: 30,
  vx, vy,      // 速度（px/s）
  on: null,    // 当前所站平台引用；null = 腾空
  face: 1,     // 朝向 +1 / -1（渲染时用它做 scale 翻转）
  inv: 0,      // 无敌帧剩余时长（秒）
  anim: 0,     // 走路动画相位
  drain: 0,    // 被天花板挤压的累积计时（每 0.3 秒扣一次血）
  burn: 0,     // 站在尖刺上的累积灼烧计时（每 0.5 秒响一声）
  lock: 0      // 弹簧竖直锁定剩余时长（>0 期间屏蔽左右输入，见 5.13）
};
```

**为什么用 `on` 引用而不是布尔量**：需要每帧复检"我还站在这块板上吗"（板是否已失效、是否被移出数组、是否被抬到边界、我的 x 是否还在板面之上），拿到引用才能一次性判完。

### 3.3 平台实体

```js
{ id, x, y, w: 92, h: 18, type, st, t, vy, dir, anim }
```

| 字段 | 含义 |
|---|---|
| `id` | 自增编号，兼作"层号"（踩到更深 id 就 +1 层） |
| `type` | `'normal'` \| `'spike'` \| `'spring'` \| `'flip'` \| `'belt'` |
| `st` | 翻转铁板状态机：`0` 正常 → `1` 翻转中（0.42s，翻到 55% 撤支撑） → `2` 已翻面（不可站） |
| `t` | 状态倒计时（翻转铁板用） |
| `vy` | 预留速度字段（当前无类型使用，保留以兼容旧结构） |
| `dir` | 传送带方向：`-1` 向左 / `+1` 向右（非传送带为 `0`） |
| `anim` | 外观动画相位（传送带履带齿滚动、弹跳板斜纹） |

---

## 4. 坐标系设计（本项目最关键的一条约定）

> 这段是踩过坑之后才写下来的，**所有后续改动都必须遵守**。

| 坐标系 | 定义 | 用途 |
|---|---|---|
| **世界坐标** | 以出生点为原点的绝对坐标，`y` 随游戏进程**不断增大** | 所有实体的 `x/y` 都存在世界坐标里；物理计算、碰撞判定全部在世界坐标完成 |
| **屏幕坐标** | Canvas 像素坐标，`(0,0)` 在画布左上角 | **只在绘制那一刻**把世界坐标转成屏幕坐标 |
| **相机偏移 `camY`** | 世界坐标 → 屏幕坐标的 y 位移量 | 唯一转换桥梁 |

转换规则（**只有一处允许做这个减法**）：

```js
屏幕 y = 世界 y - camY
```

补充约定：

1. **`x` 方向不做相机滚动**（横向是固定的一屏宽），所以 x 无需转换。
2. 逻辑计算（碰撞、钳位、边界）**一律不出现 `camY`**，只有两处例外，且都必须显式说明理由：
   - 平台回收/生成的前瞻判断（`pl.y > camY - 90`、`deepestY() < camY + 视野底部 + 420`）——这里的 `camY` 代表"当前可视区域"，属于逻辑与相机耦合的必然情况。
   - 天花板判定（`p.y - camY < CEIL`）——因为天花板是**固定在屏幕上的**（它不随世界滚动），所以必须换算到屏幕坐标比较。
3. 相机只下不上：`if (target > camY) camY = target`。
4. 相机目标位置统一公式：`camY = player.y - (F_TOP + FIELD.h * 0.42)` —— 让玩家固定在屏幕高度 42% 处，留出下方更多可视空间。

---

## 5. 关键技术与算法

### 5.1 主循环与时间步长

```js
let last = performance.now();
function loop(now){
  const dt = Math.max(0, Math.min(0.033, (now - last) / 1000));  // ← 关键钳位
  last = now;
  if (state === 'play') update(dt);
  decayFx(dt);          // ← 无条件调用（结束界面的抖动/闪红也要衰减）
  render();
  requestAnimationFrame(loop);
}
```

三个设计点：

1. **`dt` 上限 33ms**：切后台再回来时 `now - last` 可能是好几秒，不钳位就会一帧跨越整屏导致穿模。下限 0 用于防负数 dt。
2. **`decayFx` 独立于 `update`**：特效衰减必须无条件执行（曾经写在 `update` 里，结果进入结束态后画面永远在抖）。
3. **物理用 `dt` 积分，渲染用整数像素**（`Math.round`），保证像素风不糊。

### 5.2 重力与限速

```js
p.vy = Math.min(p.vy + G * dt, 760);   // G = 1800，760 为终速
p.y += p.vy * dt;
```

`Math.min` 的限速**不是可选项**：没有它，长距离自由落体一帧位移可以超过 100px，而一帧只做一次位置比较的碰撞检测会直接漏检（穿模）。限速把单帧最大位移压到 `760 × 0.033 ≈ 25px`，这才让"整帧扫掠检测"有意义。

### 5.3 单向碰撞检测（扫掠法）

**问题**：朴素的位置重叠检测有两个死穴——① 高速穿过（本帧起点在板上方、终点在板下方，中间没有采样点）；② 从下方被顶起时会被"推"回平台上方，看起来像被卡住。

**方案**：记录本帧起点脚底 → 终点脚底，只要这段区间**跨越**了某块板的顶面，就算踩上：

```js
const prevFeet = p.y + p.h;      // 本帧开始时的脚底（在世界坐标）
p.y += p.vy * dt;
const feet = p.y + p.h;          // 本帧结束时的脚底

if (p.vy > 0) {                  // 只在下落时检测 → 这就是"单向"
  let hit = null;
  for (const pl of plats) {
    if (pl.st === 2) continue;                    // 已翻面的翻转铁板不可站
    if (pl.y - camY < CEIL_GATE) continue;        // 边界附近的板不可站（防被压扁）
    if (feet >= pl.y && prevFeet <= pl.y + 12 &&  // 本帧跨越了板顶面
        p.x + p.w > pl.x + 3 && p.x < pl.x + pl.w - 3) {  // 水平重叠（收 3px 内缩，防边缘悬空）
      if (!hit || pl.y < hit.y) hit = pl;         // 取最高的那块（y 最小）
    }
  }
  if (hit) { /* 按 type 分流处理 */ }
}
```

四个必须理解的细节：

| 细节 | 作用 |
|---|---|
| `p.vy > 0` 才检测 | **这就是"单向"**：只关心下落，上升时完全不管，自然就穿过去了 |
| `prevFeet <= pl.y + 12` | 允许 12px 的容差，避免"上一帧脚底刚好在板上方 1px"这类边界抖动 |
| `if (!hit \|\| pl.y < hit.y)` | 一帧穿过两块板时**必须踩上面那块**。曾经误写成命中即 `break`，导致玩家"瞬移到下层平台" |
| `pl.y - camY < CEIL_GATE` | 与天花板保持 96px 的安全距离，这一区间的板子不允许站立（否则角色会被板子顶进天花板，形成既不能站也不能走的死循环） |

### 5.4 落地后的支撑复检

踩上平台后，玩家是"跟随"这块板移动的（`p.y = pl.y - p.h`），**必须每帧复检支撑条件**，否则会出现"走出板边却浮在空中"的粘滞 bug：

```js
if (p.on) {
  const pl = p.on;
  if (pl.st === 2 || plats.indexOf(pl) < 0 || pl.y - camY < CEIL_GATE ||
      p.x + p.w <= pl.x + 2 || p.x >= pl.x + pl.w - 2) {
    p.on = null;      // 失去支撑 → 让重力接管
  } else {
    p.y = pl.y - p.h; p.vy = 0;
    if (pl.id > floor) floor = pl.id;   // 只有站在更深的板上才 +1 层
    if (pl.type === 'spike') spikeBurn(dt);   // 尖刺是实体：站着不躲就一直掉血
    else p.burn = 0;                          // 离开尖刺即清空灼烧计时
  }
}
```

**为什么尖刺的伤害要挂在这里**：尖刺是**可站立的实体平台**（`p.on = 尖刺板`），因此"站在尖刺上"这个状态会被每帧复检，持续伤害就有了天然的驱动点；玩家一旦走出板边，复检立即清空 `p.on` → 灼烧自动停止，**不需要额外的停止逻辑**。

### 5.5 持续伤害（尖刺灼烧）

```js
function spikeBurn(dt){
  const p = player;
  p.burn += dt;
  hp -= D.burn * dt;                   // ← 按时间积分，不是按帧扣常数；D.burn 随难度
  if (shake  < 3)     shake  = 3;      // 轻微屏抖（不覆盖更大的受击抖动）
  if (flashT < 0.10) flashT = 0.10;
  puff(p.x + rnd(0, p.w), p.y + p.h - 4, '#FF6B7A', 1);   // 每帧 1 颗血花
  if ((p.burn % 0.5) < dt) beep(240, .06, 'square', .03); // 每 0.5 秒一声
  if (hp <= 0) { hp = 0; gameOver('spike'); }
}
```

三个设计要点：

| 要点 | 说明 |
|---|---|
| **与 `hurt()` 分离** | `hurt()` 是"一次性碰撞伤害"（含 1.1 秒无敌帧），`spikeBurn()` 是"持续伤害"（不吃无敌帧）。两者**不能合并**——如果灼烧走 `hurt()`，第一次判定就会给自己挂上 1.1 秒无敌，玩家站在尖刺上反而无伤 |
| **按 `dt` 积分** | `hp -= 22 * dt` 保证掉血速度与帧率无关；缺点是 `hp` 会变成浮点数 |
| **`shake`/`flashT` 取较大值** | 用 `if (x < n) x = n` 而不是直接赋值，避免"灼烧把一次大受击的强烈抖动削掉" |

> ⚠️ **浮点血量的显示陷阱**：`hp` 变成小数后，HUD 里如果直接 `String(hp)`，会显示成 `69.49` 这种一眼穿帮的文本。HUD 必须取整——本项目用 `Math.ceil(hp)`（向上取整，让"还剩 1 点血"在视觉上更符合直觉）。

**为什么尖刺要从"扎穿"改成"实体"**：初版尖刺是"踩一下 -25 血并被扎穿继续下落"，但玩家会发现**最优解是"主动踩尖刺"**——因为它扣的血比"错过平台被顶到天花板"少，而且踩完还能继续下潜。这直接反转了玩法意图。改成实体 + 灼烧之后，尖刺真正变成了"必须躲开"的东西。随机乱走机器人的平均成绩从 100.9 层掉到 55.6 层，正好印证它恢复了威胁性（见 BUILD_LOG v0.9）。

### 5.6 平台生成算法（按层生成）

**旧方案（错误）**：用一个自增计数器 `genY += PITCH` 算生成高度。
**为什么错**：世界在滚动，所有平台每帧都在 `y -= rise*dt`。计数器记录的却是"生成那一刻的坐标"，两次生成之间世界已经移动了一段距离，于是实际间距 = `PITCH + rise*dt_frames`，表现为"明明写死等距，实测却是 104 / 113.55 / 115.29"。

**新方案：相对最深平台的当前坐标生成**

```js
function spawnPlat(){
  const base = deepestY();                     // 当前最深的可站平台（实时值）
  const y = isFinite(base) ? base + PITCH : genY;   // 恒为 PITCH = 88
  genY = y;
  ...
}
// 生成前瞻：视野下方 420px 内必须始终有板
while (deepestY() < camY + FIELD.y + FIELD.h + 420) spawnPlat();
```

`deepestY()` **不再排除**任何平台（含 `st === 2` 的翻转铁板）。原因：v0.11 起「翻转铁板」只绕中轴换面、**不移动位置**（与旧易碎板的"塌落下沉"不同），因此它仍是可靠的生成基准；若排除它，生成高度会跳回上一块，破坏 `PITCH` 等距。所有平台实体（无论状态）都参与取最深值。

**水平落点（`nextPlatX`）**：

```js
const minX = FIELD.x + 4, maxX = FIELD.x + FIELD.w - 4 - PLAT_W;  // 全宽范围内
let x = rnd(minX, maxX);
// 最多重试 12 次，保证与上一块错开半个板宽 → 同一槽位不重叠
for (let t = 0; t < 12 && lastX !== null && Math.abs(x - lastX) < PLAT_W * 0.5; t++) x = rnd(minX, maxX);
lastX = x;
```

**为什么水平不做硬性 `minGap / maxGap` 限制**（最初的设计是限制的，后来放开了）：
本游戏中"错过平台"**并不致命**——踩空只是继续往下掉，还能接住更下面的板。真正致命的是①下方完全没有板（垂直断层）②被平台顶到天花板压扁。因此：
- **垂直方向必须刚性等距**（`PITCH = 88` 恒定）→ 保证视野下方永远有板、永远可接续；
- **水平方向只需防止两块板完全重叠**（错开半块板宽），无需限制跨度。

这一条从"限制横向跨度"到"锁定垂直间距"的转变，是 v0.4 迭代最重要的认知升级。

### 5.7 相机滚动

```js
const target = p.y - (F_TOP + FIELD.h * 0.42);
if (target > camY) camY = target;    // 只下不上
```

世界所有物体每帧统一 `y -= rise*dt`（平台整体上升），配合相机同步下移，形成"平台推着你往下走"的压迫感。上升速度随深度递增，且**三档难度各有自己的基础值 / 递增系数 / 上限**：

```js
const rise = Math.min(D.riseMax, D.riseBase + floor * D.riseGain);
```

| 难度 | 公式 | 0 层实速 | 上限触顶层 |
|---|---|---|---|
| 简单 | `min(190, 54 + 层×1.45)` | 54 px/s | 94 层 |
| 困难 | `min(265, 68 + 层×1.9)` | 68 px/s | 104 层 |
| 地狱 | `min(345, 88 + 层×2.5)` | 88 px/s | 103 层 |

重力 `G` 与水平移动 `MOVE` **不参与难度分档**，三档手感一致，难度只体现在节奏、陷阱密度与伤害上（理由见 AGENTS.md §4「难度档位的三条规矩」）。

### 5.8 天花板硬约束

```js
if (p.y - camY < CEIL) {           // CEIL = 62（屏幕坐标）
  p.y = camY + CEIL;               // ① 硬钳位，保证不会跑出视口
  if (p.on) p.on = null;           // ② 释放支撑（否则会与板子互相拉扯）
  if (p.vy < 0) p.vy = 0;
  p.vy = Math.max(p.vy, 140);      // ③ 强制往下推
  p.drain += dt;
  if (p.drain > 0.3) { p.drain = 0; hurt(12, 'top'); p.vy = 140; }  // ④ 每 0.3s 扣血
}
```

"硬约束"的含义：无论玩家怎么按方向键、无论平台怎么推，角色**必须**回到可视区内。同时用"每 0.3 秒扣 12 血"替代"接触即死"，让玩家有 2–3 秒的挣扎窗口——这是体验上的刻意设计，不是漏洞。

### 5.9 渲染管线

```
loop()
 └─ render()
     ├─ setTransform 复位 + imageSmoothingEnabled = false
     ├─ drawBackground()   蓝色渐变 + 噪点纹理（视差 0.35）
     ├─ save() [+ shake 随机位移]
     │   ├─ save() → clip(游戏区)                 ← 关键：裁剪掉游戏区外的绘制
     │   │   ├─ plats.forEach(drawPlat)           平台（世界y - camY）
     │   │   ├─ particles 粒子
     │   │   ├─ drawMan(player)                   玩家（世界y - camY）
     │   │   └─ 100 层横幅
     │   └─ restore()
     ├─ drawWalls()   左右砖墙（独立滚动量 wallScroll）
     ├─ drawCeiling() 天花板锯齿（固定在屏幕顶部）
     ├─ drawHUD()     生命条 + 层数（固定在屏幕顶部）
     ├─ 闪红覆盖层
     └─ 若 state !== 'play' → 半透明黑遮罩 + 霓虹标题 / 结束信息
```

三个约定：

1. **游戏区裁剪**（`clip`）让平台与角色在进入 HUD 区域时被切掉，不会"画到血条上"。
2. **绘制顺序 = 层次**：背景 → 平台/角色 → 墙体 → 天花板 → HUD → 遮罩。墙体与天花板必须画在角色之后，才能形成"角色被关在里面"的空间感。
3. **所有绘制函数自己算屏幕坐标**（`pl.y - camY`），调用方不传坐标。

### 5.10 像素绘制工具

```js
function px(x, y, w, h, fill, stroke)   // 取整 + 填充 + 1px 黑描边（像素风的统一入口）
function pxText(txt, x, y, size, color, align)   // 等宽像素字体 + 对齐
function neon(txt, x, y, size, color)   // shadowBlur 发光 + 黄色描边，用于标题
```

玩家小人（20×30）用一组 `part(x, y, w, h, color)` 拼装：红帽、脸、眼睛、黄衣、手臂、背带裤、腿、鞋，每个部件先画黑色描边矩形再填色，天然形成像素描边的块面感。走路时用 `Math.sin(p.anim)*3` 驱动左右腿错位，腾空时在身体两侧补两条竖线作为"腾空姿态"。左右翻转用 `ctx.scale(face, 1)`。

### 5.11 音效设计

```js
function beep(freq, dur, type, vol)  // WebAudio 方波 + 指数衰减包络
```

| 事件 | 频率序列 |
|---|---|
| 踩踏 | 300Hz × 0.045s |
| 弹跳 | 660Hz → (70ms) → 920Hz |
| 受击 | 170Hz × 0.18s |
| 站在尖刺上（灼烧） | 240Hz × 0.06s，每 0.5 秒一次 |
| 死亡 | 130Hz × 0.5s |
| 100 层 | 523 → 659 → 784 → 1046Hz |

注意：`AudioContext` 必须在**用户手势之后**创建（浏览器自动播放策略），因此 `beep()` 内部做惰性初始化，并用 `try/catch` 包裹——音效失败绝不能影响游戏运行。

### 5.12 难度档位（数据驱动的单一真源）

三档难度**不是三套 if/else**，而是一张表 `DIFF` + 一个当前引用 `D`：

```js
const DIFF = {
  easy: { name:'简单', riseBase:54, riseGain:1.45, riseMax:190,
          spikeMul:0.55, flipMul:0.75, burn:7,  hitSpike:7,  hitTop:7,
          color:'#00E03C', hint:'上升较慢 · 陷阱稀少 · 伤害较低' },
  hard: { name:'困难', riseBase:68, riseGain:1.9,  riseMax:265,
          spikeMul:1.0,  flipMul:1.0,  burn:11, hitSpike:12, hitTop:12,
          color:'#FFC400', hint:'标准节奏 · 原汁原味' },
  hell: { name:'地狱', riseBase:88, riseGain:2.5,  riseMax:345,
          spikeMul:1.5,  flipMul:1.3,  burn:17, hitSpike:19, hitTop:19,
          color:'#FF2A2A', hint:'高速上升 · 陷阱密集 · 伤害翻倍' }
};
const DIFF_ORDER = ['easy','hard','hell'];
let diffKey = 'hard', D = DIFF[diffKey];      // 逻辑里只认 D，不认难度名
```

难度相关的取值点只有 5 处，**全部通过 `D.xxx` 间接读取**：

| 取用点 | 表达式 |
|---|---|
| 平台上升速度（5.7） | `min(D.riseMax, D.riseBase + floor*D.riseGain)` |
| 尖刺生成概率（5.6） | `(0.15 + min(0.13, d*0.004)) * 0.75 * D.spikeMul` |
| 翻转铁板生成概率（5.6） | `(0.12 + min(0.14, d*0.005)) * D.flipMul` |
| 尖刺灼烧（5.5） | `hp -= D.burn * dt` |
| 撞尖刺 / 挤天花板的一次性伤害 | `hurt(D.hitSpike)` / `hurt(D.hitTop)` |

> 另有**不随难度变**的两项概率常量：弹跳板 `pSpring = 0.17`、传送带 `pBelt = 0.10`。

**设计约束**：
- `G`（重力）与 `MOVE`（水平移动）**不入表**，三档完全一致 —— 难度只改节奏/密度/伤害，不改操作手感。
- 三档**不改规则**：尖刺在所有难度都是"可站立 + 持续掉血"。规则分档会让每个 bug 都要修三遍。
- 各类型陷阱概率是**累加阈值**判定，故 `pSpike + pSpring + pFlip + pBelt < 1` 必须恒成立（极深处三档分别为 58.2% / 74.1% / 92.3%）。
- 纪录按难度分键：`localStorage['down100_best_' + diffKey]`；切换难度时 `best` 重读，避免跨档污染。

**类型判定抽成独立函数** `pickType(d)`（而不是内联在 `spawnPlat()` 里），这样验证脚本可以直接调用它做 20 万次采样统计——可测试性倒逼出的结构，不要合并回去。

### 5.13 特殊平台的机制实现（v0.11）

三种平台类型各有独立的状态/计时逻辑，全部以常量参数化：

**① 翻转铁板 `flip`** —— 踩上后绕水平中轴翻转，翻转过程中把人漏下去。
```js
const FLIP_TIME = 0.42;   // 从踩上到彻底翻过去所需时间
const FLIP_DROP = 0.55;   // 翻到 55% 时撤支撑
// 平台循环内（st===1 时推进）：
pl.t += dt;
if (!pl.droped && pl.t >= FLIP_TIME * FLIP_DROP) {
  pl.droped = true;
  if (player.on === pl) { player.on = null; player.vy = Math.max(player.vy, 160); }
}
if (pl.t >= FLIP_TIME) pl.st = 2;          // 翻面完成，此后不可站
```
- 状态机：`st 0`（正常） → 踩上置 `st=1`（翻转中） → `t≥FLIP_TIME` 置 `st=2`（已翻面）。
- 视觉：绕中轴翻转用**板厚投影** `h2 = max(2, round(h * |cos(k·π)|))`（1→0→1），翻面后画深色背面 + 加强筋。
- **不可站判定**：`st===2` 的板在支撑复检里被排除；`deepestY()` **不排除**（见 5.6，因为它不移动位置）。

**② 弹簧竖直锁定 `spring`** —— 起跳后短暂屏蔽左右输入，保证轨迹竖直。
```js
const SPRING_LOCK = 0.22;                        // 秒
// 命中弹簧时：
p.vy = -620; p.on = null; p.vx = 0; p.lock = SPRING_LOCK;
// 每帧横移计算前：
if (p.lock > 0) { p.lock -= dt; dir = 0; }       // dir 是本次输入方向，归零 = 不响应
```
> 为什么不是"直接禁用按键"：横移 `dir` 在每帧开头由输入解算出来，锁定期把它强制归零即可，无需碰输入系统；锁定期结束后玩家**当前按住的方向键仍然有效**，手感连续。

**③ 传送带 `belt`** —— 站在上面被横向推动，方向随机（生成时定 `dir`）。**外观**：胶囊形浅银灰履带 + 两端金属滚轮（外圈/内盘/左上高光/轴心四层叠出），上下边缘的分段履带齿按 `dir` 滚动，线速度与 `BELT_SPEED` 对齐（`anim×20`）——**方向靠齿的滚动表达，不用箭头**。
```js
const BELT_SPEED = 120;                          // px/s，必须 < MOVE/2（MOVE=290）
// 站在传送带上时（每帧）：
p.x += pl.dir * BELT_SPEED * dt;                 // 含边界钳制；被推时记走路相位、约 0.4s 一声低音
```
> **关键约束 `BELT_SPEED < MOVE/2`**：玩家移动速度的一半是 145。若传送带速度 ≥ 145，玩家逆着按住方向键也走不动 → 传送带变成"单程陷阱"。取 120 保证玩家能稳定反向对抗，它只是一个"拖慢你、逼你多花时间"的干扰项，而不是死路。
> 通行设计上，这条约束直接回应了 v0.6 传送带回滚的根因（旧版 120px/s 与平台上升叠加后"逆向踩板不可能"）——本次**降低速度并保证可对抗**，矛盾消解（另见 TD-01）。

---

## 6. 性能设计

| 措施 | 效果 |
|---|---|
| 平台数组回收（移出视野上方 90px 即删除） | 数组长度稳定在 12–13 项，不随时间增长 |
| 离屏画布预生成纹理 + `createPattern` 平铺 | 底纹每帧只需 1 次 `fillRect` |
| 平台绘制前先做视野粗筛（`y < F_TOP-40` 直接 return） | 屏幕外平台零绘制开销 |
| 全部绘制取整（`Math.round`） | 避免抗锯齿，像素风更锐利 |
| 实测绘制调用量 | 4000 帧累计 `fill` 约 173 万次、`fillText` 约 17 万次，无掉帧（约 430 次 fill/帧） |

---

## 7. 兼容性与降级

| 场景 | 处理 |
|---|---|
| 触屏设备 | 媒体查询 `@media (pointer: coarse)` 显示左右虚拟按键；`touchstart/touchend/touchcancel` 与鼠标事件共用 `bindHold()` |
| 浏览器自动播放策略 | `AudioContext` 惰性创建、`try/catch` 包裹，失败静默 |
| 隐私模式 / localStorage 被禁 | 读写都包 `try/catch`，降级为会话内纪录 |
| 高 DPI 屏 | 画布固定 420×640 逻辑像素，靠 CSS 整体缩放；不做 DPR 适配（像素风反而需要放大时的颗粒感） |
| 移动端页面滚动干扰 | `touch-action: none` + 禁用文本选中 |

---

## 8. 已知技术债

| 编号 | 问题 | 影响 | 建议处理时机 |
|---|---|---|---|
| TD-01 | 传送带（belt）已于 v0.11 **重新引入**（速度 120 < `MOVE`/2，玩家可反向对抗，见 BUILD_LOG v0.11 与 §5.13） | 历史矛盾已消解 | 若与"平台整体上升"再次打架，优先调 `BELT_SPEED` 而非回滚 |
| TD-02 | 全部状态挂在全局 `let` 变量上，无模块隔离 | 变量多了以后容易误改 | 若继续扩展，应拆分到 `state` 对象中 |
| TD-03 | 单文件 797 行，靠注释分区维护 | AI 改动时需先读全文件 | 超过 1000 行时必须拆分为多文件 + ES Module |
| TD-04 | 无自动化测试，验证依赖自建桩环境脚本 | 每次改完要手动跑一次 | 把 `.workbuddy/verify-game.js` 作为回归脚本保留 |
| TD-05 | 难度曲线只调过一次，100 层以上偏难 | 高玩体验略陡 | **已由三档难度缓解**：新手走简单档、老手走地狱档。曲线本身仍可在 `DIFF` 表里继续调 `riseGain` 与陷阱倍率 |
