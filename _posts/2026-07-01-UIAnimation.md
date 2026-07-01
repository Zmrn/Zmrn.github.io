---
layout: post
title: "简易 UI 过渡动画系统"
categories: unity
---

{: .note}
> 本文介绍一种基于组件和事件的轻量 UI 过渡动画方案，是[栈式导航的 Unity UI 管理方案](/_posts/2026-07-01-UIManagement.html)的配套附录。

UI 动画通常使用 Animator 或 Tween 插件来实现。其中 Animator 的实现非常重，会占用相当多的内存，初始化也会耗费一定时间，并且使用起来也有一定门槛。Tween 插件则只是使用起来有一定门槛。

实际上 UI 过渡动画往往非常简单，通常都是一些界面内组件的飞入、旋转、缩放、淡入淡出，难度比 PPT 制作还要低，大多数时候都可以通过一些较为简单的脚本配置来实现。这里介绍一种基于组件和事件的方案：动画逻辑封装在独立的 `UIAnimation` 组件里。在 UIManager 打开界面后，当前打开的页面上所有 `UIAnimation` 自动播放入场动画；UIManager 在关闭页面前广播一个退场事件，所有正在活跃的 `UIAnimation` 收到后播放反向动画，等待一个固定时间后再实际关闭相应界面。整个过程不需要页面脚本参与，动画都在一个固定的时间（通常是零点几秒）内播完，并且配置方式足够简单，可以节约大量人力。

# 基类：UIAnimation

`UIAnimation` 是所有 UI 动画组件的基类，继承自 `MonoBehaviour`，可以挂到任意 UI 节点上。

**核心字段：**

| 字段 | 说明 |
|---|---|
| `autoPlayWhenEnable` | 节点 `OnEnable` 时自动播放入场动画 |
| `needTime` | 动画时长（秒），通常使用默认值，不推荐修改 |
| `waitTime` | 开始前的延迟时间，用于错开同一页面内多个元素的动画，或让一个对象在不同时间播放不同动画 |
| `overrideCurve` / `easeCurve` | 是否覆盖默认曲线，默认为 EaseInOut |

**核心方法：**

- `Play(reverse = false)`：播放动画。`reverse = true` 时反向播放（退场）
- `Complete()`：立即跳到终态，用于跳过动画
- `GetReady()`：立即回到初态，为下次入场做准备

**子类需要覆写的方法：**

```csharp
protected virtual void AnimeUpdate(float v) { }  // v ∈ [0,1]，每帧调用
protected virtual void OnComplete() { }           // 动画播完时（到达终态）
protected virtual void OnReady() { }              // 重置到初态时
```

**与 UIManager 联动：**

由 `UIManager` 发送退场事件来进行通知，`UIAnimation` 监听退场事件播放退场动画，不需业务逻辑或 `UIManager` 持有动画组件的引用。入场时则是通过 `UIAnimation` 自己的 `OnEnable` 触发。

**全局时间因子：**

```csharp
public static float PlayTimeMulply = 0.5f;
public static float StandardWaitTime = 0.5f * PlayTimeMulply; // UIManager 等待退场动画的时长
```

所有动画的实际时长都乘以这个因子。调整 `PlayTimeMulply` 即可全局加快或放慢所有 UI 动画，方便开发期间调试。

# 调用时序

每次执行 Forward / Back / Replace 前，UIManager 都会先检查当前是否有活跃的动画组件（由 `UIAnimationManager` 统一追踪），如果有则广播退场事件并等待 `StandardWaitTime`：

```
调用 ForwardAsync / BackAsync / ReplaceAsync
    ↓
UIAnimationManager.Count > 0 ?
    ↓ 是
广播退场事件 → 所有活跃 UIAnimation 播放退场动画
等待 StandardWaitTime（默认 0.25s）
    ↓
执行实际导航逻辑（加载新页面、切换栈等）
    ↓
新页面 OnEnable → autoPlayWhenEnable → 播放入场动画
```

# 内置实现

## UIAnimationAlpha（透明度淡入）

自动挂载 `CanvasGroup`。入场时从 `fromAlpha`（通常为 0）线性插值到 1，退场反向。

```csharp
[SerializeField] float fromAlpha = 0;

protected override void AnimeUpdate(float v) =>
    Group.alpha = Mathf.Lerp(fromAlpha, 1f, v);
```

## UIAnimationFlyIn（位移 + 缩放飞入）

入场时从偏移位置和初始缩放飞向目标位置，退场反向飞出。在 Editor 中会绘制 Gizmo 线段显示起点，方便调整。

```csharp
[SerializeField] Vector3 fromPos;   // 相对于目标位置的偏移，如 (0, -200, 0) 表示从下方飞入
[SerializeField] float fromScale = 1;

protected override void AnimeUpdate(float v)
{
    transform.localPosition = Vector3.Lerp(trueFromPos, initPos, v);
    transform.localScale    = Vector3.Lerp(trueFromScale, initScale, v);
}
```

# 使用方式

在 Prefab 中为需要动画的节点挂上对应的 `UIAnimation` 组件，在 Inspector 里配置时长、延迟和参数即可。多个子节点可以各自挂不同的动画组件并设置不同的延迟，实现错落的入场效果。

```
RoleInfoPage (Page)
├── Header          → UIAnimationFlyIn  waitTime=0.00  fromPos=(0, 80, 0)
├── AvatarPanel     → UIAnimationAlpha  waitTime=0.05
├── StatsPanel      → UIAnimationFlyIn  waitTime=0.10  fromPos=(60, 0, 0)
└── ActionButton    → UIAnimationAlpha  waitTime=0.15
```

同一个子节点可以挂载多个动画组件，实现组合效果。也可以通过调整waitTime设计动画的时序。