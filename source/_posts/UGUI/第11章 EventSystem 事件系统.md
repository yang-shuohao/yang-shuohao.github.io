---
title: 第11章 EventSystem 事件系统
categories:
  - Unity
  - UGUI
tags:
  - UGUI
  - Unity
cover: /images/UGUI/UGUI.png
abbrlink: 7fad15d
date: 2026-08-26 00:00:11
---
# 第11章 EventSystem 事件系统

Unity 的 UGUI 并不仅仅是一套用于绘制界面的系统，它同时也是一套完整的交互系统。按钮点击、拖拽、滑动、输入框输入、ScrollRect 滚动、Toggle 勾选、Slider 拖动，这些看似简单的交互行为，背后实际上都依赖于 EventSystem 事件系统完成输入采集、射线检测、事件分发以及回调执行。

很多开发者在使用 UGUI 时，只停留在“给 Button 添加 OnClick 回调”的层面，很少真正理解 EventSystem 内部到底发生了什么。但当项目规模扩大后，输入冲突、UI 穿透、多摄像机点击异常、拖拽失效、射线阻塞错误、移动端触摸兼容问题等，都会逐渐暴露出来。而这些问题，本质上都与 EventSystem 的运行机制密切相关。

UGUI 的事件系统并不是一个单独存在的模块，它实际上连接了多个核心系统。

首先是输入系统，鼠标、键盘、触摸屏、手柄等输入设备的数据，都会先进入 InputModule。

随后是射线系统，GraphicRaycaster、PhysicsRaycaster、Physics2DRaycaster 等 Raycaster 会负责检测当前输入命中了哪些对象。

接着是事件分发系统。EventSystem 会根据当前命中的目标，调用 ExecuteEvents 执行对应接口，例如：

- IPointerClickHandler 

- IBeginDragHandler 

- IDragHandler 

- IEndDragHandler 

- ISubmitHandler 

最后才会真正触发 Button、Toggle、Slider 等控件中的逻辑代码。

从整体结构来看，UGUI 事件系统本质上是一套：

- 输入采集系统 

- UI 射线检测系统 

- 事件路由系统 

- 接口回调系统 

它与渲染系统共同组成了完整的 UGUI 框架。

本章将从源码层面深入分析 EventSystem 的内部结构，理解 Unity 如何完成一次 UI 点击事件的完整流程，包括：

- 输入如何进入 EventSystem 

- 射线检测如何找到目标 UI 

- GraphicRegistry 如何维护可交互 Graphic 

- GraphicRaycaster 如何进行排序与命中 

- RaycasterManager 如何统一管理 Raycaster 

- ExecuteEvents 如何派发事件 

- 新输入系统如何兼容 UGUI 

- UI 输入为何会影响帧同步 

通过本章内容，你将真正理解一次 Button 点击，到底经历了哪些底层步骤。

理解 EventSystem 后，很多曾经“玄学”的 UI 问题，都会变得清晰可控。

## 11\.1 输入与事件分发

在 UGUI 中，所有 UI 交互的起点都来自输入系统。

无论是鼠标点击、屏幕触摸、键盘导航，还是手柄操作，本质上都需要先转换为 EventSystem 可以识别的事件数据，然后再经过射线检测与事件分发，最终触发对应 UI 组件的回调逻辑。

很多开发者认为 Button 的点击是 Button 自己完成的，但实际上 Button 本身并不会主动检测鼠标。真正负责输入检测与事件派发的，是 EventSystem 与 InputModule。

因此，要理解 UGUI 的交互机制，首先必须理解：

- 输入是如何进入 EventSystem 的 

- EventSystem 如何更新输入状态 

- 输入数据如何转换为 PointerEventData 

- UI 事件如何被派发给目标对象 

### 11\.1\.1 EventSystem 的核心职责

EventSystem 是整个 UGUI 事件系统的核心调度器。

它主要负责以下几件事情：

1. 管理当前激活的 InputModule 

2. 更新输入状态 

3. 维护当前选中的 UI 对象 

4. 执行 Raycast 检测 

5. 分发 UI 事件 

6. 管理焦点与导航状态 

在场景中创建 UI 时，Unity 会自动生成一个 EventSystem 对象。

其默认结构如下：

```Plain Text
EventSystem
 ├── EventSystem
 └── StandaloneInputModule
```

其中：

- EventSystem 负责事件系统调度 

- StandaloneInputModule 负责传统输入系统输入采集 

在新输入系统中，则使用 InputSystemUIInputModule 替代 StandaloneInputModule。

EventSystem 本身并不负责读取鼠标与键盘，它只负责：

- 调用 InputModule 

- 接收输入结果 

- 分发事件 

因此，真正的输入入口，其实是 InputModule。

### 11\.1\.2 EventSystem 更新流程

EventSystem 每帧都会执行 Update。

源码位于UnityEngine\.EventSystems\.EventSystem

核心逻辑如下：

```C#
protected virtual void Update()
{
    if (current != this)
        return;

    TickModules();

    bool changedModule = false;

    for (int i = 0; i < m_SystemInputModules.Count; i++)
    {
        var module = m_SystemInputModules[i];

        if (module.IsModuleSupported() && module.ShouldActivateModule())
        {
            if (m_CurrentInputModule != module)
            {
                ChangeEventModule(module);
                changedModule = true;
            }
            break;
        }
    }

    if (!changedModule && m_CurrentInputModule != null)
        m_CurrentInputModule.Process();
}
```

整个流程可以概括为：

```Plain Text
EventSystem.Update
        │
        ├── 更新所有 InputModule
        │
        ├── 选择当前激活模块
        │
        └── 执行当前模块 Process
```

这里最关键的是：

```C#
m_CurrentInputModule.Process();
```

真正的输入处理与事件派发，都是在这里开始的。

### 11\.1\.3 BaseInputModule

所有输入模块都继承自 BaseInputModule。

继承结构如下：

```Plain Text
BaseInputModule
 ├── PointerInputModule
 │    ├── StandaloneInputModule
 │    ├── TouchInputModule
 │    └── InputSystemUIInputModule
```

BaseInputModule 定义了输入模块的基础能力，包括：

- 输入更新 

- Raycast 查询 

- 事件数据创建 

- 焦点管理 

- 事件派发辅助函数 

其核心成员：

```C#
protected EventSystem eventSystem;
```

表示当前所属 EventSystem。

最核心的虚函数：

```C#
public virtual void Process()
{
}
```

所有输入模块都会重写它。

因此，不同输入系统的本质区别，其实就是 Process 的实现不同。

### 11\.1\.4 StandaloneInputModule 输入流程

StandaloneInputModule 是旧输入系统默认使用的输入模块。

它负责：

- 鼠标输入 

- 键盘输入 

- 手柄导航输入 

其核心入口：

```C#
public override void Process()
```

源码结构大致如下：

```C#
public override void Process()
{
    bool usedEvent = SendUpdateEventToSelectedObject();

    if (!ProcessTouchEvents() && input.mousePresent)
        ProcessMouseEvent();

    if (eventSystem.sendNavigationEvents)
    {
        if (!usedEvent)
            usedEvent |= SendMoveEventToSelectedObject();

        if (!usedEvent)
            SendSubmitEventToSelectedObject();
    }
}
```

这里实际上分成了三部分。

第一部分是更新当前选中对象：

```C#
SendUpdateEventToSelectedObject();
```

例如 InputField 的输入更新。

第二部分是处理鼠标与触摸：

```C#
ProcessMouseEvent();
ProcessTouchEvents();
```

第三部分是处理导航事件：

```C#
SendMoveEventToSelectedObject();
SendSubmitEventToSelectedObject();
```

例如：

- 键盘方向键 

- 手柄导航 

- 回车确认 

### 11\.1\.5 PointerEventData

UGUI 中所有点击、拖拽、移动事件，最终都会转换为 PointerEventData。

它本质上是一次“指针输入状态”。

PointerEventData 内部包含：

```C#
public Vector2 position;
public Vector2 delta;
public GameObject pointerEnter;
public GameObject pointerPress;
public GameObject lastPress;
public RaycastResult pointerCurrentRaycast;
```

其中：

- position 表示当前鼠标位置 

- delta 表示移动增量 

- pointerEnter 表示当前悬停对象 

- pointerPress 表示当前按下对象 

- pointerCurrentRaycast 表示当前射线命中结果 

可以理解为 PointerEventData 是一次 UI 输入行为的完整上下文。

后续所有事件分发，都会依赖它。

### 11\.1\.6 UI 事件分发流程

当鼠标点击时，UGUI 会经历完整的事件分发流程。

整体流程如下：

```Plain Text
鼠标输入
    │
    ▼
StandaloneInputModule.ProcessMouseEvent
    │
    ▼
RaycastAll
    │
    ▼
找到命中对象
    │
    ▼
ExecuteEvents.Execute
    │
    ▼
调用接口函数
```

最终会调用：

```C#
IPointerClickHandler.OnPointerClick
```

例如 Button：

```C#
public class Button :
    Selectable,
    IPointerClickHandler
{
    public virtual void OnPointerClick(PointerEventData eventData)
    {
        Press();
    }
}
```

因此，Button 并不会主动检测点击，真正检测点击的是 EventSystem，Button 只是实现了事件接口。

### 11\.1\.7 ExecuteEvents 事件执行器

UGUI 中所有事件派发，都依赖 ExecuteEvents。

它本质上是一个静态事件执行器。

例如：

```C#
ExecuteEvents.Execute(
    target,
    eventData,
    ExecuteEvents.pointerClickHandler
);
```

这里会调用：

```C#
IPointerClickHandler.OnPointerClick
```

ExecuteEvents 内部维护了大量事件函数：

```C#
pointerEnterHandler
pointerExitHandler
pointerDownHandler
pointerUpHandler
pointerClickHandler
beginDragHandler
dragHandler
endDragHandler
```

因此，UGUI 的事件系统，本质上其实是接口回调系统。

只要对象实现对应接口，就可以接收到事件。

### 11\.1\.8 事件冒泡机制

UGUI 的事件并不是简单只发送给当前对象。

它还支持层级向上传递。

例如：

```Plain Text
Button
 └── Text
```

点击 Text 时：

- Text 没有点击接口 

- 事件会向父节点继续查找 

- 最终 Button 接收到点击 

源码中使用：

```C#
ExecuteEvents.ExecuteHierarchy
```

实现父级遍历。

核心逻辑：

```Plain Text
当前节点
   │
   ▼
父节点
   │
   ▼
继续向上
```

直到找到可处理事件的对象。

因此，很多 UI 即使点到子节点，父节点仍然能响应点击。

### 11\.1\.9 当前选中对象

EventSystem 会维护 currentSelectedGameObject 表示当前焦点对象。

例如：

- 当前激活 InputField 

- 当前键盘导航对象 

- 当前手柄选中对象 

切换选中对象时：

```C#
SetSelectedGameObject()
```

会自动触发：

```C#
OnSelect
OnDeselect
```

这也是 InputField 获取焦点时能够开始输入的原因。

### 11\.1\.10 输入系统与 UI 解耦

UGUI 的设计中，一个非常重要的思想是输入系统与 UI 解耦。

Button 不关心鼠标来自哪里，它只关心是否收到了点击事件。

因此：

- 可以替换输入系统 

- 可以接入新输入系统 

- 可以支持虚拟输入 

- 可以支持网络同步输入 

- 可以支持录制回放 

整个 UI 系统依赖的只是标准化事件数据。

这也是 UGUI 事件系统扩展性很强的根本原因。

### 11\.1\.11 小结

本节我们初步分析了 UGUI 输入系统与事件分发机制。

可以看到，UGUI 的交互并不是由 Button、Toggle 等控件主动完成的，而是由 EventSystem 统一调度。

整个输入流程本质上可以概括为：

```Plain Text
输入设备
    │
    ▼
InputModule
    │
    ▼
PointerEventData
    │
    ▼
Raycast
    │
    ▼
ExecuteEvents
    │
    ▼
UI 接口回调
```

理解这一流程后，后续分析 Raycast、GraphicRaycaster、GraphicRegistry 等核心系统时，就能够真正看懂：

Unity 是如何从鼠标位置，最终找到目标 UI 并触发点击事件的。

## 11\.2 Raycast 机制

在 UGUI 事件系统中，Raycast 是整个输入检测流程中最核心的部分之一。

EventSystem 本身并不知道鼠标当前点击了哪个 UI，它只负责收集输入数据。而真正负责“寻找目标对象”的，则是 Raycast 系统。

简单来说 Raycast 的职责就是从当前输入位置，找到真正被点击的对象。

例如：

- 鼠标点击 Button 

- 手指触摸 Image 

- 拖拽 ScrollRect 

- Pointer 悬停 UI 

- UI 阻挡 3D 场景点击 

这些功能，本质上都依赖 Raycast。

很多开发者会把 UGUI 的 Raycast 与 Physics\.Raycast 混淆，但两者实际上完全不同。

UGUI 使用的是 Graphic Raycast，而不是物理射线。

它并不会真正发射一条物理空间中的射线，而是通过屏幕坐标与 RectTransform 区域计算，找到命中的 UI 元素。

因此，UGUI Raycast 更准确地说应该叫 UI 命中检测系统。

### 11\.2\.1 EventSystem 中的 Raycast 流程

在上一节中我们已经知道：

输入最终会进入：

```C#
StandaloneInputModule.Process()
```

随后进入：

```C#
ProcessMouseEvent()
```

接着会调用：

```C#
eventSystem.RaycastAll()
```

源码如下：

```C#
public void RaycastAll(
    PointerEventData eventData,
    List<RaycastResult> raycastResults)
{
    raycastResults.Clear();

    var modules = RaycasterManager.GetRaycasters();

    for (int i = 0; i < modules.Count; ++i)
    {
        var module = modules[i];

        if (module == null || !module.IsActive())
            continue;

        module.Raycast(eventData, raycastResults);
    }

    raycastResults.Sort(s_RaycastComparer);
}
```

整个流程如下：

```Plain Text
PointerEventData
        │
        ▼
EventSystem.RaycastAll
        │
        ▼
遍历所有 Raycaster
        │
        ▼
执行 Raycast
        │
        ▼
收集 RaycastResult
        │
        ▼
排序
        │
        ▼
返回最终命中结果
```

这里有几个非常重要的点：

第一，EventSystem 并不会自己进行射线检测，它只是统一调度所有 Raycaster。

第二，UGUI 支持多个 Raycaster 同时存在。

例如：

- GraphicRaycaster 

- PhysicsRaycaster 

- Physics2DRaycaster 

第三，所有命中结果最终都会统一排序。

因此，UI 与 3D 对象其实可以共同参与事件竞争。

### 11\.2\.2 BaseRaycaster

所有 Raycaster 都继承自 BaseRaycaster。

继承结构如下：

```Plain Text
BaseRaycaster
 ├── GraphicRaycaster
 ├── PhysicsRaycaster
 └── Physics2DRaycaster
```

它定义了统一接口：

```C#
public abstract void Raycast(
    PointerEventData eventData,
    List<RaycastResult> resultAppendList);
```

所有射线检测系统都必须实现它。

这样 EventSystem 就可以统一调度。

这也是 Unity 事件系统高度解耦的重要原因。

### 11\.2\.3 GraphicRaycaster

UGUI 中最重要的 Raycaster 是 GraphicRaycaster。

它负责 Canvas UI 的命中检测。

通常每个 Canvas 上都会自动挂载 GraphicRaycaster。

它的核心工作包括：

- 获取 Canvas 下所有 Graphic 

- 检测鼠标位置是否命中 RectTransform 

- 判断是否被遮挡 

- 按深度排序 

- 返回最终结果 

其本质上是 UI 图元检测器。

### 11\.2\.4 GraphicRegistry 与 Graphic 查询

GraphicRaycaster 并不会遍历整个场景，否则性能会非常差。

实际上，UGUI 内部维护了一套 GraphicRegistry。

它负责维护每个 Canvas 下所有可 Raycast 的 Graphic。

GraphicRaycaster 会直接从 Registry 获取列表：

```C#
GraphicRegistry.GetGraphicsForCanvas(canvas)
```

这样就避免了：

- 全场景遍历 

- GetComponentsInChildren 

- Transform 深度扫描 

因此 UGUI 的 Raycast 性能，本质上依赖于 Registry 缓存机制。

### 11\.2\.5 GraphicRaycaster 检测流程

GraphicRaycaster 的核心流程如下：

```Plain Text
获取 Canvas 下所有 Graphic
        │
        ▼
过滤不可点击对象
        │
        ▼
检测 RectTransform 是否命中
        │
        ▼
检测 Graphic.Raycast
        │
        ▼
按 depth 排序
        │
        ▼
生成 RaycastResult
```

其中 RectTransform 命中检测非常关键。

源码中使用：

```C#
RectTransformUtility.RectangleContainsScreenPoint
```

检测屏幕坐标是否位于 UI 矩形内部。

这意味着 UGUI 点击检测本质上是二维矩形检测，而不是三维物理碰撞。

### 11\.2\.6 Graphic\.Raycast

很多开发者认为只要鼠标进入 RectTransform 就一定会命中，实际上并不是。

GraphicRaycaster 在矩形检测后，还会调用：

```C#
Graphic.Raycast()
```

例如：

```C#
public virtual bool Raycast(
    Vector2 sp,
    Camera eventCamera)
{
    if (!isActiveAndEnabled)
        return false;

    return true;
}
```

Image、Text 等组件都可以重写它。

例如：

- MaskableGraphic 

- Text 

- Image 

都可能进行额外过滤。

典型场景包括：

- Alpha 命中检测 

- Mask 裁剪检测 

- 自定义点击区域 

因此，RectTransform 命中不代表最终一定可点击。

### 11\.2\.7 RaycastTarget

UGUI 中有一个非常重要的属性 Raycast Target，位于Graphic组件上。

例如：

- Image 

- Text 

- RawImage 

都包含它。

源码中：

```C#
[SerializeField]
private bool m_RaycastTarget = true;
```

GraphicRaycaster 会直接过滤：

```C#
if (!graphic.raycastTarget)
    continue;
```

因此关闭 Raycast Target 后，该 UI 将不参与事件检测。

这是优化 UI Raycast 性能的重要方式。

例如：纯装饰性 Text 建议关闭 Raycast Target，否则大量无意义 Text 会参与点击检测。

### 11\.2\.8 RaycastResult

所有命中结果都会保存为 RaycastResult。

其结构包括：

```C#
public GameObject gameObject;
public BaseRaycaster module;
public float distance;
public int depth;
public int sortingLayer;
public int sortingOrder;
```

其中：

- gameObject 表示命中对象 

- module 表示来源 Raycaster 

- depth 表示 Graphic 深度 

- sortingOrder 表示排序层级 

最后 EventSystem 会统一排序。

排序规则非常复杂。

它会综合：

- Camera Depth 

- Sorting Layer 

- Sorting Order 

- Graphic Depth 

- Distance 

确保视觉上最前面的对象优先响应事件。

### 11\.2\.9 UI 与 3D 点击竞争

很多开发者都遇到过点击 UI 时，场景物体也被点击。

其本质原因是 UI 与 Physics Raycast 同时命中。

因为 EventSystem 会同时执行：

```C#
GraphicRaycaster
PhysicsRaycaster
```

如果没有正确阻挡3D 场景仍然会收到点击。

常见解决方案：

```C#
EventSystem.current.IsPointerOverGameObject()
```

用于检测当前是否点击在 UI 上。

如果是 UI 则阻止场景交互。

### 11\.2\.10 Blocking Objects

GraphicRaycaster 支持 Blocking Objects。

用于 UI 阻挡 3D 射线。

其模式包括：

- None

- TwoD

- ThreeD

- All

当启用后 GraphicRaycaster 会先检测物理阻挡。

如果前方有物体，则 UI 不再响应。

这一机制常用于世界空间 UI。

例如：

- 角色头顶血条

- 世界空间按钮

- 3D UI 面板

### 11\.2\.11 Screen Space 与 World Space Raycast

不同 Canvas 模式下，Raycast 逻辑也不同。

#### Screen Space Overlay

最简单，不依赖 Camera，直接屏幕坐标检测。

#### Screen Space Camera

需要 Camera 参与：

```Plain Text
屏幕坐标
    │
    ▼
Camera
    │
    ▼
UI 空间
```

#### World Space

最复杂。

需要：

- Camera 射线 

- Plane 相交 

- 世界坐标转换 

因此，World Space UI 的 Raycast 成本通常更高。

### 11\.2\.12 Raycast 性能问题

UGUI Raycast 在复杂 UI 下可能成为性能瓶颈。

原因包括：

- Graphic 数量过多 

- 大量 RaycastTarget 

- 深层 Canvas 

- 多个 Raycaster 

- World Space UI 

尤其是在移动端，每帧大量 Raycast 会带来明显 CPU 开销。

优化核心包括：

1. 关闭无用 RaycastTarget 

2. 减少 Graphic 数量 

3. 减少嵌套 Canvas 

4. 避免大型 World Space UI 

5. 减少每帧 Pointer 检测 

其中关闭 Text 的 RaycastTarget 往往收益极大。

### 11\.2\.13 小结

本节我们深入分析了 UGUI 的 Raycast 机制。

可以看到，UGUI 的点击检测本质上并不是物理射线，而是一套基于 Graphic 与 RectTransform 的二维命中检测系统。

整个流程可以概括为：

```Plain Text
输入位置
    │
    ▼
EventSystem.RaycastAll
    │
    ▼
GraphicRaycaster
    │
    ▼
GraphicRegistry
    │
    ▼
RectTransform 检测
    │
    ▼
Graphic.Raycast
    │
    ▼
RaycastResult 排序
    │
    ▼
返回最终命中对象
```

理解 Raycast 后，很多 UI 问题都会变得非常容易分析，例如：

- UI 点击失效 

- UI 穿透 

- ScrollRect 无法拖动 

- Button 被遮挡 

- 世界空间 UI 无法点击 

- UI 点击性能过高

## 11\.3 UI 点击流程

在使用 UGUI 时，开发者最常接触的交互行为就是按钮点击。

例如：

```C#
button.onClick.AddListener(OnClick);
```

看起来似乎只是简单注册了一个回调函数，但实际上，一次 UI 点击的背后，EventSystem 内部会经历一整套复杂流程。

从鼠标按下，到最终触发 Button\.onClick，中间涉及：

- 输入采集 

- PointerEventData 更新 

- Raycast 检测 

- Pointer Enter 

- Pointer Down 

- Pointer Up 

- Click 判定 

- ExecuteEvents 分发 

- Button 状态切换 

- UnityEvent 回调 

理解整个点击流程，对于分析：

- Button 无法点击 

- 点击触发两次 

- 拖拽冲突 

- ScrollRect 吃掉点击 

- UI 穿透 

- 点击失效 

等问题非常重要。

本节将完整分析一次 Button 点击到底经历了什么。

### 11\.3\.1 UI 点击整体流程

一次标准 UI 点击流程如下：

```Plain Text
鼠标按下
    │
    ▼
InputModule.Process
    │
    ▼
RaycastAll
    │
    ▼
找到目标 Graphic
    │
    ▼
PointerEnter
    │
    ▼
PointerDown
    │
    ▼
记录 Press 对象
    │
    ▼
鼠标抬起
    │
    ▼
PointerUp
    │
    ▼
Click 判定
    │
    ▼
PointerClick
    │
    ▼
Button.onClick.Invoke
```

注意真正触发点击的，并不是 MouseDown，而是 MouseUp 后的 Click 判定。

因此，按钮点击实际上是按下加抬起的组合行为。

### 11\.3\.2 鼠标输入入口

点击流程的起点位于：

```C#
StandaloneInputModule.Process()
```

随后进入：

```C#
ProcessMouseEvent()
```

源码如下：

```C#
protected void ProcessMouseEvent()
{
    ProcessMouseEvent(0);
}
```

其中0表示鼠标左键，随后会获取MouseState。

内部包含：

- Left Button

- Right Button

- Middle Button

每个按钮状态都会进一步处理。

### 11\.3\.3 获取 PointerEventData

鼠标状态最终会转换为PointerEventData

核心代码：

```C#
GetMousePointerEventData()
```

其内部会更新：

```C#
pointerData.position
pointerData.delta
pointerData.scrollDelta
pointerData.button
```

例如：

```C#
pointerData.position = input.mousePosition;
```

同时还会执行：

```C#
eventSystem.RaycastAll()
```

获取当前鼠标命中的 UI。

最终结果保存到：

```C#
pointerData.pointerCurrentRaycast
```

这里非常关键。

因为后续所有点击事件，都依赖当前 Raycast 结果。

### 11\.3\.4 鼠标进入事件

在点击之前，首先会处理 Pointer Enter。

源码：

```C#
HandlePointerExitAndEnter()
```

流程如下：

```Plain Text
当前 Hover 对象
        │
        ▼
与上一帧 Hover 比较
        │
        ├── 不同
        │      ├── PointerExit
        │      └── PointerEnter
        │
        └── 相同
               └── 不处理
```

因此只有 Hover 目标变化时才会触发 Enter/Exit。

对应接口：

```C#
IPointerEnterHandler
IPointerExitHandler
```

例如 Button 的高亮状态切换本质上就是 Hover 事件。

### 11\.3\.5 鼠标按下流程

当检测到鼠标按下时：

```C#
PressedThisFrame()
```

会进入：

```C#
ProcessMousePress()
```

核心逻辑如下：

```C#
var newPressed = ExecuteEvents.ExecuteHierarchy(
    currentOverGo,
    pointerEvent,
    ExecuteEvents.pointerDownHandler);
```

这里会向父节点层级查找：

```C#
IPointerDownHandler
```

找到后：

执行：

```C#
OnPointerDown()
```

例如 Button：

```C#
public virtual void OnPointerDown(
    PointerEventData eventData)
{
    Press();
}
```

随后 EventSystem 会记录：

```C#
pointerEvent.pointerPress = newPressed;
```

表示当前按下对象。

这个对象后续会用于 Click 判定。

### 11\.3\.6 Click 的本质

很多开发者认为鼠标按下时就已经 Click，实际上并不是。

UGUI 中 Click 必须满足：

1. MouseDown 

2. MouseUp 

3. 按下与抬起是同一对象 

只有满足这三个条件才会触发：

```C#
IPointerClickHandler
```

因此拖拽过程中通常不会触发 Click。

### 11\.3\.7 鼠标抬起流程

鼠标抬起时会执行：

```C#
ReleaseMouse()
```

随后：

```C#
ExecuteEvents.Execute(
    pointerEvent.pointerPress,
    pointerEvent,
    ExecuteEvents.pointerUpHandler);
```

触发：

```C#
IPointerUpHandler
```

然后进行 Click 判定：

```C#
if (pointerEvent.pointerPress == pointerUpHandler)
{
    ExecuteEvents.Execute(
        pointerEvent.pointerPress,
        pointerEvent,
        ExecuteEvents.pointerClickHandler);
}
```

这里是整个点击系统最关键的部分。

只有按下对象等于抬起对象才会触发 Click。

### 11\.3\.8 Button\.OnClick 的触发

Button 实现了IPointerClickHandler

源码：

```C#
public virtual void OnPointerClick(
    PointerEventData eventData)
{
    if (eventData.button != PointerEventData.InputButton.Left)
        return;

    Press();
}
```

Press\(\)内部执行m\_OnClick\.Invoke\(\)。

这里的m\_OnClick实际上是UnityEvent。

因此 Inspector 中注册的按钮回调最终本质上只是UnityEvent 调用。

### 11\.3\.9 Button 状态切换

Button 不只是触发点击，它还会更新自身状态。

例如：

- Normal

- Highlighted

- Pressed

- Disabled

- Selected

状态切换来自Selectable。

Button 继承结构：

```Plain Text
UIBehaviour
 └── Selectable
      └── Button
```

Selectable 内部负责：

- 颜色变化

- Sprite 切换

- Animation 播放

例如：DoStateTransition\(\)会根据当前状态更新 Button 外观。

因此 Button 的高亮与按压效果，本质上并不是 Animator 自动完成的，而是 Selectable 主动控制。

### 11\.3\.10 拖拽为什么会取消点击

很多开发者发现 ScrollRect 中 Button 有时无法点击。

原因就在于拖拽会取消 Click。

当鼠标移动超过阈值pixelDragThreshold系统会认为当前进入 Drag 状态。

随后：

```C#
pointerEvent.dragging = true;
```

并触发：

- BeginDrag

- Drag

- EndDrag

此时 Click 将不再触发，因为当前行为已经被判定为拖拽。

### 11\.3\.11 点击穿透问题

UI 点击流程中还有一个常见问题：点击穿透。

例如：

```Plain Text
点击 Button
    │
    ├── Button 响应
    └── 场景物体也响应
```

本质原因是 UI 与 Physics 同时检测。

解决方式：

```C#
EventSystem.current.IsPointerOverGameObject()
```

例如：

```C#
if (EventSystem.current.IsPointerOverGameObject())
    return;
```

阻止场景点击。

### 11\.3\.12 双击检测

UGUI 内部还支持双击检测。

源码：

```C#
pointerEvent.clickTime
pointerEvent.clickCount
```

如果短时间内连续点击：

```C#
clickCount++
```

因此开发者可以直接判断：

```C#
eventData.clickCount == 2
```

实现双击逻辑。

### 11\.3\.13 点击流程中的接口调用顺序

一次完整点击，接口调用顺序如下：

```Plain Text
PointerEnter
    │
    ▼
PointerDown
    │
    ▼
InitializePotentialDrag
    │
    ▼
PointerUp
    │
    ▼
PointerClick
```

如果发生拖拽，则会变成：

```Plain Text
PointerDown
    │
    ▼
BeginDrag
    │
    ▼
Drag
    │
    ▼
EndDrag
```

Click 通常不会触发。

### 11\.3\.14 ExecuteHierarchy 的作用

点击事件通常使用：

```C#
ExecuteEvents.ExecuteHierarchy()
```

而不是：

```C#
Execute()
```

区别在于：

#### Execute

只执行当前对象。

#### ExecuteHierarchy

会向父节点查找。

因此点击 Text 子节点时，Button 父节点仍然可以收到事件。

这也是 UGUI 事件冒泡的核心机制。

### 11\.3\.15 小结

本节我们完整分析了 UGUI 的点击流程。

可以看到一次 Button 点击，背后实际上经历了完整的输入、射线、事件分发与状态管理流程。

整个过程可以概括为：

```Plain Text
输入采集
    │
    ▼
PointerEventData 更新
    │
    ▼
RaycastAll
    │
    ▼
获取命中对象
    │
    ▼
PointerDown
    │
    ▼
记录 Press 对象
    │
    ▼
PointerUp
    │
    ▼
Click 判定
    │
    ▼
ExecuteEvents
    │
    ▼
Button.OnClick
```

理解这一流程后，很多复杂 UI 问题都能够快速定位，例如：

- Button 点击失效 

- ScrollRect 吃点击 

- UI 穿透 

- 双击异常 

- 拖拽冲突 

- Hover 状态错误

## 11\.4 GraphicRegistry 机制

在 UGUI 中，GraphicRaycaster 需要对当前 Canvas 下的所有 UI 进行射线检测。

但这里存在一个关键问题：**GraphicRaycaster 如何快速知道当前 Canvas 中有哪些 Graphic？**

最直接的方法，当然是每帧遍历整个场景查找 UI，但这种方式性能极差。

因此 UGUI 引入了 GraphicRegistry。

GraphicRegistry 的核心作用只有一个：维护 Canvas 与 Graphic 的映射关系。

这样 GraphicRaycaster 就能快速获得当前 Canvas 下所有可渲染、可检测的 UI 元素，而无需遍历整个场景。

### 11\.4\.1 GraphicRegistry 的定位

GraphicRegistry 本质上是 UGUI 的全局 Graphic 注册中心。

源码定义如下：

```C#
public class GraphicRegistry
```

它属于静态管理类。

内部核心数据结构如下：

```C#
private readonly Dictionary<Canvas, IndexedSet<Graphic>>
    m_Graphics = new Dictionary<Canvas, IndexedSet<Graphic>>();
```

这里维护了：

```Plain Text
Canvas
    ↓
Graphic 集合
```

也就是说每个 Canvas 都会对应一个 Graphic 列表。

这也是 GraphicRaycaster 能高效工作的基础。

### 11\.4\.2 为什么需要 GraphicRegistry

假设一个界面中存在：

- Button 

- Image 

- Text 

- RawImage 

- Slider 

- Toggle 

这些 UI 本质上都继承自 Graphic。

当鼠标点击时，GraphicRaycaster 需要检测当前鼠标是否命中这些 Graphic。

如果没有 Registry，GraphicRaycaster 可能需要：

```Plain Text
遍历整个场景
    ↓
查找所有 Graphic
    ↓
过滤当前 Canvas
    ↓
执行 Raycast
```

这显然代价巨大。

尤其是复杂 UI 场景：

- 数千个 UI 

- 多 Canvas 

- 动态界面 

- ScrollView 

每帧遍历会造成严重性能问题。

因此 UGUI 使用 GraphicRegistry 提前缓存所有 Graphic。

### 11\.4\.3 Graphic 的注册流程

当一个 Graphic 启用时，会自动注册到 GraphicRegistry。

入口位于：

```C#
Graphic.OnEnable()
```

源码如下：

```C#
protected override void OnEnable()
{
    base.OnEnable();

    CacheCanvas();

    GraphicRegistry.RegisterGraphicForCanvas(
        canvas,
        this
    );

    SetAllDirty();
}
```

这里最核心的是：

```C#
GraphicRegistry.RegisterGraphicForCanvas()
```

表示当前 Graphic 注册到所属 Canvas。

### 11\.4\.4 RegisterGraphicForCanvas

注册函数源码如下：

```C#
public static void RegisterGraphicForCanvas(
    Canvas c,
    Graphic graphic
)
```

内部逻辑如下：

```C#
IndexedSet<Graphic> graphics;

instance.m_Graphics.TryGetValue(c, out graphics);

if (graphics != null)
{
    graphics.AddUnique(graphic);
    return;
}

graphics = new IndexedSet<Graphic>();
graphics.Add(graphic);

instance.m_Graphics.Add(c, graphics);
```

整个流程如下：

```Plain Text
获取 Canvas 对应 Graphic 集合
        ↓
不存在则创建 IndexedSet
        ↓
加入当前 Graphic
```

这里最关键的数据结构是：

```C#
IndexedSet<Graphic>
```

### 11\.4\.5 IndexedSet 的作用

IndexedSet 是 UGUI 自定义容器。

它结合了：

- List 

- HashSet 

两种结构特点。

源码定义如下：

```C#
public class IndexedSet<T>
```

内部维护：

```C#
List<T> m_List;
Dictionary<T, int> m_Dictionary;
```

其优势包括：

相比普通 List，IndexedSet 能避免重复注册。

相比 HashSet，IndexedSet 保持顺序。

这非常适合 UI 系统。

因为 UI 不仅需要快速查找，还需要稳定遍历顺序。

### 11\.4\.6 Graphic 的注销流程

当 Graphic 被禁用时，会自动从 Registry 中移除。

入口位于：

```C#
Graphic.OnDisable()
```

源码如下：

```C#
protected override void OnDisable()
{
    GraphicRegistry.UnregisterGraphicForCanvas(
        canvas,
        this
    );

    CanvasUpdateRegistry.UnRegisterCanvasElementForRebuild(this);

    canvasRenderer.Clear();

    base.OnDisable();
}
```

核心逻辑：

```C#
GraphicRegistry.UnregisterGraphicForCanvas()
```

### 11\.4\.7 UnregisterGraphicForCanvas

注销函数源码如下：

```C#
public static void UnregisterGraphicForCanvas(
    Canvas c,
    Graphic graphic
)
```

内部逻辑如下：

```C#
IndexedSet<Graphic> graphics;

if (instance.m_Graphics.TryGetValue(c, out graphics))
{
    graphics.Remove(graphic);

    if (graphics.Count == 0)
        instance.m_Graphics.Remove(c);
}
```

整个流程如下：

```C#
获取 Canvas Graphic 集合
        ↓
移除当前 Graphic
        ↓
集合为空则移除 Canvas
```

这样可以保证 GraphicRegistry 永远只保存有效 UI。

### 11\.4\.8 GraphicRaycaster 如何使用 Registry

GraphicRaycaster 在射线检测时，首先会获取当前 Canvas 的 Graphic 列表。

源码如下：

```C#
var canvasGraphics = GraphicRegistry.GetGraphicsForCanvas(canvas);
```

其内部实现：

```C#
public static IList<Graphic> GetGraphicsForCanvas(Canvas canvas)
{
    IndexedSet<Graphic> graphics;
    if (instance.m_Graphics.TryGetValue(canvas, out graphics))
        return graphics;

    return s_EmptyList;
}
```

这里直接通过 Dictionary 查询。

时间复杂度：O\(1\)

这意味着即使场景存在大量 UI，Raycaster 仍能快速定位当前 Canvas 的 Graphic。

### 11\.4\.9 为什么按 Canvas 分类

很多人会疑惑为什么 Registry 要按 Canvas 分类？

原因在于 UGUI 的渲染本身就是以 Canvas 为单位。

不同 Canvas：

- 独立 Batch 

- 独立排序 

- 独立 Raycaster 

- 独立重建 

因此，射线检测也必须基于 Canvas。

例如：

- Canvas\_MainUI

- Canvas\_TopUI

- Canvas\_WorldUI

它们：

- 深度不同 

- 摄像机不同 

- 排序不同 

如果所有 Graphic 混在一起 Raycast 成本会极高。

### 11\.4\.10 GraphicRegistry 与 Canvas 重建

GraphicRegistry 不仅服务于 Raycast。

还间接参与：

- Canvas 重建 

- Graphic 更新 

- UI 排序 

- 渲染同步 

因为所有 Graphic 都必须先注册，后续系统才能知道当前 Canvas 中有哪些 UI。

例如：

```Plain Text
Graphic
    ↓
GraphicRegistry
    ↓
CanvasUpdateRegistry
    ↓
CanvasRenderer
```

整个 UGUI 更新链彼此关联。

### 11\.4\.11 GraphicRegistry 的生命周期特点

GraphicRegistry 是全局单例。

内部包括：

```C#
private static GraphicRegistry s_Instance;
```

获取方式：

```C#
public static GraphicRegistry instance
```

因此整个游戏只存在一个 GraphicRegistry，所有 Canvas 都共享它。

这也是为什么 GraphicRegistry 查询速度非常快。

因为它属于纯内存缓存结构。

### 11\.4\.12 动态 UI 与 Registry 更新

动态生成 UI 时：

```C#
Instantiate(prefab)
```

Graphic 会自动触发：

```C#
OnEnable()
```

从而自动注册。

销毁 UI 时：

```C#
Destroy(gameObject)
```

会触发：

```C#
OnDisable()
```

从而自动注销。

因此，开发者通常不需要手动管理 Registry。

UGUI 生命周期已经自动完成。

### 11\.4\.13 Registry 与射线性能

GraphicRegistry 最大价值在于降低 Raycast 搜索成本。

如果没有 Registry 每次点击需要遍历整个场景。

而有了 Registry 每次点击直接获取当前 Canvas Graphic 列表。

性能差距非常巨大。

尤其是在：

- MMO UI 

- 背包系统 

- ScrollView 

- 聊天窗口 

- 编辑器工具 

等复杂场景中。

### 11\.4\.14 GraphicRegistry 的局限性

GraphicRegistry 虽然高效，但它并不会帮你过滤：

- raycastTarget 

- cull 

- depth 

- Mask 

它只负责保存当前 Canvas Graphic 集合。

真正的过滤逻辑仍然位于：

```Plain Text
GraphicRaycaster.Raycast()
```

因此，Registry 只是数据源，并不是完整 Raycast 系统。

### 11\.4\.15 小结

本节深入分析了 GraphicRegistry 的工作机制。

GraphicRegistry 本质上是 Canvas 与 Graphic 的全局映射缓存。

其核心结构如下：

```Plain Text
Canvas
    ↓
IndexedSet<Graphic>
```

整个注册流程如下：

```Plain Text
Graphic.OnEnable
        ↓
RegisterGraphicForCanvas
        ↓
加入 Canvas Graphic 集合
```

注销流程如下：

```Plain Text
Graphic.OnDisable
        ↓
UnregisterGraphicForCanvas
        ↓
移除 Graphic
```

GraphicRegistry 的存在，使 GraphicRaycaster 可以快速获得当前 Canvas 下所有 UI 元素，从而避免遍历整个场景。

这也是 UGUI 射线系统能够保持高性能的重要原因之一。

## 11\.5 RaycasterManager

GraphicRegistry 负责维护 Canvas 与 Graphic 的映射关系，而 RaycasterManager 则负责管理所有 Raycaster。

在 UGUI 中，EventSystem 并不会直接持有 GraphicRaycaster、PhysicsRaycaster 等组件。

那么问题来了**EventSystem 如何知道当前场景中有哪些 Raycaster？**

答案就是 **RaycasterManager**。

RaycasterManager 的核心职责只有一个：统一管理当前场景中的所有 BaseRaycaster。

这样 EventSystem 才能在输入发生时，统一调用所有 Raycaster 进行检测。

### 11\.5\.1 RaycasterManager 的定位

RaycasterManager 本质上是 UGUI 的全局 Raycaster 注册中心。

源码定义如下：

```C#
internal static class RaycasterManager
```

它属于静态管理类。

内部核心数据如下：

```C#
private static readonly List<BaseRaycaster> s_Raycasters
    = new List<BaseRaycaster>();
```

这里保存了当前场景所有 Raycaster

包括：

- GraphicRaycaster 

- PhysicsRaycaster 

- Physics2DRaycaster 

- 自定义 Raycaster 

因此，整个 EventSystem 的 Raycast 流程，都依赖这个全局列表。

### 11\.5\.2 为什么需要 RaycasterManager

假设场景中存在：

```Plain Text
Canvas_MainUI
    └── GraphicRaycaster

Canvas_WorldUI
    └── GraphicRaycaster

MainCamera
    └── PhysicsRaycaster
```

当鼠标点击时：

EventSystem 必须知道：

1. 哪些 UI 可以检测 

2. 哪些 3D 物体可以检测 

3. 哪些 2D 物体可以检测 

如果没有统一管理，EventSystem 可能需要：

```Plain Text
遍历整个场景
    ↓
查找所有 Raycaster
    ↓
逐个执行检测
```

这种方式显然效率很低。

因此，UGUI 使用 RaycasterManager 维护全局 Raycaster 缓存。

### 11\.5\.3 Raycaster 注册流程

所有 Raycaster 启用时，都会自动注册到 RaycasterManager。

入口位于：

```C#
BaseRaycaster.OnEnable()
```

源码如下：

```C#
protected override void OnEnable()
{
    base.OnEnable();
    RaycasterManager.AddRaycaster(this);
}
```

这里最核心的是：

```C#
RaycasterManager.AddRaycaster(this);
```

表示当前 Raycaster 注册到全局管理器。

### 11\.5\.4 AddRaycaster

注册函数源码如下：

```C#
public static void AddRaycaster(BaseRaycaster baseRaycaster)
{
    if (s_Raycasters.Contains(baseRaycaster))
        return;

    s_Raycasters.Add(baseRaycaster);
}
```

逻辑非常简单：

1. 防止重复注册 

2. 加入全局列表 

最终形成：

```Plain Text
s_Raycasters
    ├── GraphicRaycaster
    ├── GraphicRaycaster
    ├── PhysicsRaycaster
    └── Physics2DRaycaster
```

### 11\.5\.5 Raycaster 注销流程

当 Raycaster 被禁用时：

```C#
BaseRaycaster.OnDisable()
```

会自动执行：

```C#
RaycasterManager.RemoveRaycasters(this);
```

源码如下：

```C#
protected override void OnDisable()
{
    RaycasterManager.RemoveRaycasters(this);
    base.OnDisable();
}
```

因此，Raycaster 生命周期完全自动管理，开发者通常不需要手动处理。

### 11\.5\.6 RemoveRaycasters

注销函数源码如下：

```C#
public static void RemoveRaycasters(BaseRaycaster baseRaycaster)
{
    if (!s_Raycasters.Contains(baseRaycaster))
        return;

    s_Raycasters.Remove(baseRaycaster);
}
```

这里会从全局列表中移除当前 Raycaster。

从而保证 RaycasterManager 中始终只保留有效 Raycaster。

### 11\.5\.7 EventSystem 如何使用 RaycasterManager

在输入发生时，EventSystem 会执行：

```C#
EventSystem.RaycastAll()
```

内部源码如下：

```C#
var modules = RaycasterManager.GetRaycasters();
```

随后：

```C#
for (int i = 0; i < modules.Count; ++i)
{
    var module = modules[i];

    if (module == null || !module.IsActive())
        continue;

    module.Raycast(eventData, raycastResults);
}
```

这里说明：

EventSystem 并不关心：

- GraphicRaycaster 

- PhysicsRaycaster 

- 自定义 Raycaster 

它只负责统一调度。

真正的检测逻辑位于各个 Raycaster 内部。

### 11\.5\.8 GetRaycasters

获取函数源码如下：

```C#
public static List<BaseRaycaster> GetRaycasters()
{
    return s_Raycasters;
}
```

这里直接返回全局列表。

时间复杂度：O\(1\)

因此，EventSystem 可以快速获得所有 Raycaster。

### 11\.5\.9 BaseRaycaster 的统一抽象

RaycasterManager 能统一管理所有 Raycaster，核心原因在于所有 Raycaster 都继承自 BaseRaycaster。

源码定义如下：

```C#
public abstract class BaseRaycaster : UIBehaviour
```

核心接口：

```C#
public abstract void Raycast(
    PointerEventData eventData,
    List<RaycastResult> resultAppendList
);
```

因此，无论是：

- UI 射线 

- 3D 射线 

- 2D 射线 

- XR 射线 

最终都会被统一抽象成 Raycast\(\)

这就是 EventSystem 能支持多输入系统的关键。

### 11\.5\.10 多 Raycaster 协同工作

UGUI 支持多个 Raycaster 同时工作。

例如：

- GraphicRaycaster

- PhysicsRaycaster

- Physics2DRaycaster

点击发生时，EventSystem 会：

```Plain Text
依次调用所有 Raycaster
        ↓
合并所有 RaycastResult
        ↓
统一排序
        ↓
选择优先级最高对象
```

因此，UI 与 3D 场景输入可以同时存在。

例如：

- UI Button 

- 3D 模型点击 

- 2D Sprite 点击 

都能进入同一事件系统。

### 11\.5\.11 Raycaster 的优先级

多个 Raycaster 同时命中时：

UGUI 会根据：

- sortingLayer 

- sortingOrder 

- depth 

- distance 

进行综合排序。

最终优先级最高对象获得事件。

例如：

```Plain Text
Top UI
    ↓
Main UI
    ↓
3D Object
```

如果 UI 位于最上层，则不会点击穿透到底层 3D。

### 11\.5\.12 自定义 Raycaster

开发者可以自己扩展 Raycaster。

例如：

```C#
public class MyRaycaster : BaseRaycaster
{
    public override void Raycast(
        PointerEventData eventData,
        List<RaycastResult> resultAppendList)
    {

    }
}
```

当组件启用时，自动注册到 RaycasterManager。

因此，EventSystem 会自动参与调度。

这也是 UGUI 输入系统扩展性极强的重要原因。

### 11\.5\.13 RaycasterManager 与 GraphicRegistry 的区别

很多开发者容易混淆：

- GraphicRegistry 

- RaycasterManager 

两者区别如下：

进一步来说：

它们属于不同层级。

整个关系如下：

```Plain Text
EventSystem
    ↓
RaycasterManager
    ↓
GraphicRaycaster
    ↓
GraphicRegistry
    ↓
Graphic
```

### 11\.5\.14 RaycasterManager 的性能特点

RaycasterManager 的设计非常轻量。

它只负责：

- 注册 

- 注销 

- 提供列表 

真正的性能开销主要位于：

```C#
GraphicRaycaster.Raycast()
```

因为 RaycasterManager 本身并不执行任何检测。

因此，即使场景中存在多个 Raycaster，管理成本依然很低。

### 11\.5\.15 小结

本节深入分析了 RaycasterManager 的工作机制。

RaycasterManager 本质上是 UGUI 的全局 Raycaster 注册中心。

其核心结构如下：

```C#
List<BaseRaycaster>
```

整个注册流程如下：

```Plain Text
BaseRaycaster.OnEnable
        ↓
AddRaycaster
        ↓
加入全局列表
```

注销流程如下：

```Plain Text
BaseRaycaster.OnDisable
        ↓
RemoveRaycasters
        ↓
移除 Raycaster
```

EventSystem 正是通过 RaycasterManager，才能统一调度：

- GraphicRaycaster 

- PhysicsRaycaster 

- Physics2DRaycaster 

- 自定义 Raycaster 

从而实现完整统一的输入检测系统。

## 11\.6 GraphicRaycaster 源码

GraphicRaycaster 是 UGUI 中最核心的射线检测器。

整个 UI 点击系统，本质上都依赖 GraphicRaycaster 完成命中检测。

当鼠标点击屏幕时：

- EventSystem 负责调度 

- InputModule 负责输入采集 

- GraphicRaycaster 负责 UI 命中检测 

- ExecuteEvents 负责事件派发 

因此 GraphicRaycaster 是连接“输入系统”与“UI 系统”的核心桥梁。

### 11\.6\.1 GraphicRaycaster 的继承结构

GraphicRaycaster 继承自 BaseRaycaster。

源码定义如下：

```C#
public class GraphicRaycaster : BaseRaycaster
```

因此它本质上属于 Raycaster 系统。

核心入口函数如下：

```C#
public override void Raycast(
    PointerEventData eventData,
    List<RaycastResult> resultAppendList
)
```

整个 UI 点击检测都从这里开始。

### 11\.6\.2 Raycast 主流程

GraphicRaycaster 的 Raycast 流程可以概括为：

```Plain Text
获取 Canvas Graphic
        ↓
过滤无效 Graphic
        ↓
屏幕矩形检测
        ↓
Graphic 自定义检测
        ↓
深度排序
        ↓
生成 RaycastResult
        ↓
加入结果列表
```

整个过程本质上属于屏幕空间矩形检测，而不是 3D 物理射线。

### 11\.6\.3 获取 Canvas Graphic

Raycast 开始后，首先会获取当前 Canvas 下所有 Graphic。

源码如下：

```C#
var canvasGraphics = GraphicRegistry.GetGraphicsForCanvas(canvas);
```

这里会从 GraphicRegistry 获取当前 Canvas 的 Graphic 列表。

因此，GraphicRaycaster 并不会遍历整个场景，而是直接获得缓存好的 UI 列表。

这是 UGUI 射线系统高性能的重要原因。

### 11\.6\.4 Graphic 静态缓存列表

GraphicRaycaster 内部维护了一个静态缓存：

```C#
private static readonly List<Graphic> s_SortedGraphics
    = new List<Graphic>();
```

作用是缓存当前命中的 Graphic。

这样可以避免每次 Raycast 创建新 List，从而减少 GC。

UGUI 内部大量使用这种缓存设计，这是源码中非常常见的优化方式。

### 11\.6\.5 Graphic 基础过滤

获取 Graphic 后，会先进行基础过滤。

源码如下：

```C#
if (graphic.depth == -1)
    continue;

if (!graphic.raycastTarget)
    continue;

if (graphic.canvasRenderer.cull)
    continue;
```

这些过滤条件非常关键。

- depth == \-1

表示当前 Graphic 未参与渲染，因此无需检测。

- raycastTarget == false

表示当前 UI 不接收射线。

例如：

- 装饰性 Image 

- 背景 Text 

- 非交互 UI 

通常都会关闭 raycastTarget，否则会阻挡下层 Button。

- canvasRenderer\.cull

表示当前 Graphic 已被裁剪。

例如：

- Mask 外区域 

- ScrollView 不可见区域 

这些 UI 不应继续参与检测。

### 11\.6\.6 RectTransform 屏幕检测

通过基础过滤后，UGUI 会检测当前鼠标是否位于 UI 矩形区域内。

源码如下：

```C#
RectTransformUtility.RectangleContainsScreenPoint(
    graphic.rectTransform,
    pointerPosition,
    eventCamera
)
```

这里本质上是屏幕坐标是否位于 RectTransform 内部。

注意 UGUI 默认不是像素级检测，而是矩形检测。

因此，即使 Image 透明区域，也可能响应点击。

### 11\.6\.7 屏幕坐标转换

RectangleContainsScreenPoint 内部会进行：

```Plain Text
屏幕坐标
    ↓
世界坐标
    ↓
局部坐标
    ↓
Rect 判断
```

核心函数包括：

```C#
ScreenPointToLocalPointInRectangle()
```

以及：

```C#
Rect.Contains()
```

因此，UI Raycast 本质上依赖 RectTransform 坐标系统。

### 11\.6\.8 Graphic 自定义 Raycast

矩形检测通过后，Graphic 自身还可以进一步过滤。

源码如下：

```C#
graphic.Raycast(pointerPosition, eventCamera)
```

Graphic 默认实现：

```C#
public virtual bool Raycast(
    Vector2 sp,
    Camera eventCamera
)
{
    return true;
}
```

但 Image 会重写这个函数。

### 11\.6\.9 Alpha Hit Test

Image 支持 Alpha 检测。

例如：Alpha Hit Test Minimum Threshold

源码内部会检测当前像素 Alpha 是否大于阈值。

因此，透明区域可以不响应点击。

这是解决透明区域挡点击问题的重要机制。

但 Alpha Hit Test 开销较高，因此默认关闭。

### 11\.6\.10 Mask 与 RectMask2D 检测

Graphic\.Raycast 内部还会检测：

- Mask 

- RectMask2D 

- CanvasGroup 

例如：是否被 Mask 裁剪

如果 UI 位于 Mask 外，则不会响应点击。

因此，UGUI 的输入系统与裁剪系统是联动的。

### 11\.6\.11 Ignore Reversed Graphics

GraphicRaycaster 包含一个重要选项：Ignore Reversed Graphics

开启后 UGUI 会检测 UI 是否背对摄像机。

核心代码如下：

```C#
Vector3.Dot(
    graphic.transform.forward,
    ray.direction
)
```

如果 Dot 小于 0，表示 UI 背向摄像机，则忽略点击。

这个机制主要用于：World Space Canvas

### 11\.6\.12 Blocking Objects 检测

GraphicRaycaster 还支持物理阻挡。

Inspector 中包括：Blocking Objects

支持：

开启后，GraphicRaycaster 会先执行 Physics\.Raycast。

如果中间存在 Collider，则 UI 不再接收点击。

### 11\.6\.13 阻挡距离计算

BlockingObjects 开启后，GraphicRaycaster 会计算物理命中距离

随后 UI 距离如果大于物理距离，则 UI 被阻挡。

因此，3D 物体可以挡住世界空间 UI。

### 11\.6\.14 Graphic 深度排序

所有命中的 Graphic 并不会立刻返回。

而是先加入 s\_SortedGraphics 随后执行排序：

```C#
s_SortedGraphics.Sort((g1, g2) => g2.depth.CompareTo(g1.depth));
```

这里depth越大越靠近前面。

因此，显示在最上层的 UI 会优先响应输入。

### 11\.6\.15 depth 的来源

Graphic\.depth 来源于 CanvasRenderer\.absoluteDepth。

它表示 Graphic 在整个 Canvas 中的最终渲染深度。

因此，Raycast 排序与渲染顺序完全一致。

这也是视觉层级与输入层级一致的重要原因。

### 11\.6\.16 生成 RaycastResult

排序完成后 GraphicRaycaster 会生成 RaycastResult。

核心代码如下：

```C#
var castResult = new RaycastResult
{
    gameObject = go,
    module = this,
    distance = distance,
    screenPosition = eventPosition,
    depth = m_RaycastResults[index].depth,
    sortingLayer = canvas.sortingLayerID,
    sortingOrder = canvas.sortingOrder
};
```

这里保存：

- 命中对象 

- 所属 Raycaster 

- UI 深度 

- SortingLayer 

- SortingOrder 

### 11\.6\.17 多 Canvas 排序

多个 Canvas 同时存在时 EventSystem 会合并所有 RaycastResult，随后统一排序。

排序规则包括：

1. sortingLayer 

2. sortingOrder 

3. depth 

4. distance 

因此：

```Plain Text
TopCanvas
    ↓
MainCanvas
    ↓
WorldCanvas
```

会形成稳定输入层级。

### 11\.6\.18 GraphicRaycaster 与 PhysicsRaycaster 的区别

GraphicRaycaster：基于 Graphic

PhysicsRaycaster：基于 Collider

两者区别如下：

UGUI 点击完全不依赖物理系统。

### 11\.6\.19 GraphicRaycaster 的性能瓶颈

GraphicRaycaster 最大开销来源于Graphic 遍历。

因为每次 Raycast 都需要遍历当前 Canvas 下所有 Graphic。

因此复杂 UI 中：

- raycastTarget 过多 

- Canvas UI 过多 

- 大量 Mask 

- 深层嵌套 

都会影响点击性能。

优化建议包括：

1. 关闭无用 raycastTarget 

2. 减少超大 Canvas 

3. 避免大量 Mask 

4. 减少层级嵌套 

5. 分离静态与动态 Canvas 

## 11\.7 新输入系统支持

UGUI 在早期版本中依赖旧输入系统（Input Manager），其输入来源主要来自 Input\.GetMouseButton、Input\.mousePosition 等 API。随着 Unity 新输入系统（Input System Package）的引入，UGUI 需要一套新的输入适配层来兼容设备输入模型的变化。

新输入系统的核心特点是事件驱动与设备抽象化，不再直接提供“鼠标/键盘”这样的固定入口，而是统一抽象为 Action 与 Device。因此 UGUI 必须通过 InputSystemUIInputModule 来完成对 EventSystem 的接入。

### 11\.7\.1 新输入系统架构变化

旧输入系统结构如下：

```Plain Text
Input Manager
    ↓
StandaloneInputModule
    ↓
PointerEventData
    ↓
UGUI EventSystem
```

新输入系统结构如下：

```Plain Text
Input System (Action / Device)
    ↓
InputSystemUIInputModule
    ↓
PointerEventData
    ↓
UGUI EventSystem
```

可以看到变化核心在于 StandaloneInputModule 被 InputSystemUIInputModule 替代。

### 11\.7\.2 InputSystemUIInputModule 的作用

InputSystemUIInputModule 是 UGUI 对新输入系统的官方适配模块，其职责是：

1. 监听 Input Action 

2. 转换为 PointerEventData 

3. 触发 EventSystem 输入流程 

4. 统一鼠标、触摸、手柄输入 

其核心类定义如下：

```C#
public class InputSystemUIInputModule : BaseInputModule
```

因此它仍然属于 UGUI 输入模块体系。

但与旧模块不同，它不再直接读取 Input，而是依赖 Input Action。

### 11\.7\.3 输入 Action 映射

新输入系统通过 InputActionAsset 定义输入行为，例如：

- Point（鼠标/触摸位置） 

- LeftClick（点击） 

- RightClick（右键） 

- MiddleClick（中键） 

- ScrollWheel（滚轮） 

- Navigate（键盘/手柄导航） 

这些 Action 在 UI 模块中被绑定到 PointerEventData。

例如：

```C#
Pointer.position ← Point Action
Pointer.press ← LeftClick Action
Pointer.scroll ← ScrollWheel Action
```

因此，UGUI 不再直接依赖设备，而是依赖抽象输入行为。

### 11\.7\.4 Pointer State 构建流程

InputSystemUIInputModule 的核心流程仍然是 Process 方法：

```C#
public override void Process()
```

其内部逻辑可以概括为：

```Plain Text
读取 InputAction
    ↓
构建 PointerEventData
    ↓
执行 Raycast
    ↓
更新 Pointer 状态
    ↓
触发 UI 事件
```

其中最关键步骤是 PointerEventData 构建。

### 11\.7\.5 PointerEventData 在新输入系统中的变化

在新输入系统中，PointerEventData 不再依赖 Input\.mousePosition，而是来自 Action：

```C#
eventData.position = m_Point.action.ReadValue<Vector2>();
```

点击状态来源：

```C#
eventData.eligibleForClick = m_LeftClick.action.triggered;
```

滚轮来源：

```C#
eventData.scrollDelta = m_ScrollWheel.action.ReadValue<Vector2>();
```

因此所有输入来源被统一抽象为 InputAction。

### 11\.7\.6 多设备统一输入模型

新输入系统最大的变化是所有设备统一抽象为 Action。

例如：

但 UGUI 不再关心设备类型，而只关心 InputAction。

因此同一个 Button 可以同时响应：

- 鼠标点击 

- 触摸点击 

- 手柄确认 

### 11\.7\.7 EventSystem 与 InputModule 关系变化

在新输入系统中 EventSystem 仍然不变，但 InputModule 行为发生变化。

结构如下：

```Plain Text
EventSystem
    ↓
InputSystemUIInputModule
    ↓
PointerEventData
    ↓
RaycasterManager
    ↓
GraphicRaycaster
```

EventSystem 仍然只负责调度，不参与输入来源判断。

### 11\.7\.8 新输入系统 Raycast 流程

新输入系统 Raycast 流程与旧系统完全一致：

```Plain Text
InputSystemUIInputModule
    ↓
EventSystem.RaycastAll
    ↓
RaycasterManager
    ↓
GraphicRaycaster
    ↓
RaycastResult
```

因此输入系统变化不会影响 UI 射线系统。

UGUI 的核心 Raycast 架构保持稳定。

### 11\.7\.9 输入焦点与 Pointer Lock

新输入系统引入 Pointer Lock 概念：

例如：

- 锁定鼠标 

- 第一人称视角 

- FPS 游戏 

在 Pointer Lock 模式下鼠标位置不再移动 UI 光标，而是转化为 delta movement。

UGUI 仍然可以接收输入，但方式不同。

### 11\.7\.10 UI 导航系统兼容

新输入系统仍然支持旧 UI Navigation：

例如：

```Plain Text
Button A → Submit
Button B → Cancel
方向键 → Navigate
```

InputSystemUIInputModule 会将 Navigate Action 转换为MoveEvent。

从而驱动：

- Button 切换 

- InputField 焦点移动 

- Toggle 选择 

### 11\.7\.11 UI 与 Input Action 分层设计

新输入系统设计中 UGUI 不再直接读取输入设备，而是分为三层：

```Plain Text
Device Layer（设备层）
    ↓
Action Layer（行为层）
    ↓
UI Layer（事件层）
```

其中：

- Device：鼠标、键盘、手柄 

- Action：点击、移动、滚动 

- UI：PointerEventData、ExecuteEvents 

这种设计提升了：

- 可扩展性 

- 跨平台能力 

- 输入统一性 

### 11\.7\.12 与旧 InputModule 的差异

### 11\.7\.13 性能与事件驱动优势

新输入系统采用事件驱动模型：

```Plain Text
InputAction triggered
    ↓
回调 UI Module
    ↓
生成 PointerEventData
    ↓
进入 EventSystem
```

相比旧系统：

- 不再轮询 Input 

- 减少 CPU 消耗 

- 多设备统一调度 

因此在复杂 UI 或多输入设备场景下性能更优。

## 11\.8 UI 输入与帧同步

UGUI 的输入系统完全运行在 Unity 主线程中，因此所有 UI 输入本质上都与帧同步绑定。无论鼠标点击、拖拽、Hover、键盘输入还是手柄导航，最终都依赖 EventSystem 在 Update 中逐帧驱动。

整个输入流程如下：

```Plain Text
PlayerLoop
    ↓
Update
    ↓
EventSystem.Update
    ↓
InputModule.Process
    ↓
Raycast
    ↓
ExecuteEvents
    ↓
UI 响应
```

可以看到，UGUI 输入并不存在独立线程，也不存在后台输入循环，所有输入都只能在当前帧中处理。因此当帧率下降时，输入响应也会同步下降。

### 11\.8\.1 EventSystem 的帧驱动本质

EventSystem 本质上是一个 UIBehaviour。

源码如下：

```C#
public class EventSystem : UIBehaviour
```

其核心逻辑位于：

```C#
protected virtual void Update()
```

因此 EventSystem 每帧只会更新一次。

所有输入检测、Raycast、事件派发，也全部只会执行一次。

这意味着输入频率等于当前帧率。

如果游戏运行在 30FPS，则输入系统每秒只能处理 30 次输入更新。

### 11\.8\.2 输入采样与帧率关系

旧输入系统中的：

```C#
Input.GetMouseButtonDown()
```

本质上也是当前帧输入状态。

StandaloneInputModule 会在：

```C#
ProcessMouseEvent()
```

中读取这些状态。

因此如果某一帧卡顿输入读取也会延迟。

这也是低帧率下 Button 点击迟钝的根本原因。

### 11\.8\.3 UI 输入与 FixedUpdate 无关

UGUI 输入系统完全不依赖 FixedUpdate。

其执行流程始终为：

```Plain Text
Update
    ↓
EventSystem.Update
```

而不是FixedUpdate。

因此修改Time\.fixedDeltaTime，不会影响 UI 点击频率。

很多开发者误以为输入属于物理系统。

实际上 UI 输入与物理帧没有关系。

### 11\.8\.4 UI 输入不受 timeScale 影响

即使：

```C#
Time.timeScale = 0;
```

UGUI 输入依然有效。

例如：

- Button 仍可点击 

- Hover 仍会更新 

- InputField 仍能输入 

原因在于EventSystem\.Update 并不会停止。

UGUI 输入系统默认属于unscaled update。

因此暂停游戏并不会暂停 UI。

### 11\.8\.5 PointerEventData 的跨帧状态

UGUI 点击并不是单帧完成，而是跨帧状态机。

例如一次点击流程：

- 第一帧：PointerDown

- 第二帧：等待输入

- 第三帧：PointerUp

- 第四帧：Click 判定

中间状态全部保存在PointerEventData

例如：

```C#
pointerPress
eligibleForClick
dragging
clickTime
clickCount
```

因此 PointerEventData 本质上是跨帧输入状态容器。

### 11\.8\.6 拖拽属于多帧行为

拖拽系统本质上属于持续输入行为。

典型流程如下：

- 第一帧：PointerDown

- 第二帧：开始移动

- 第三帧：Dragging

- 第四帧：持续拖拽

- 第五帧：PointerUp

因此eventData\.dragging必须持续跨帧保存。

一旦进入拖拽状态：

```C#
eligibleForClick = false;
```

从而取消点击资格。

这样可以避免拖拽与点击同时触发。

### 11\.8\.7 双击检测机制

UGUI 双击检测依赖：

```C#
Time.unscaledTime
```

源码如下：

```C#
if ((Time.unscaledTime - clickTime) < 0.3f)
    clickCount++;
```

这里使用 unscaledTime，而不是 deltaTime。

原因在于双击检测不应受 timeScale 影响。

即使游戏暂停，双击判定依然正常。

### 11\.8\.8 UI 输入与协程关系

很多 UI 动画使用协程StartCoroutine\(\)，但协程并不会影响 EventSystem。

例如：

```Plain Text
Button 点击
    ↓
启动协程
    ↓
下一帧继续执行
```

UI 输入系统仍然正常运行。

因此协程与 EventSystem 属于并行逻辑。

### 11\.8\.9 UI 输入与网络同步

UGUI 输入默认属于本地输入，并不会自动网络同步。

例如：客户端点击 Button，服务器并不会自动收到事件。

开发者通常需要：

```Plain Text
Button.onClick
    ↓
RPC
    ↓
服务器逻辑
```

因此UGUI 本身不负责网络同步

### 11\.8\.10 UI 输入与帧同步游戏

在帧同步游戏中：

例如：

- MOBA 

- RTS 

- 格斗游戏 

通常不会同步 UI 点击。

原因在于UI 输入依赖本地环境。

包括：

- 分辨率 

- Canvas 

- 摄像机 

- 帧率 

因此帧同步系统通常只同步逻辑输入，而不是UI 输入事件。

### 11\.8\.11 UI 输入性能瓶颈

UGUI 输入性能问题通常来自：

因此：UI 输入优化 = UI 性能优化

### 11\.8\.12 输入系统的本质

UGUI 输入系统本质上是：

```Plain Text
主线程
    ↓
Update 驱动
    ↓
跨帧状态机
    ↓
事件派发系统
```

它并不是：

- 实时硬件输入 

- 独立线程输入 

- 后台异步输入 

因此所有 UI 输入最终都受当前帧影响

## 本章小结

本章围绕 EventSystem 事件系统展开，系统性分析了 UGUI 中“从输入到交互”的完整实现链路。

首先，从输入与事件分发机制入手，明确了 EventSystem 的核心职责：将底层输入设备的状态变化，转化为标准化 UI 事件，并通过统一接口分发给目标对象。在这一过程中，InputModule 负责输入采集，EventSystem 负责调度，而 UI 组件只负责事件响应，从而形成清晰的职责划分。

随后，通过 Raycast 机制的分析，建立了“输入坐标到 UI 命中对象”的映射过程。GraphicRaycaster 作为核心实现，依赖 GraphicRegistry 提供候选集合，并通过矩形检测、裁剪判断与深度排序，完成精确命中计算。RaycasterManager 则从全局层面管理所有 Raycaster，使不同射线体系能够统一调度。

在 UI 点击流程中，可以看到事件系统并非简单触发单一事件，而是由 PointerDown、PointerUp、PointerClick 等多个阶段组成的完整生命周期。PointerEventData 在其中承担状态记录与数据传递的核心角色，使复杂交互（如拖拽、悬停）能够稳定实现。

进一步地，GraphicRegistry 与 RaycasterManager 的分析揭示了 UGUI 在性能优化上的设计思路：通过“注册表 \+ 集中管理”的方式，将全局遍历转化为局部查询，从而显著降低 Raycast 开销。

在新输入系统支持部分，可以看到 UGUI 通过 InputSystemUIInputModule，将输入采集从传统轮询模型升级为基于 Action 的抽象模型。在保持事件分发体系不变的前提下，实现了对多设备与复杂输入场景的扩展支持。

最后，通过 UI 输入与帧同步机制的分析，明确了 EventSystem 在 Unity 帧循环中的执行位置，以及输入、UI 更新与渲染之间的时序关系。输入事件基于帧间状态变化触发，而 UI 几何更新与渲染则存在一帧延迟，这一特性是理解交互一致性与调试问题的关键。

从整体架构来看，EventSystem 本质上是 UGUI 的“输入中间层”。它将输入系统、空间检测与事件分发整合为一套统一流程，使 UI 能够在多平台、多设备环境下保持一致的交互行为。

至此，UGUI 的三大核心体系已经逐步建立：以 RectTransform 与 Layout 为基础的空间系统，以 Graphic 为核心的几何生成系统，以及以 EventSystem 为核心的交互系统。三者相互协作，共同构成完整的 UI 框架。



