# Quin Widget 开发规格文档
## 快速决策 桌面小组件 · 动效与交互说明

---

## 一、组件尺寸

| 平台 | 尺寸 | 规格 |
|------|------|------|
| iOS | 329 × 155 pt | WidgetKit `WidgetFamily.systemMedium` |
| Android | 约 329 × 155 dp | Glance `SizeMode.Exact` 或 `SizeMode.Responsive` |

---

## 二、状态定义

组件共有三个状态：

| 状态 | 说明 |
|------|------|
| **初始态 INITIAL** | 默认状态，展示左侧标题文字 + 右侧扇形牌轮 |
| **加载态 LOADING** | 用户点击后、API 返回前的等待状态 |
| **完成态 DONE** | API 返回结果，展示牌面 + 答案 + 按钮 |

---

## 三、交互逻辑

### 初始态
- **可点击区域**：整个组件
- **点击后**：执行「揭晓动画序列」（见第四节）

### 加载态
- **可点击区域**：无（禁用点击，防止重复触发）

### 完成态
- **可点击区域 ①**：左侧区域（答案文字 + 「展开来看」按钮）→ deeplink 跳转 App 详情页
- **可点击区域 ②**：右上角 ↺ 重置按钮（26×26 pt/dp）→ 执行「重置动画序列」（见第五节）
- iOS 实现：用 `Link(destination:)` 处理 deeplink，用 `AppIntent` 处理 ↺（需 iOS 17+ Interactive Widgets）
- Android 实现：两个独立 `Button`，各自绑定 `GlanceModifier.clickable`；deeplink 用 `actionStartActivity`

---

## 四、iOS · 揭晓动画序列（用户点击初始态）

> 总时长约 5.8 秒

| 时间点 (ms) | 元素 | 动画内容 | 时长 / 曲线 | SwiftUI 写法 |
|------------|------|---------|------------|-------------|
| 0 | 左侧文字 | translateX 0 → −36pt，opacity 1 → 0 | 320ms / easeIn | `.transition(.move(edge:.leading).combined(with:.opacity))` |
| 100 | 牌轮 | opacity 1 → 0 | 420ms / linear | `.transition(.opacity)` |
| 220 | 牌（牌背）| 出现在牌轮位置（x:212, y:22），translateY 0 → −14 → +3 → 0，scale 0.92 → 1.04 → 1，opacity 0 → 1（上浮后落定） | 680ms / spring(response:0.55, dampingFraction:0.7) | `.transition(.scale(0.92).combined(with:.opacity))` + `.animation(.spring(response:0.55, dampingFraction:0.7))` |
| 920 | 牌 | 向左滑动：x 212 → 134pt | 780ms / spring(response:0.6, dampingFraction:0.85) | `withAnimation(.spring(response:0.6, dampingFraction:0.85))` |
| 1060 | 加载遮罩 | opacity 0 → 1（覆盖在牌面上） | 220ms / easeIn | `.transition(.opacity)` |
| 1880 | 加载遮罩 | opacity 1 → 0 | 220ms / easeOut | `.transition(.opacity)` |
| 2060 | 牌 · 翻牌第一阶段 | scaleX 1 → 0（压缩）+ 高光扫过 | 220ms / easeIn | `withAnimation(.easeIn(duration:0.22)) { scaleX = 0 }` |
| 2280 | 牌 · 内容切换 | 牌背换成牌面图（此时 scaleX=0，不可见） | 0ms 瞬间 | ZStack 用 `if/else` 切换 @State bool |
| 2280 | 牌 · 翻牌第二阶段 | scaleX 0 → 1（展开）+ 高光扫过 | 280ms / cubicBezier(0, 0.85, 0.45, 1) | `withAnimation(.spring(response:0.5, dampingFraction:0.75)) { scaleX = 1 }` |
| 3140 | 牌 | 向右滑动：x 134 → 224pt | 780ms / spring(response:0.6, dampingFraction:0.85) | `withAnimation(.spring(response:0.6, dampingFraction:0.85))` |
| 3480 | 牌名标签 | opacity 0 → 1 | 380ms / easeOut | `.transition(.opacity)` |
| 3640 | 小标题「快速决策」 | opacity 0 → 1 | 300ms / easeOut | `.transition(.opacity)` |
| 3980 | 答案大字（Yes/No/Maybe） | opacity 0 → 1，慢慢浮现 | 1100ms / cubicBezier(0.4, 0, 0.2, 1) | `.animation(.easeInOut(duration:1.1))` |
| 5280 | 「展开来看」按钮 | opacity 0 → 1（答案完全显示后才出现） | 500ms / easeOut | `.transition(.opacity).animation(.easeOut(duration:0.5))` |
| 5280 | ↺ 重置按钮 | opacity 0 → 1 | 300ms / easeOut | `.transition(.opacity)` |

### 翻牌实现说明（重要）

WidgetKit 不支持 `rotation3DEffect` 和透视变换。替代方案：

1. 用 ZStack 叠放牌背视图和牌面视图
2. `withAnimation(.easeIn(duration:0.22))` 将 `scaleX` 压缩至 0
3. 在 `scaleX=0` 时通过 `@State` bool 切换显示牌背 → 牌面（此时不可见，无跳帧）
4. `withAnimation(.spring(response:0.5, dampingFraction:0.75))` 将 `scaleX` 展开回 1
5. 在牌面上叠加一个 `LinearGradient` 高光层，用 `offset` 动画横扫模拟翻面光泽感

---

## 五、iOS · 重置动画序列（用户点击 ↺）

| 时间点 (ms) | 元素 | 动画内容 | 时长 / 曲线 |
|------------|------|---------|------------|
| 0 | 按钮 + 答案 + 小标题 + ↺ | opacity → 0（同时） | 220ms / easeOut |
| 220 | 牌 · 逆向翻牌第一阶段 | scaleX 1 → 0（压缩牌面） | 220ms / easeIn |
| 440 | 牌 · 内容切换 | 牌面换回牌背 | 0ms 瞬间 |
| 440 | 牌 · 逆向翻牌第二阶段 | scaleX 0 → 1（展开牌背） | 280ms / spring |
| 720 | 牌周围金色光晕 | box-shadow opacity 0 → 1 → 0（金光一闪） | 进 320ms / 出 250ms |
| 1070 | 牌 | scale 1 → 1.09 → 0.06，rotate 0 → −10°，opacity 1 → 0（神秘消散） | 520ms / cubicBezier(0.4, 0, 0.8, 0.5) |
| 1310 | 牌轮（5张牌） | 各自从中心向外绽开：scale 0.1 → 1，旋转至最终角度，opacity 0 → 1；错开顺序：中间牌先（delay 0ms），再 ±1（80ms），再 ±2（160ms） | 每张 500ms / spring(response:0.4, dampingFraction:0.65) |
| 1610 | 左侧文字 | translateX −24 → 0，opacity 0 → 1 | 380ms / spring(response:0.45, dampingFraction:0.8) |

---

## 六、Android · 状态切换说明

> Android RemoteViews 动画能力有限，三态之间均为线性 crossfade 切换，无位移或变形动画。

### 架构推荐
- 使用 Jetpack Glance + `GlanceAppWidget`
- 用 `StateDefinition` 维护 `enum WidgetState { INITIAL, LOADING, DONE }`
- 数据流：`Widget 点击` → `ActionCallback.onAction()` → `WorkManager` 单次任务 → API 请求 → 结果写入 `DataStore<Preferences>` → `GlanceAppWidget.updateAll(context)` → Glance 读取 DataStore → 重组为 DONE 状态

### 状态切换动画

| 切换 | 方式 | 时长 | 说明 |
|------|------|------|------|
| INITIAL → LOADING | `RemoteViews` 通过 `setAlpha()` AnimatorSet 做 crossfade，或 Glance 重组 | 250ms / linear | 旧布局 alpha 1→0，新布局 alpha 0→1，中点交叉 |
| LOADING → DONE | 同上 crossfade | 300ms / linear | DONE 态所有内容同时出现，无分步揭晓 |
| DONE → INITIAL | Crossfade | 250ms / linear | 直接切换，无消散动画 |

> Android 插值器仅支持：Linear、AccelerateDecelerate、Accelerate、Decelerate、Bounce、Overshoot，不支持 cubic-bezier 和 spring 曲线。

### Loading 旋转圈
在 RemoteViews XML 中使用系统 `ProgressBar`（indeterminate 圆形）。颜色通过 `indeterminateTint` 属性设置，仅支持纯色，不支持渐变。备选方案：用 `RotateDrawable` 驱动自定义静态图标旋转。

### 牌面图片
将塔罗牌面预渲染为 `Bitmap`，或将全部 78 张 × 正逆位存为 vector drawable，通过 `RemoteViews.setImageViewBitmap()` 或 `setImageViewResource()` 设置。建议尺寸：62 × 98 dp。

### Android 删除的动画及原因

| 功能 | 删除原因 | Android 替代方案 |
|------|---------|----------------|
| 牌从牌轮中「弹出」的上浮落定动画 | RemoteViews 不支持 keyframe 动画 | 删除，牌面在 DONE 态 crossfade 时直接出现 |
| 牌横向滑动位移 | RemoteViews 不支持位置动画 | 删除，牌在 DONE 布局中静态定位 |
| 2D/3D 翻牌 | RemoteViews 不支持 transform 动画 | 删除，牌面随 crossfade 直接显示 |
| 答案分步揭晓 | RemoteViews 不支持逐视图动画排序 | 所有内容同时出现 |
| 重置时牌神秘消散 | RemoteViews 不支持 scale/rotate 动画 | 直接 crossfade 回 INITIAL |
| 牌轮从中心向外绽开 | RemoteViews 不支持 stagger | 牌轮以静态图直接出现 |

---

## 七、颜色 Token

| 结果 | Hex | 用途 |
|------|-----|------|
| Yes | `#3FA876` | 答案文字颜色、「展开来看」按钮背景 |
| No | `#CC5858` | 答案文字颜色、「展开来看」按钮背景 |
| Maybe | `#7E62B8` | 答案文字颜色、「展开来看」按钮背景 |
| 金色 | `#C09050` | 牌背纹理线条、光晕、loading 弧线颜色 |
| 组件背景 | `#FFFFFF` | Widget 背景（浅色模式） |
| 页面底色 | `#F2EEE7` | 组件外部背景 |
| 文字主色 | `#1C1A2E` | 标题、主文字 |
| 文字次色 | `#6B6680` | 副标题、说明文字 |
| 文字浅色 | `#A09CB0` | 牌名、标签等辅助文字 |
