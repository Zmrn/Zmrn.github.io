---
layout: post
title: "栈式导航的 Unity UI 管理方案"
categories: unity
---

{: .note}
> 一款游戏通常有大量各式各样的UI界面，玩家在其中反复进入、退出、跳转。一种常见而粗暴的UI管理方式是，每次跳转都靠手写逻辑来决定"打开哪个UI、关闭哪个UI、回到哪个UI"，随着界面数量增长，这种管理方式会让游戏渐渐变得难以维护。这种简单粗暴的维护方式实际上还有不少项目在用，维护这种界面的时候往往非常痛苦。更有甚者会使用 Canvas 的 Sorting Order 来处理页面间的遮挡关系，这种设计无疑是灾难性的。上面提到的两个问题在实际项目里并不少见，特别是一些历史包袱比较重的项目。
>
> 栈式导航是一种更成熟的解决方案：在页面之间跳转时，想要展示某个界面就 Push 这个界面来让它入栈，让栈顶的界面显示在最前，并根据需要隐藏后方的界面；从这个界面返回的时候就 Pop 这个界面，之前显示的界面就会自动变回栈顶显示出来。这样页面只需关心自己的生命周期，没有特殊需求的话，页面不需要知道是谁打开了自己、也不需要知道关闭后应该显示谁。本文只提供框架思路，不提供具体代码。

![栈式导航示意图](/_posts/2026-07-01-UIManagement/stack-navigation.png)

# UI 管理方式

## 两类 UI 单元

这套方案将所有 UI 划分为两类基本单元：

- **Page（页面）**：占据全屏的完整界面，是用户当前所在的主视图，例如主城界面、战斗准备界面、商店界面等。
- **Floating（浮窗）**：叠加在页面之上的弹窗或对话框，例如确认弹窗、角色详情弹窗、设置面板等。

两者设计时可继承自同一个基类，但在导航逻辑上有本质区别：Page 由导航栈驱动，Floating 由显示/隐藏事件驱动。

大多数界面都应该是 Page ，但是对于一些弹窗则应该使用 Floating 来实现。这些 Floating 通常出现在 Page 的前面，并且根据需求允许同时展示多个不同的 Floating 。

![RoleInfoPage 示例](/_posts/2026-07-01-UIManagement/role-info-page.png)

![AskFloating 示例](/_posts/2026-07-01-UIManagement/ask-floating.png)

## 分层渲染

UI 元素按层级隔离渲染，每层对应独立的 Canvas 或 RectTransform 节点。层级从低到高依次为：

```
PageLayer       ← 页面层（主要游戏界面）
FloatingLayer   ← 普通浮窗层
```

并且可以自定义新的 Layer ，根据需求添加。如 BaseLayer 用于实现常驻的底纹， DialogLayer 专门用于展示会话窗口，以保证它总是展示在 Floating 的前面。

## 页面容器

为了方便统一管理 Page 和它所属的浮窗，每个 Page 在实例化时都会被放入一个**页面容器（PageContainer）**中。由于 Unity UI 的渲染层级是越靠下的越晚渲染，页面越靠前，所以栈顶的页面其实更靠下。

```
UIRoot
├── PageLayer
│   ├── PageContainer                ← 栈底 Page 容器（不可见，被压在下方）
│   │   ├── PageRoot
│   │   │   └── MainPage             ← 栈底 Page
│   │   └── FloatingRoot             ← 属于这个 Page 的 Floating 容器（当前无 Floating）
│   └── PageContainer                ← 栈顶 Page 容器（当前可见）
│       ├── PageRoot
│       │   └── ShopPage             ← 当前展示的 Page
│       └── FloatingRoot
│           └── ItemDetailFloating   ← 属于这个 Page 的 Floating
└── FloatingLayer                    ← 全局浮窗，展示在所有页面之上
    └── ConfirmFloating              ← 一个确认窗口
```

PageContainer 本身走对象池复用——页面切换时，旧容器整体入池，新容器从池中取出初始化，避免频繁的 Instantiate / Destroy 开销。页面在被覆盖时会保留在层级中，只是被隐藏；在 Pop 后相应的 Page 和 Floating 会被销毁（频繁使用的 Page 或 Floating 可以选择回收到对象池，例如在 Page 和 Floating 的基类上添加一个 bool 值来决定这个对象是否会进入对象池）。

## 栈式导航

Page 的导航采用**栈（Stack）**来管理，核心操作如下：

| 操作 | 行为 |
|---|---|
| Forward<Page> | 将新页面压入导航栈，跳转过去，相当于 Push。是一个泛型函数，可以传入任何页面脚本 |
| Back | 弹出当前页面，返回上一页，相当于 Pop |
| Replace | 替换当前页面，不增加栈深度，相当于先 Back 再 Forward |
| BackTo | 回退到栈中某个指定页面，可能会触发多次 Back |

Page 的一些导航需要均设计为**异步**操作，主要是显示页面的时候的导航，这样可以在导航发生时加入资源加载、数据请求等异步任务。如果请求过重，玩家可能会感觉点击后过了一会儿界面才打开，所以这里的设计需要权衡。如果想给个别页面设置专门的 Loading 状态，可以考虑打开后再在相应页面里实现。

Floating 的导航不使用异步操作，因为确认弹窗之类的 Floating 对时机要求比较紧迫，我们更希望调用后弹窗立刻显示出来。所以需要反复展示的弹窗必须要支持对象池。Floating 第一次展示时会同步加载资源，如果有明显的卡顿的话，可以考虑对相应界面进行预加载。

```csharp
// 打开一个页面，进入商店
await UIManager.ForwardAsync<ShopPage>();

// 打开 Floating 时传入参数
UIManager.ShowFloating<ItemDetailFloating>(new ItemDetailFloating.Param { itemId = 101 });

// ItemDetailFloating 的实现
public class ItemDetailFloating : Floating
{
    public class Param
    {
        public int itemId;
    }

    public override void OnShow(object param)
    {
        var p = (Param)param;
        Refresh(p.itemId);
    }

    ... // 其它代码
}

// 关闭当前页面，返回上一页
UIManager.Back();

// 跨多级返回到指定页面
UIManager.BackTo<ShopPage>();

// 替换当前页面（不增加栈深度，常用于登录后跳主城之类的操作，相当于先 Back 在 Forward）
await UIManager.ReplaceAsync<MainPage>();
```

## Page 生命周期

Page 在导航过程中可能会经历以下回调，开发者可以按需覆写：

```
OnPrepareAsync(param)  ← 进入前异步准备，如果有需要请求的数据通常可以在这里请求，并且页面的资源加载也发生在这个时间（不过这应该是框架功能，不需要开发者专门在这个回调里实现）。并且有需要的话，打开这个界面的时候可以传入任意参数。
OnForwardTo(param)     ← 通过 Forward 进入这个界面时的回调，最常用的回调，页面的一些同步初始化可以放在这里，也可以拿到打开界面时传入的参数。
OnBackTo()             ← 通过 Back 或 BackTo 回到这个界面时的回调，一般界面都不太需要实现这个，但是部分界面会有需要。
OnNavigatedTo()        ← 进入这个界面时的回调，不论 Back 到这里还是 Forward 到这里都会触发。一般不太会用到。
OnLeave()              ← 离开这个界面时，不论是被其他页面覆盖还是自己返回。一般不太会用到。
```

除此之外，由于 Page 和 Floating 继承自 MonoBehaviour， 所以 MonoBehaviour 的那些生命周期根据需要也是可用的，如 Awake、 Update、 OnEnable 等。

## Floating 生命周期

Floating 通常会挂在 FloatingLayer 下，总是展示在所有 Page 的上方。浮窗由统一的管理器控制显示与隐藏。Floating 的生命周期回调如下：

```
OnShow(page, param)   ← 显示时（可挂载在特定 Page 上，并可以接收任意参数。 page 传 null 则放入通用的 FloatingLayer）
OnHide()              ← 隐藏时
```

但每个 Page 也可以持有自己的浮窗挂载节点，用于实现某些浮窗属于某个页面的特殊需求，如从某个 Page 里打开一个浮动的子页面，并且可以被新打开的界面覆盖。当 Page 被 Pop 时，该页面下挂载的所有浮窗会自动随之回收，不需要调用方手动清理。

## 全局事件

UI 系统可对外暴露若干全局事件，供需要感知 UI 状态的其他模块订阅，无需直接持有 UI 对象的引用。比较常见的也就是页面切换的事件：

```csharp
Action<Page>            TopPageChanged          // 顶层页面发生变化
Action<Floating, bool>  FloatingDisplayChanged  // 浮窗显示/隐藏状态变化
```

这些事件通常可用于驱动新手引导一类的系统，避免引导类的代码侵入业务逻辑。

---

# UI 资源管理方式

管理 UI 资源（包括 UI 预制件）需要结合项目的资源管理框架进行设计。这里只会讲述一些推荐的和 UI 资源相关的资源管理方式。

## Prefab 文件名与类名严格绑定

Page / Floating 的 Prefab 文件名**必须与脚本名和类名完全一致**，系统在运行时将类名作为预制件资源的地址去加载，省去了任何手写映射：

```
类名: LoginPage      → Prefab: LoginPage.prefab
类名: ShopFloating   → Prefab: ShopFloating.prefab
```

这个约定的好处是：新增一个页面只需建文件、写脚本，不需要在任何配置表里登记。当我们使用 ForwardAsync 打开界面时，通过传入一个相应界面脚本的泛型即可打开。不需要在代码中管理字符串，直接用类名就可以找到预制件。

```csharp
// 打开一个页面，进入商店
await UIManager.ForwardAsync<ShopPage>();
```

这样做更方便后续维护，如果在 IDE 里修改了某个页面的脚本名，所有需要展示这个界面的地方都会被同步修改（只是别忘了改预制件的名字）。并且开发时可以直接从 Forward 调用的地方跳转到相应的界面。

---

## 资源分类方式

UI 资源（脚本 + Prefab）以**业务模块**为单位组织，而不是把所有 Page 集中放在一个大文件夹里。每个业务模块独占一个顶层文件夹，内部用 `$Pages/` 和 `$Floatings/` 两个子目录存放 UI 相关内容，脚本和 Prefab 放在同一位置，不拆分。

许多项目会将页面放到同一个文件夹里，这样会对资源拆分（如果有需要的话）造成一定的阻碍；并且会导致同一个业务逻辑的 UI 和其他 UI 混在一起，降低开发者的开发效率。

对于确实无法或无需归属到某个业务模块的通用 Page 或 Floating（如全局 Loading 弹窗、确认弹窗等），则在顶层单独开 `$Pages/` 和 `$Floatings/` 两个文件夹统一收纳。

{: .note}
> 使用 `$` 符号标记文件夹是否会被放入资源包里。
> 
> 预制件引用的图片或其他资源无需放在这里，而是作为二级资源放在其他的通用资源库的文件夹里，由美术和策划负责人进行管理，以便打包时通过搜索引用将其合理分包。
>
> 相关内容可参考：[如何构建零冗余又内存友好的资源打包方案？](/_posts/2026-02-25-bundle.html)

```
MainAssets/
│
├── $Equipment/                         ← 装备业务模块
│   ├── $Pages/
│   │   ├── EquipmentPage.cs
│   │   ├── EquipmentPage.prefab
│   │   ├── EquipmentDetailPage.cs
│   │   ├── EquipmentDetailPage.prefab
│   │   ├── EquipmentSynthesisPage.cs
│   │   └── EquipmentSynthesisPage.prefab
│   └── $Floatings/
│       ├── EquipmentInfoFloating.cs
│       └── EquipmentInfoFloating.prefab
│
├── $Shop/                              ← 商店业务模块
│   ├── $Pages/
│   │   ├── ShopPage.cs
│   │   └── ShopPage.prefab
│   └── $Floatings/
│       ├── PurchaseConfirmFloating.cs
│       └── PurchaseConfirmFloating.prefab
│
├── $Pages/                             ← 通用 / 无法或无需归类的 Page
│   ├── LoginPage.cs
│   ├── LoginPage.prefab
│   ├── MainPage.cs
│   └── MainPage.prefab
│
└── $Floatings/                         ← 通用 / 无法或无需归类的 Floating
    ├── AskConfirmFloating.cs
    ├── AskConfirmFloating.prefab
    ├── LoadingFloating.cs
    ├── LoadingFloating.prefab
    ├── ItemDetailFloating.cs
    └── ItemDetailFloating.prefab
```

## UI 资源加载链路

以打开一个新页面为例，从导航请求到页面显示的完整链路如下：

```
UI 导航请求（Forward）
    ↓
向资源管理框架请求异步加载对应 Prefab
    ↓
从对象池取出（或新建）PageContainer
    ↓
在 PageContainer.pageRoot 下实例化页面 Prefab
    ↓
依次执行生命周期（OnPrepareAsync → OnNavigatedTo）
    ↓
将 PageContainer 插入对应的 UILayer
```

## 对象池与资源释放

- PageContainer 整体走对象池，页面切换时回收而非销毁。
- Page / Floating Prefab 在 Pop/Hide 时可配置是否放入对象池以便下次复用（适合频繁开关的浮窗）。
- 资源句柄（Handle）由 UI 系统内部统一缓存在字典里，页面关闭时如果无需放入对象池的话，自动销毁预制件并 Release 相关句柄，调用方无需手动释放。

---

# 附录：UI 动画系统

参见：[简易 UI 过渡动画系统](/_posts/2026-07-01-UIAnimation.html)
