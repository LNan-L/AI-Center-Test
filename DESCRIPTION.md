# 仓库简介文案 · AI-Center-Test

GitHub 仓库的 **Description** 字段有 350 字符上限，**Topics** 最多 20 个标签。以下内容可直接复制使用。

---

## 一、仓库 Description（推荐）

```text
零依赖单文件前端演示：Canvas 2D 星空战舰，含视差星空、超空间跃迁、炮塔自动交战、HUD 与 WebAudio 合成音效。
```

*中文 55 字，远低于上限。英文备选：*

```text
Zero-dependency single-file front-end demo: a Canvas 2D star dreadnought with parallax starfield, hyperspace jumps, auto-turret combat, HUD and WebAudio-synthesized sound.
```

*极简版（适合列表页）：*

```text
单文件 Canvas 2D 星空战舰实时演示 ｜ 零依赖 · 无构建 · 无素材
```

---

## 二、仓库 Topics（推荐 9 个）

```text
canvas2d
javascript
html5
frontend-demo
game-demo
starfield
spaceship
webaudio
no-dependencies
```

可选补充：`single-file`、`procedural-generation`、`particle-system`、`parallax`

---

## 三、About 摘要（用于 README 顶部或发布说明）

**AI-Center-Test** 是 AI-Center 测试系列的第一个前端演示工程，用来验证一件事：

> 一个**单文件、零依赖、无外部素材**的 HTML 页面，能否撑起一个有叙事感的实时科幻场景？

结论是可以。这个页面里没有一张图片、没有一段音频、没有一个第三方库 —— 星空、星云、战舰、敌机、爆炸、护盾全部由 Canvas 2D 程序化绘制，四类音效（引擎低鸣 / 激光 / 爆炸 / 跃迁）全部由 WebAudio 实时合成。它在巡航、跃迁、索敌、开火、爆炸之间切换，并用一套 HUD 把状态反馈给你。

**亮点**

- 5 层视差星空 + 4 团预渲染星云；跃迁时星空拉成光条，并叠加环向隧道与全屏泛光
- 程序化矢量战舰：装甲分块、舰桥窗灯、引擎尾焰、5 座随动炮塔、双层护盾弧与六边形网格
- 完整交战循环：敌机成波次来袭、炮塔自动索敌、命中火花、爆炸粒子、冲击波
- 性能工程：FPS 自适应降级、`resize` 防抖、DPR 上限、热点绘制全部按"每帧分配数"核算
- 交互：鼠标指挥视差、点击/空格跃迁、`C` 巡航、`F` 齐射、`M` 音效、一键全屏

**一句话**：打开就炫，源码可读，改起来方便。

---

## 四、首个 Release 说明（v1.0.0）

```markdown
## 星空战舰动态演示 v1.0.0

首个公开版本：单文件 Canvas 2D 实时演示。

### 包含
- 5 层视差星空、4 团预渲染星云、近景尘埃速度层
- 程序化矢量战舰（舰体 / 舰桥 / 炮塔 / 护盾 / 引擎尾焰）
- 超空间跃迁：星空拖尾、隧道射线、全屏泛光、冲击波、镜头震动
- 交战系统：波次敌机、自动索敌、命中判定、爆炸粒子
- HUD：速度、护盾、舰体、威胁、FPS、星数、扫描线、暗角
- WebAudio 实时合成音效（引擎 / 激光 / 爆炸 / 跃迁）
- 性能：FPS 自适应降级、resize 防抖、DPR 上限

### 运行
双击 `star-warship.html` 即可，无需服务器与依赖。

### 已知限制
- 弹速超过约 3000 px/s 时逐帧圆检测会穿模
- 敌机为直线 + 正弦浮动，暂无 AI 与编队
```

---

## 五、社交平台一句话（可选）

> 一张 HTML 文件，一个零依赖的星空战舰：满屏视差星空、超空间跃迁、自动索敌开火，连音效都是浏览器现合成的。没有图片，没有音频文件，没有框架。
