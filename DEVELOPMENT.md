# 开发文档 · 星空战舰动态演示

> 版本 1.0.0 ｜ 最后更新 2026-10-12 ｜ 适用文件 `star-warship.html`

本文档面向要**读懂、调优或二次开发**这个演示的工程师，说明整体架构、各模块职责与关键算法、渲染管线顺序、性能预算与降级策略，并给出可直接照做的扩展指南。

---

## 目录

1. [设计目标与非目标](#1-设计目标与非目标)
2. [运行环境与技术栈](#2-运行环境与技术栈)
3. [总体架构](#3-总体架构)
4. [模块详解](#4-模块详解)
5. [渲染管线](#5-渲染管线)
6. [核心算法](#6-核心算法)
7. [状态机与交互](#7-状态机与交互)
8. [性能预算与自动降级](#8-性能预算与自动降级)
9. [参数速查表](#9-参数速查表)
10. [扩展指南](#10-扩展指南)
11. [调试与排错](#11-调试与排错)
12. [编码约定](#12-编码约定)
13. [已知限制与路线图](#13-已知限制与路线图)

---

## 1. 设计目标与非目标

### 目标

| # | 目标 | 判定标准 |
| --- | --- | --- |
| G1 | **单文件交付** | 一个 `.html` 文件双击即可运行，无网络请求 |
| G2 | **零依赖** | 不引入任何第三方库、字体、图片、音频文件 |
| G3 | **实时 60fps** | 1080p 中端设备稳定 60fps；低端设备自动降级而非卡死 |
| G4 | **完整叙事感** | 至少包含：巡航、跃迁、索敌、开火、爆炸、HUD 反馈、音效 |
| G5 | **代码可读** | IIFE 内按编号分节，模块边界清晰，便于局部替换 |

### 非目标

- ❌ 不做游戏逻辑（无胜负、无成长、无存档）
- ❌ 不做资源管线（不加载任何外部素材）
- ❌ 不做移动端手势全适配（仅单指拖动视差与点击跃迁）
- ❌ 不追求物理正确（弹道与爆炸均为美术表达）

---

## 2. 运行环境与技术栈

| 项 | 说明 |
| --- | --- |
| 运行环境 | 现代浏览器（Chrome / Edge / Firefox / Safari 最近两个大版本） |
| 渲染后端 | Canvas 2D（`getContext("2d", { alpha: false })`） |
| 音频后端 | WebAudio API（Oscillator / AudioBufferSource / BiquadFilter） |
| 语言特性 | ES2020（箭头函数、模板字符串、可选链、解构、默认参数） |
| 构建工具 | **无**。直接编辑 `.html`，刷新浏览器即可 |
| 依赖管理 | **无** `package.json`、无 `node_modules` |

### 必要 API 依赖

- `CanvasRenderingContext2D`：`createLinearGradient` / `createRadialGradient` / `globalCompositeOperation` / `ellipse` / `setTransform`
- `HTMLCanvasElement`：离屏画布（星云预渲染）
- `requestAnimationFrame` / `performance.now`
- `window.devicePixelRatio`
- `AudioContext`（可选，静音时完全不初始化）
- `document.documentElement.requestFullscreen`（可选，全屏按钮）

---

## 3. 总体架构

整个演示包在一个 IIFE 中，内部按 **13 个编号分节** 组织。数据流是单向的：

```
                  ┌─────────────────────────────────────────┐
   用户输入 ──────▶│  §6 输入层：pointer / keydown / 按钮      │
                  └───────────────┬─────────────────────────┘
                                  │ 写入
                                  ▼
                  ┌─────────────────────────────────────────┐
                  │  §1 状态层 state / pointer / camera      │◀── 唯一可写全局
                  └───────────────┬─────────────────────────┘
                                  │ 读取 + 推进
                                  ▼
   §9 update(dt)  ←── 固定时间步逻辑：星空位移、状态机、索敌、弹道、粒子、冲击波
                                  │
                                  ▼
   §10 渲染层 ── drawBackground → drawStars → drawForegroundDust
                → drawWarpTunnel → drawShockwaves → drawEnemies
                → drawShip → drawBolts → drawParticles → drawLensFlare
                                  │
                                  ▼
   §11 HUD 同步（节流 100ms）  §12 主循环 rAF 驱动
```

### 关键设计决定

| 决定 | 理由 |
| --- | --- |
| **实体用扁平数组 + 反向遍历删除** | 粒子峰值可达 1400+，对象池会引入生命周期管理复杂度；反向 `splice` 语义简单且无额外分配 |
| **星云预渲染到离屏 canvas** | 每帧重绘 26 组径向渐变不可接受；预渲染后每帧仅一次 `drawImage` |
| **单一 `state` 对象承载跨模块状态** | 避免散落的模块级 `let`，便于调试时整表打印 |
| **镜头只通过 `camera.{x,y,roll}` 表达** | 舰体、炮口、星空视差都从同一组参数派生，保证视觉一致 |
| **音效全部合成，不加载音频文件** | 满足 G2；同时让音高与包络能跟随 `state.warp` 实时变化 |

---

## 4. 模块详解

### §0 基础工具

```js
const clamp = (v, a, b) => v < a ? a : v > b ? b : v;
const lerp  = (a, b, t) => a + (b - a) * t;
const rand  = (a, b) => a + Math.random() * (b - a);
const TAU   = Math.PI * 2;
```

- `lerp` 是全篇最常用的算子，配合 **帧率无关的指数平滑** 使用（见 §6.1）。
- 屏幕尺寸 `W/H`、中心 `CX/CY`、视口缩放 `S`、`shipScale` 均为模块级变量，由 `resize()` 统一写入。

### §1 全局状态

三个对象，职责严格分离：

| 对象 | 含义 | 谁写 |
| --- | --- | --- |
| `state` | 世界状态（时间、跃迁强度、护盾、战果、画质等级…） | `update()` 与动作函数 |
| `pointer` | 原始与平滑后的指针归一化坐标 `[0,1]` | 输入层写 `tx/ty`，`update()` 写 `x/y` |
| `camera` | 由指针派生出的镜头偏移与滚转 | 仅 `update()` |

`state` 字段一览：

| 字段 | 初值 | 说明 |
| --- | --- | --- |
| `t` | 0 | 累计时间（秒），驱动所有周期性动画 |
| `dt` | 1/60 | 本帧步长（已 clamp 到 ≤ 1/20） |
| `warp` | 0 | 跃迁强度缓动值 `0→1` |
| `warpTarget` | 0 | 跃迁目标（1 = 正在进入超空间） |
| `warpCooldown` | 0 | 跃迁冷却剩余秒数 |
| `cruise` | true | 自动巡航；关闭时星空与敌机速度整体降到 28% |
| `autoFire` | true | 是否允许炮塔自动开火 |
| `audio` | false | 音效开关（与 `audio.on` 同步） |
| `autoQuality` | 0 | 画质等级 `0`（高）→ `3`（低），由 FPS 自动调节 |
| `fpsEMA` | 60 | FPS 指数移动平均，用于降级判定 |
| `kills` / `shield` / `hull` / `threat` | — | 战斗与 HUD 数值 |
| `weaponHeat` / `weaponState` | 0 / `"AUTO"` | 武器热量与 HUD 显示的武器状态 |
| `shake` / `flash` | 0 | 镜头震动强度（px）/ 全屏泛光强度 |

### §2 星空 / 星云

- `makeStar(layer)`：按层号生成星星，`z = layer / (STAR_LAYERS - 1)` 作为深度；半径、基亮度、闪烁速度都与 `z` 正相关。
- `makeNebula(size, alpha, hue)`：在离屏 canvas 上叠 26 个随机径向渐变（色相抖动 ±26°），再撒 `size * 0.35` 颗受距离衰减的星尘，返回该 canvas。
- `buildNebulae()`：生成 4 团不同尺寸 / 色相 / 漂移速度的星云；漂移越快的尺寸越小，用于制造纵深。

### §3 粒子 / 弹丸 / 敌机

| 系统 | 数据结构 | 上限 | 说明 |
| --- | --- | --- | --- |
| `particles` | 扁平对象数组 | 1400（超出 `shift()` 丢最旧） | 通用火花 / 爆炸 / 尾迹 |
| `bolts` | 扁平对象数组 | 无硬上限，寿命 1.1s | 能量弹，含速度、长度、伤害 |
| `enemies` | 扁平对象数组 | 生成端限制 7 | 敌机，含血量、色相、受击闪白 |
| `shockwaves` | 扁平对象数组 | 随事件生成 | 环形冲击波，半径按 `lerp` 逼近 `max` |

`burst(x, y, n, hue, power)` 是统一的爆炸函数：粒子速度、寿命、尺寸都按 `power` 缩放，因此同一函数同时服务「命中火花（power 0.45）」与「敌机爆炸（power ≈ size/22）」。

### §4 战舰几何

舰体定义在**局部坐标系**：舰首朝 `+x`，尾喷口在 `x ≈ -290`，半高最大 `86`。绘制时依次施加：

```
translate(CX + camera.x, CY + camera.y)   // 镜头位置（不含震动，震动在 ctx 外层）
rotate(camera.roll)                        // 镜头滚转
scale(shipScale, shipScale)                // 视口自适应缩放
translate(0, bob)                          // 悬浮上下浮动
```

`hullPath(g)` 只负责构建路径，填充 / 描边 / 裁剪 / 高光复用同一路径对象，这是保持舰体细节不"发散"的关键。

5 座炮塔定义在 `buildTurrets()`：

| 位置 | 数量 | 缩放 |
| --- | --- | --- |
| `(-110, ∓22)` | 2 | 1.0 |
| `(70, ∓30)` | 2 | 0.85 |
| `(195, 0)`（舰首） | 1 | 0.7 |

### §5 音效

`audio` 是带内部状态的单例，`init()` 懒加载（首次开启音效时才创建 `AudioContext`，规避自动播放策略报错）。

| 方法 | 合成方式 |
| --- | --- |
| 引擎低鸣 | 3 路失谐锯齿波（42 / 63.5 / 84.8 Hz）→ 增益 → 240Hz 低通 → 主输出，增益由开关键控 |
| `laser()` | 方波 `1400 → 180Hz` 指数扫频，经 bandpass(900Hz, Q=1.2)，18ms 包络衰减 |
| `boom(power)` | 2 秒白噪声缓冲 → lowpass `1200·power → 90Hz` 指数扫频，20ms 起振 / 700ms 衰减 |
| `warp()` | 循环白噪声 → bandpass(Q=6)，频率 `160 → 4200 → 300Hz` 三段扫频，1.5s 包络 |

> 所有包络使用 `exponentialRampToValueAtTime`，起始值必须 `> 0`（代码统一用 `0.0001`），这是 WebAudio 的硬性要求。

### §6 输入

- `pointermove` / `pointerdown` → `setPointer()`：归一化坐标，并在首次交互时隐藏提示条。
- `pointerdown` 检查 `e.target.closest("#dock")`，避免点击按钮时误触发跃迁。
- 键盘：`空格`（跃迁，`preventDefault` 防页面滚动）、`c`、`f`、`m`。

### §13 启动顺序

```js
resize();                              // 测量视口 → 建立星空 / 星云 / 尘埃
buildTurrets();                        // 建立炮塔
setTimeout(triggerWarp, 700);          // 开场跃迁，让第一帧就有高潮
setTimeout(() => spawnWave(3), 2600);  // 第一波敌机
setTimeout(() => hideHint(true), 11000); // 提示条兜底淡出
requestAnimationFrame(frame);          // 进入主循环
```

---

## 5. 渲染管线

每帧的绘制顺序**不可随意调换**，因为它决定了遮挡关系（Canvas 2D 没有深度缓冲）：

| 序 | 层 | 合成模式 | 备注 |
| --- | --- | --- | --- |
| 1 | `drawBackground` | source-over → lighter | 深空底色渐变 + 4 团星云（受镜头 0.35 系数视差） |
| 2 | `drawStars` | lighter | 视差系数随 `z` 从 0.2 到 1.0 插值 |
| 3 | `drawForegroundDust` | lighter | 近景尘埃，画质 ≥ 2.5 时跳过 |
| 4 | `drawWarpTunnel` | lighter | `warp < 0.02` 直接 return |
| 5 | `drawShockwaves` | lighter | 双层环 |
| 6 | `drawEnemies` | 混合 | 尾焰 lighter，机体 source-over |
| 7 | `drawShip` | 混合 | 引擎辉光（先）→ 舰体 → 细节 → 炮塔 → 护盾弧 |
| 8 | `drawBolts` | lighter | 拖尾线性渐变 + 弹头径向辉光 |
| 9 | `drawParticles` | lighter | `globalAlpha = a²`，让衰减更"干脆" |
| 10 | `drawLensFlare` | lighter | 全屏径向泛光，跃迁 / 震动时最强 |
| — | HUD | DOM | 不在 canvas 内，由 §11 节流更新 |

**震动实现**：在 `setTransform(DPR…)` 之后对 `ctx` 施加一次 `translate(rand(-sh, sh), …)`，于是所有 canvas 图层一起抖动，而 HUD（DOM）保持稳定 —— 这是刻意设计，稳定 HUD 能显著提升"仪表盘"的可读性。

---

## 6. 核心算法

### 6.1 帧率无关的指数平滑

所有缓动都使用同一模式：

```js
value = lerp(value, target, 1 - Math.pow(k, dt));
```

`k` 表示「经过 1 秒后剩余的偏差比例」。例如 `k = 0.001`（指针）意味着一秒内偏差收敛到 0.1%；`k = 0.02`（镜头位移）更慢更沉稳。相比 `lerp(a, b, 0.1)`，这种做法在 30fps 与 144fps 下的观感一致。

### 6.2 三层速度场的星空滚动

```js
const baseSpeed = 26 + state.warp * 2400;             // 巡航 26 px/s，跃迁峰值 2426 px/s
const sp = baseSpeed * (.25 + s.z * 1.9) * (cruise ? 1 : .28);
s.x -= sp * dt;
s.y += (sp * .06) * dt * (s.z - .4);                  // 近层轻微下坠，制造侧倾感
```

星星越界后从右侧回收，`y` 越界则回卷，因此任何分辨率下星空密度恒定。

超空间拖尾长度：

```js
const streak = s.trail * (240 + s.z * 900) * (cruise ? 1 : .3);
```

`s.trail` 是对 `state.warp` 的独立缓动值，因此"进入超空间"与"退出超空间"的拖尾会略微滞后于 `warp` 本身，视觉上更柔和。

### 6.3 炮塔索敌与瞄准

```js
// 索敌：取舰体前方最近的一架
for (const e of enemies) {
  if (e.x < CX) continue;               // 只打前方
  const d = e.x - CX;
  if (d < bestD) { bestD = d; target = e; }
}

// 瞄准：把目标角换算到炮塔本地角，并做 ±π 环绕归一化
const want = Math.atan2(ty - wy, tx - wx) - camera.roll;
let d = ((want - t.ang + Math.PI) % TAU + TAU) % TAU - Math.PI;
t.ang += d * Math.min(1, dt * 6);       // 转速有上限，炮塔"转得动"才有重量感
```

**炮口世界坐标**必须与 `drawShip` 的变换链严格一致，否则弹道会从错位的位置射出：

```
muzzle = R(camera.roll) · (turretLocal · shipScale) + (CX, CY)
```

注意**不要**把 `camera.x/y` 加进来 —— 震动由 `ctx.translate` 在更外层实现，若此处重复计入，炮口会与舰体脱节。

### 6.4 弹道命中判定

逐帧做一次**圆-圆**粗略检测：

```js
if (Math.hypot(e.x - b.x, e.y - b.y) < e.size * .8 + 6) { /* 命中 */ }
```

弹速 1500 px/s 时，单帧位移在 60fps 下约 25px，小于最小敌机半径（≈13px + 6px 容差），因此无需线段扫掠。**若把弹速提高到 3000 px/s 以上，请改为线段-圆扫掠检测**，否则会穿模。

### 6.5 自动开火节流

```js
if (Math.random() < dt * (2.4 + enemies.length * .35)) fireVolley();
```

概率随时间步长线性增长，等价于「平均每 `1/(2.4 + 0.35n)` 秒开火一次」，敌机越多射速越高，天然形成战斗节奏。`weaponHeat` 上限用于防止高频齐射刷屏。

---

## 7. 状态机与交互

### 跃迁状态机

```
        triggerWarp()                     warp 达到 1
IDLE ──────────────────▶ CHARGING ──────────────────▶ PEAK
  ▲        (冷却 3.2s)        warp += dt*1.35            │
  │                                                       │ warpTarget = 0
  └──────────────── DECAY ◀──────────────────────────────┘
                     warp -= dt*0.62
```

- 冷却期内 `triggerWarp()` 直接返回（键盘与点击均生效）。
- 触发瞬间同时施加：`flash = 0.9`、`shake = 12`、一枚全屏冲击波、90 颗粒子、音效。
- 跃迁期间敌机横向速度 ×3，自动开火在 `warp > 0.5` 时暂停 —— 让"赶路"与"战斗"两种节奏互不干扰。

### 交互优先级

| 事件 | 处理 |
| --- | --- |
| `pointerdown` 落在 `#dock` 内 | 交给按钮，画面不动 |
| `pointerdown` 落在画布上 | 更新指针 + 触发跃迁 |
| `keydown` 空格 | 跃迁（阻止默认滚动） |
| `keydown` c / f / m | 切换巡航 / 齐射 / 音效 |

---

## 8. 性能预算与自动降级

### 单帧预算（1080p / 60fps → 16.6ms）

| 阶段 | 预算 | 主要开销 |
| --- | --- | --- |
| 逻辑 `update()` | ≤ 2ms | 4200 颗星与 ≤1400 粒子的积分 |
| 背景 + 星云 | ≤ 3ms | 4 次 `drawImage`（最大 760²，缩放采样） |
| 星空 | ≤ 4ms | 4200 次 `arc`（约 380 颗亮星带十字光芒） |
| 舰体 + 敌机 | ≤ 3ms | 矢量路径 + 每帧少量渐变对象 |
| 弹丸 / 粒子 / 泛光 | ≤ 3ms | 加法混合填充 |

### 降级策略

```js
if (state.fpsEMA < 42) state.autoQuality = Math.min(3, state.autoQuality + .01);
else                   state.autoQuality = Math.max(0, state.autoQuality - .004);
```

| 等级 | 生效条件 | 行为 |
| --- | --- | --- |
| `autoQuality > 1.5` | FPS 持续偏低 | 跳过 `z < 0.34` 的最远星空层（约省 20% 星点） |
| `autoQuality ≥ 2.5` | 继续偏低 | 关闭近景尘埃层 |

**上升快（0.01/帧）、回落慢（0.004/帧）** 是刻意的滞回设计：一旦掉帧立刻降级保流畅，恢复时缓慢试探，避免画质在两档之间抖动。

### 已实施的优化

1. **星云预渲染**：把 26 组渐变固化成位图，每帧成本从「26 次 `createRadialGradient` + fill」降到 1 次 `drawImage`。
2. **尘埃不用渐变对象**：26 条尘埃使用纯色描边 + 加法混合模拟渐隐，避免每帧创建 26 个 `CanvasGradient`。
3. **跃迁拖尾不用渐变对象**：星空拖尾原为逐星线性渐变（峰值 4200 个/帧），改为带 `z` 权重的纯色描边 + 加法混合，视觉损失可忽略，开销降一个数量级。
4. **`resize` 防抖 220ms**：星云重建需数百毫秒，拖拽窗口 / 移动端转屏时不再反复触发。
5. **DPR 上限 2**：4K 屏上渲染面积控制在 1/4 以内，视觉几乎无损。
6. **DPR 感知的星星数量**：`clamp(W * H / 2100, 700, 4200)`，小屏不浪费、大屏不稀疏。

---

## 9. 参数速查表

| 想改变 | 位置 | 参数 |
| --- | --- | --- |
| 星星总量 | `resize()` | `W * H / 2100`（越小越密），上下限 `700 / 4200` |
| 星空层数 | 常量 | `STAR_LAYERS = 5`（同时影响 `makeStar` 的 `z` 分布） |
| 巡航速度 | `update()` | `baseSpeed` 中的 `26` |
| 跃迁速度峰值 | `update()` | `state.warp * 2400` |
| 跃迁上升 / 衰减速率 | `update()` | `1.35` / `0.62` |
| 跃迁冷却 | `triggerWarp()` | `3.2` 秒 |
| 舰体大小 | `resize()` | `shipScale = clamp(S * 1.05, .34, 1.5)` |
| 炮塔数量与位置 | `buildTurrets()` | `defs` 数组 |
| 炮塔转速 | `update()` | `dt * 6` |
| 弹速 | `fireVolley()` | `1500` px/s（**超过 3000 请改用扫掠检测**） |
| 自动开火频率 | `update()` | `dt * (2.4 + enemies.length * .35)` |
| 敌机上限 / 生成率 | `update()` | `enemies.length < 7`、`dt * .8` |
| 粒子上限 | `spawnParticle()` | `particles.length > 1400` |
| 星云数量与色相 | `buildNebulae()` | `cfg` 数组（`s` 尺寸 / `a` 强度 / `h` 色相 / `sp` 漂移） |
| 护盾弧半径 | `drawShip()` | `320 + k * 46` |
| 主音量 | `audio.init()` | `master.gain.value = .26` |
| 降级阈值 | `frame()` | `state.fpsEMA < 42` |

---

## 10. 扩展指南

### 10.1 加一种新的敌人

在 `makeEnemy()` 里按概率返回不同形态，并在 `drawEnemies()` 中按 `e.kind` 分支绘制：

```js
function makeEnemy() {
  const kind = Math.random() < .2 ? "frigate" : "fighter";
  // …
  return { kind, hp: kind === "frigate" ? 6 : 1,
           size: kind === "frigate" ? 78 : rand(16, 46), /* … */ };
}
```

注意同步调整：`e.size` 参与命中判定与爆炸规模；`e.hp > 1` 时才绘制血条。

### 10.2 加一种新武器（例如鱼雷）

1. 在 `fireVolley()` 旁新增 `torpedoes` 数组，或在 `bolts` 上用 `type` 字段区分。
2. 命中判定沿用 §6.4，但鱼雷应额外处理范围伤害：命中时对 `radius` 内所有敌机扣血。
3. 绘制时用更长的拖尾与独立色相（例如 `hue: 30`）区分视觉语言。

### 10.3 换掉音效后端

`audio` 暴露的接口只有 `setOn / laser / boom / warp`。若要接真实音频文件，只需在 `init()` 中用 `decodeAudioData` 预加载 buffer，并把上述方法改为 `playBuffer(name)`，其余调用点无需改动。

### 10.4 把演示嵌进别的页面

```html
<iframe src="star-warship.html" style="width:100%;height:100vh;border:0"
        allow="fullscreen"></iframe>
```

- 演示以 `window.innerWidth/Height` 为准，会自适应 iframe 尺寸；
- 键盘快捷键作用于 `iframe` 内部（需先点击画面获得焦点）；
- 若宿主页面已有滚动，建议保留 `空格` 的 `preventDefault`（已内置），否则空格会滚动宿主页面。

### 10.5 提升到万级粒子（WebGL 路线）

当前瓶颈在 `drawParticles()` 的逐粒子 `arc()` 调用。若要上万粒子：

- 用 `WebGL2` + instanced quad；或
- 折中：把粒子预渲染成几张 16×16 径向光斑贴图，改用 `drawImage`（通常有 3–5 倍提升）；或
- 用 `Path2D` 累积同色粒子、一次性 `fill`。

---

## 11. 调试与排错

| 现象 | 可能原因 | 处理 |
| --- | --- | --- |
| 画面全黑 | `#stage` 被遮挡 / canvas 尺寸为 0 | 检查 `.hud` 的 `z-index` 与 `resize()` 是否执行 |
| 弹道从错误位置射出 | 炮口坐标与绘制变换链不一致 | 对照 §6.3 公式，确认未把 `camera.x/y` 重复计入 |
| 音效无声 | 未由用户手势触发 / 自动播放策略 | 必须点击「音效」或按 `M`；`AudioContext` 处于 `suspended` 时需 `resume()` |
| 帧率低 | 分辨率过高 / 星数上限过大 | 调大 `W*H/2100` 的除数或降低星星上限；确认 `autoQuality` 生效 |
| 拖动窗口后卡顿 | 星云被反复重建 | 确认 `onResize` 防抖仍在（220ms） |
| 高 DPI 下模糊 | `setTransform` 未包含 DPR | 每帧开头调用 `ctx.setTransform(DPR, 0, 0, DPR, 0, 0)` |
| 修改后无变化 | 浏览器缓存 | 硬刷新（`Ctrl + F5`） |

**建议的调试手段**：HUD 左下角实时显示 FPS 与星星数量；需要更细粒度时，在 `frame()` 内用 `performance.now()` 分段打点。

---

## 12. 编码约定

1. **分节编号**：新增模块追加编号小节，保持「输入 → 状态 → 更新 → 渲染 → 输出」的阅读顺序。
2. **命名**：状态用名词（`state.warp`）；动作函数用动词（`triggerWarp` / `toggleCruise` / `fireVolley`）；绘制函数统一 `draw*`；构建函数统一 `make*` / `build*`。
3. **绘制函数不修改状态**：`draw*` 只读 `state`，任何推进都放在 `update(dt)` 内。
4. **`save()` / `restore()` 必须成对**：当前全文件共 34 处，保持平衡。
5. **数值就近写字面量**：本项目刻意不建魔法数字常量表，以便调参时"就地改、就地看"；若某参数开始跨模块复用，再提取为常量。
6. **中文注释**：注释与文档使用中文，代码标识符使用英文。

---

## 13. 已知限制与路线图

### 已知限制

- 弹道命中为逐帧圆检测，弹速超过约 3000 px/s 会穿模（见 §6.4）。
- 敌机只有直线 + 正弦浮动两种运动，无真正 AI。
- 粒子与敌机使用数组 `splice`，极端峰值下有轻微 GC 抖动。
- 触屏仅支持单指，未处理双指缩放 / 旋转。
- HUD 为 DOM 元素，极端低帧下 HUD 更新会略晚于画面（已节流 100ms，不影响可读性）。

### 路线图

- [ ] 跳跃星图与航点选择
- [ ] 敌机 AI 分层（护卫舰 / 母舰）与编队行为
- [ ] WebGL 后端替换，支持万级粒子与真实泛光
- [ ] 可视化参数面板（实时调试星空密度 / 色温 / 战斗强度）
- [ ] 录制 GIF / WebM 一键导出
- [ ] 单元测试：抽出纯函数（几何、命中判定、冷却状态机）并接入 Vitest

---

## 附：文件清单

| 文件 | 作用 |
| --- | --- |
| `star-warship.html` | 演示主体（HTML + CSS + JS 全内联） |
| `README.md` | 项目介绍与使用说明 |
| `DEVELOPMENT.md` | 本文档 |
| `DESCRIPTION.md` | 仓库简介文案 |
| `LICENSE` | MIT 许可证 |

---

*文档由 AI-Center 测试项目维护。*
