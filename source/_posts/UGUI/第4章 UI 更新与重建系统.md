---
title: 第4章 UI 更新与重建系统
categories:
  - Unity
  - UGUI
tags:
  - UGUI
  - Unity
cover: /images/UGUI/UGUI.png
abbrlink: 96d8841d
date: 2026-08-26 00:00:04
---
# 第4章 UI 更新与重建系统

在完成 RectTransform 的空间计算模型之后，UI 的结构层已经具备了“可计算的矩形表达能力”。然而，从运行时执行流程来看，这些矩形数据并不会自动参与渲染，它们只是 UI 系统中的中间结果。要将这些数据转化为最终可提交至渲染管线的 Mesh，必须依赖一套完整且严格调度的更新与重建机制。

UGUI 的核心设计之一，是围绕“延迟重建（Deferred Rebuild）”展开的更新策略。与立即更新不同，UI 系统在属性发生变化时，并不会立刻触发计算或生成 Mesh，而是通过“脏标记（Dirty Flag）”记录变化状态，并将这些变化延迟到统一的更新阶段进行批处理。这种机制显著降低了频繁属性修改带来的重复计算成本，但同时也使得 UI 更新流程呈现出明显的“阶段化”与“调度驱动”特征。

从整体架构来看，UI 更新系统并不是单一流程，而是由多个子系统协同完成，主要包括：

（1）RectTransform 驱动的布局系统（Layout System），负责计算 UI 元素的尺寸与位置；

（2）Graphic 系统，负责将 UI 数据转化为顶点信息与 Mesh；

（3）Canvas 系统，负责收集、排序并提交渲染数据；

（4）更新调度系统（CanvasUpdateRegistry），负责在合适的时机统一触发各类重建逻辑。

这些子系统之间并非独立运行，而是通过“脏标记传播 \+ 队列调度”的方式形成一条完整的数据更新链路。任意一个 UI 元素的变化，都会沿着层级结构向上传播，并最终影响整个 Canvas 的渲染结果。

需要特别指出的是，UGUI 的更新机制并不是即时可见的线性流程，而是一个跨帧调度的系统。开发者在代码中调用如 SetDirty、SetVerticesDirty 或修改 RectTransform 属性时，实际只是向系统“登记变更”，真正的计算与重建发生在后续的 Canvas 更新阶段。这种设计在提升性能的同时，也增加了调试与理解的复杂度。

因此，理解 UI 更新系统的关键，不在于单个 API 的行为，而在于掌握以下几个核心问题：

（1）UI 的“变化”是如何被记录的；

（2）脏标记是如何在层级结构中传播的；

（3）不同类型的更新（Layout、Graphic）如何被调度与排序；

（4）Mesh 的生成是在什么时机、以什么顺序完成的。

本章将围绕上述问题展开，从 UI 更新生命周期入手，逐步深入分析脏标记机制、更新队列调度、Rebuild 阶段划分以及 Mesh 生成过程，系统还原 UGUI 从“状态变化”到“渲染数据生成”的完整执行路径。同时，在关键节点处结合源码结构，揭示其设计意图与性能优化策略，为后续深入理解渲染流程打下基础。

## 4\.1 UI 更新生命周期

在 UGUI 中，UI 的变化并不会立即反映到屏幕上，而是遵循一套严格的更新生命周期。这一机制的核心目标，是将分散的 UI 修改统一收敛到固定的执行阶段，从而避免频繁的即时计算与重复渲染。

从系统设计角度来看，UGUI 的更新流程本质上是一种“状态驱动的分阶段处理模型”。所有 UI 变化都会被延迟处理，并在统一的更新节点中集中执行。

### 4\.1\.1 生命周期整体模型

UI 更新流程可以抽象为一个完整的处理链路，其核心由四个阶段构成：

状态变更阶段 → 脏标记阶段 → 调度阶段 → 重建阶段

在这一流程中，UI 系统不会在修改发生的瞬间执行计算，而是先收集变化，再统一处理。这种机制使得多个 UI 修改可以在同一帧内被合并执行，从而显著降低系统开销。

需要强调的是，这四个阶段并非彼此独立，而是形成一条连续的数据处理链路。前一阶段的输出，将作为后一阶段的输入，最终完成 UI 的更新。

### 4\.1\.2 状态变更阶段

状态变更阶段是 UI 更新流程的起点。

当 UI 发生变化时，例如：

- RectTransform 属性被修改

- 文本内容发生改变

- Image 组件更换 Sprite

- UI 元素层级结构调整

系统并不会立即执行布局计算或顶点重建，而是仅记录当前数据状态的变化。

这一阶段的核心特点是：只修改数据，不触发计算

这种设计避免了在频繁修改 UI 时产生大量重复计算，是整个生命周期性能优化的基础。

### 4\.1\.3 脏标记阶段

在状态发生变化后，系统会进入脏标记阶段。

UI 系统会根据变化的类型，对相关组件打上不同类型的“Dirty 标记”，常见类型包括：

- LayoutDirty —— 表示布局需要重新计算

- VerticesDirty —— 表示顶点数据需要更新

- MaterialDirty —— 表示材质或渲染状态需要更新

这些标记不会立即触发重建，而是用于记录“哪些内容需要更新”，为后续调度阶段提供依据。

从源码角度来看，这一阶段的核心入口通常包括：

- SetVerticesDirty

- SetLayoutDirty

- SetMaterialDirty

这些接口构成了 UI 更新系统的“触发点”，后续章节将对此进行详细分析。

### 4\.1\.4 调度阶段

调度阶段是 UI 更新系统的核心管理环节。

Unity 通过 CanvasUpdateRegistry 对所有被标记的 UI 对象进行统一管理，并将它们加入不同的更新队列中。

在特定的帧更新时机，CanvasUpdateRegistry 会按照既定顺序遍历这些队列，并调用对应组件的重建接口。

这一机制的关键作用在于：

- 将分散在各个组件中的更新请求统一集中

- 保证 UI 更新顺序的稳定性与可控性

- 避免重复执行同一对象的多次重建

调度阶段本质上是一个“任务收集与批处理执行”的过程，是连接标记与实际重建之间的桥梁。

### 4\.1\.5 重建阶段

在调度阶段完成后，系统进入重建阶段。

重建过程通常分为两个主要部分：

一、Layout 重建

系统首先计算所有参与布局的 RectTransform，确定最终的位置与尺寸。这一过程由 Layout 系统完成。

二、Graphic 重建

在布局结果确定后，Graphic 系统根据最新数据生成顶点信息（UIVertex），并构建 Mesh 数据。

最终，这些 Mesh 会提交给 CanvasRenderer，由底层渲染系统完成绘制。

这一阶段的执行顺序具有严格约束，必须先完成布局计算，再进行图形重建。

否则将导致渲染数据与布局结果不一致。

### 4\.1\.6 执行时机与帧更新关系

UI 更新生命周期并不是随时触发的，而是依赖 Unity 的帧更新机制，在固定的时间点统一执行。

通常情况下，UI 的重建发生在渲染前的特定阶段（如 Canvas 相关更新阶段），这意味着UI 的逻辑状态变化与最终显示结果之间存在一个帧级延迟。

这种“延迟生效”特性是 UGUI 的核心设计之一。

其优势在于：

- 可以将同一帧内的多次修改合并处理

- 减少不必要的重复计算与 Mesh 重建

- 提升整体运行效率

但与此同时，也带来一定的开发认知成本。如果不了解这一机制，开发者容易误判 UI 是否已经更新，从而产生逻辑错误。

### 小结

UI 更新生命周期本质上是一个“状态收集 \+ 延迟计算 \+ 统一重建”的流程。

它通过脏标记机制收集变化，通过调度系统统一管理更新任务，并在固定的执行阶段完成布局与渲染数据的重建。

理解这一生命周期，是深入掌握 UGUI 渲染机制与性能优化策略的基础。

## 4\.2 Dirty 标记机制

在 UGUI 的更新体系中，Dirty 标记机制是连接“状态变化”与“重建执行”的核心桥梁。它本身并不负责执行任何计算，而是对 UI 组件的变化进行分类记录，从而决定后续需要参与哪些更新流程。

从系统设计角度来看，Dirty 机制承担的是“变化收集器”的角色，是整个 UI 更新生命周期中的关键基础设施。

### 4\.2\.1 Dirty 机制整体设计

Dirty 标记机制的核心思想是“按需更新”。

当 UI 发生变化时，系统不会立即执行完整的重建流程，而是仅记录变化类型，并在统一的更新阶段按需处理。这种设计使得多个 UI 修改可以被合并执行，从而避免重复计算。

其基本流程可以概括为：状态变化 → 设置 Dirty 标记 → 注册到更新队列 → 延迟执行重建

在这一过程中，Dirty 标记起到的是“任务描述”的作用，而非执行逻辑。

### 4\.2\.2 Dirty 标记分类

在 UGUI 中，Dirty 标记主要分为三类，每一类对应不同的更新阶段。

#### 一、Layout Dirty

Layout Dirty 用于表示布局相关变化。

当以下情况发生时，通常会触发该标记：

- RectTransform 尺寸或位置发生变化

- 父子层级结构改变

- LayoutGroup 或 ContentSizeFitter 状态更新

该标记会在后续更新阶段触发布局系统重新计算 RectTransform 的最终几何结果。

#### 二、Vertices Dirty

Vertices Dirty 用于表示顶点数据需要更新。

常见触发场景包括：

- Text 内容变化

- Image 的 Sprite 发生变化

- 颜色、透明度等视觉属性修改

在重建阶段，该标记会驱动 Graphic 系统重新生成 UIVertex 数据，并构建新的 Mesh。

#### 三、Material Dirty

Material Dirty 用于表示材质或渲染状态变化。

例如：

- 材质切换

- 纹理变化

- Shader 参数修改

该标记会在渲染准备阶段触发材质重新绑定，确保 GPU 侧资源与当前 UI 状态一致。

### 4\.2\.3 Dirty 标记的内部实现

从源码角度来看，Dirty 标记并不是一个统一的全局结构，而是分散存储在各个 UI 组件内部。

以 Graphic 为例，其内部通常包含如下状态字段：

- m\_VertsDirty

- m\_MaterialDirty

这些字段用于标记当前对象是否需要执行对应类型的重建。

当调用 SetVerticesDirty 或 SetMaterialDirty 时，本质上是：

- 将对应标记位设置为 true

- 将当前对象注册到 CanvasUpdateRegistry

对于布局系统而言，Layout Dirty 并不直接存储在组件内部，而是通过 LayoutRebuilder 进行统一管理。

这种设计使得：

- Graphic 系统以组件为单位管理状态

- Layout 系统以 RectTransform 为核心进行批处理

两者在实现层面存在明显差异。

### 4\.2\.4 Dirty 与重建流程的关系

Dirty 标记并不会立即触发计算，而是作为后续调度的依据。

在 CanvasUpdateRegistry 的更新过程中，系统会：

- 遍历所有注册的 UI 对象

- 检查其 Dirty 状态

- 根据标记类型调用对应的 Rebuild 接口

不同类型的 Dirty 标记对应不同的重建流程：

- Layout Dirty → 触发 LayoutRebuilder

- Vertices Dirty → 调用 Graphic\.OnPopulateMesh

- Material Dirty → 更新 CanvasRenderer 材质状态

因此可以认为：Dirty 标记决定“做什么”，而 Rebuild 阶段决定“怎么做”

### 4\.2\.5 Dirty 标记的传播机制

Dirty 标记并不仅作用于单个组件，还具有一定的传播特性。

当布局相关对象发生变化时，系统通常会向上传递影响，例如：

- 子节点尺寸变化 → 父级 LayoutGroup 需要重新计算

- 父级尺寸变化 → 子节点布局可能需要更新

这种传播机制通常通过以下方式实现：

- 向上查找父级 RectTransform

- 查找是否存在 LayoutGroup

- 将对应节点加入 Layout 重建队列

这种设计保证了 UI 层级之间的依赖关系可以被正确处理。

但与此同时，也可能导致“级联重建”问题：一个节点的变化，可能引发整棵 UI 子树的重新计算

### 4\.2\.6 性能特征与使用注意事项

从性能角度来看，Dirty 机制的核心价值在于“最小化更新范围”。

理想情况下只有发生变化的 UI 元素会参与重建，未变化的部分可以复用已有数据。

但在实际开发中，如果使用不当，Dirty 机制也可能成为性能瓶颈。

常见问题包括：

- 频繁修改 RectTransform，导致 Layout 重建反复触发

- 在 Update 中不断调用 SetVerticesDirty

- 不必要的材质切换导致 Material Dirty 高频触发

这些行为都会导致 UI 系统频繁进入重建阶段，从而增加 CPU 与 GPU 开销。

因此，在使用 UGUI 时，需要特别注意：

- 减少不必要的布局变化

- 避免重复设置相同属性

- 合理控制 UI 更新频率

### 小结

Dirty 标记机制本质上是一套“变化分类与延迟执行”的系统。

它通过对 UI 状态变化进行分级标记，将复杂的实时更新问题转化为可调度的批处理任务，并为后续的重建流程提供精确的执行依据。

理解 Dirty 机制，是掌握 UGUI 性能优化与更新流程控制的关键基础。

## 4\.3 SetVerticesDirty 与 SetLayoutDirty

在 UGUI 的 Dirty 标记体系中，SetVerticesDirty 与 SetLayoutDirty 是两个最核心的触发入口。它们分别对应“视觉数据变化”与“布局结构变化”，并直接决定 UI 更新流程将进入哪个重建分支。

从系统设计角度来看，这两个接口并不负责执行更新，而是用于将 UI 状态变化“上报”给系统，是整个 UI 更新机制中的入口节点。

### 4\.3\.1 接口设计与职责定位

UGUI 将 UI 更新拆分为两个核心子系统：

- Layout 系统 —— 负责 RectTransform 的布局计算

- Graphic 系统 —— 负责顶点生成与渲染数据构建

SetLayoutDirty 与 SetVerticesDirty 分别作为这两个系统的触发入口，其职责可以概括为：

- SetLayoutDirty —— 标记“几何结构需要变化”

- SetVerticesDirty —— 标记“渲染数据需要更新”

二者通过 Dirty 标记机制与 CanvasUpdateRegistry 进行连接，从而实现“解耦触发、统一调度”的设计目标。

### 4\.3\.2 SetVerticesDirty 执行机制

SetVerticesDirty 主要用于标记 UI 的顶点数据失效。

当 UI 的视觉表现发生变化时，例如：

- Text 内容更新

- Image 的 Sprite 发生变化

- 颜色或透明度修改

- 自定义 Graphic 的数据变化

系统会调用该方法通知当前 Graphic，其 Mesh 需要重新生成。

从源码流程来看，其核心执行路径如下：

1. 设置 m\_VertsDirty 为 true

2. 调用 CanvasUpdateRegistry\.RegisterCanvasElementForGraphicRebuild

3. 将当前对象加入 Graphic 重建队列

在后续的 Canvas 更新阶段，系统会调用Graphic\.Rebuild → OnPopulateMesh

从而重新生成 UIVertex 数据，并构建 Mesh 提交给 CanvasRenderer。

需要注意的是SetVerticesDirty 仅影响渲染数据，不会改变布局结果，不会触发 Layout 系统的计算流程

因此，它属于“局部更新”，成本相对较低。

### 4\.3\.3 SetLayoutDirty 执行机制

SetLayoutDirty 用于标记 UI 的布局结构发生变化。

当以下情况发生时，通常会触发该方法：

- RectTransform 尺寸或锚点变化

- 父子层级结构调整

- LayoutGroup 或 ContentSizeFitter 状态更新

与 SetVerticesDirty 不同，SetLayoutDirty 的实现并不直接依赖 Graphic，而是通过 LayoutRebuilder 参与更新流程。

其核心执行路径为：

1. 调用 LayoutRebuilder\.MarkLayoutForRebuild

2. 向上查找可参与布局计算的根节点

3. 将该节点注册到 CanvasUpdateRegistry 的 Layout 队列

在后续更新阶段，系统会执行LayoutRebuilder\.Rebuild，从而完成整个布局树的重新计算。

需要强调的是SetLayoutDirty 的影响具有传播性，通常会作用于整个布局子树，而非单一节点。

因此，其性能开销通常高于 SetVerticesDirty。

### 4\.3\.4 两者差异与协作关系

从系统行为角度来看，SetVerticesDirty 与 SetLayoutDirty 存在明显差异。

影响范围不同：

- SetVerticesDirty 仅影响当前 Graphic

- SetLayoutDirty 可能影响整个布局层级

执行优先级不同：

- Layout 更新必须先于 Graphic 更新执行

- 因为布局结果决定顶点生成的基础数据

传播机制不同：

- Vertices Dirty 不会传播

- Layout Dirty 会沿层级结构扩散

从协作关系来看，两者并非独立，而是存在链式影响：

布局变化 → RectTransform 改变 → 顶点数据失效 → 触发 Vertices Dirty

因此，在完整更新流程中：

- SetLayoutDirty 往往是“源头变化”

- SetVerticesDirty 则是“结果变化”

这种分层设计保证了 UI 更新流程的正确性与一致性。

### 4\.3\.5 调用时机与性能影响

在实际开发中，这两个接口的调用频率直接影响 UI 性能。

常见问题包括：

- 在 Update 中频繁调用 SetVerticesDirty

- 反复修改 RectTransform 导致连续触发 SetLayoutDirty

- 同一帧内多次触发相同标记

这些行为会导致：

- UI 被反复加入重建队列

- CanvasUpdateRegistry 中任务数量膨胀

- 产生大量无效计算与 Mesh 重建

因此，在使用时需要遵循以下原则：

- 尽量合并 UI 状态修改

- 避免重复设置相同属性

- 减少不必要的布局变化

合理使用 Dirty 标记机制，是优化 UGUI 性能的关键。

### 小结

SetVerticesDirty 与 SetLayoutDirty 是 UGUI 更新体系的两个核心入口。

它们分别将 UI 状态变化引导至 Graphic 系统与 Layout 系统，并通过 Dirty 标记机制接入统一调度流程。

理解这两个接口的行为与差异，有助于深入掌握 UI 更新链路，并为后续分析 CanvasUpdateRegistry 与 Rebuild 机制奠定基础。

## 4\.4 脏标记传播路径

在 UGUI 的更新机制中，Dirty 标记并不仅仅作用于单个节点，而是会沿着 UI 层级结构发生扩散，从而影响整个 UI 树的更新范围。

理解脏标记的传播路径，是分析 UI 重建范围与性能问题的关键基础。

### 4\.4\.1 脏标记传播模型

从整体上看，UGUI 的脏标记传播可以抽象为一种“基于层级依赖的状态扩散模型”。

其核心特点包括：

- 以变化节点为起点

- 沿 UI 层级结构传播影响

- 在统一调度阶段集中处理

在逻辑上，这种传播可以分为两个方向：

- 向上传播 —— 用于解决布局依赖关系

- 向下影响 —— 用于保证视觉结果一致

需要特别强调的是UGUI 中只有“向上传播”是显式实现的机制， “向下传播”更多是间接结果，而非独立的标记流程。

### 4\.4\.2 Layout 向上传播机制

向上传播主要发生在 Layout 系统中，是脏标记传播的核心路径。

当子节点发生布局变化时，例如：

- RectTransform 尺寸或锚点变化

- ContentSizeFitter 更新

- LayoutElement 参数修改

这些变化可能影响父级容器的布局结果。因此，系统会将 Layout Dirty 标记向上传递。

从源码角度来看，这一过程的核心入口是：LayoutRebuilder\.MarkLayoutForRebuild

其执行逻辑包括：

1. 从当前节点开始向上查找父级 RectTransform

2. 查找是否存在可参与布局的组件（如 LayoutGroup）

3. 选择合适的“布局根节点”

4. 将该节点加入 Layout 重建队列

这一过程会一直向上查找，直到遇到不参与布局的节点或到达 Canvas 边界。

最终，系统只会记录“需要重建的最小根节点”，而不是对每一层节点分别注册。

这种设计的本质是：将局部变化收敛为一次整体布局重建。

既保证正确性，又避免重复计算。

### 4\.4\.3 布局变化的向下影响

与向上传播不同，UGUI 并不存在一个显式的“向下递归标记”机制。

但在实际效果上，布局变化会对所有子节点产生影响，这种影响表现为“向下传播”。

其本质过程为：父级 Layout 变化 → RectTransform 更新 → 子节点空间改变 → 顶点数据失效

例如：

- LayoutGroup 重新排列子节点位置

- 父级尺寸变化导致子节点拉伸或压缩

这些变化会直接影响子节点的最终矩形区域，从而使其：

- 需要重新生成 UIVertex

- 触发 Graphic 重建

需要注意的是这种影响不是通过“递归标记子节点 Dirty”实现的，而是在 Rebuild 阶段，由子节点自身检测状态变化触发。

因此可以认为向下传播是一种“结果驱动”的间接影响，而非标记驱动

### 4\.4\.4 标记收敛与统一调度

无论是向上传播还是向下影响，最终都会在调度阶段完成收敛。

在标记阶段，系统只是记录哪些节点需要参与 Layout 重建，哪些节点需要参与 Graphic 重建，而不会立即执行任何计算。

在 CanvasUpdateRegistry 调度阶段，所有标记会被统一整理，按 Layout → Graphic 的顺序执行，完成整棵 UI 树的状态同步。

这种“延迟传播 \+ 批量处理”的机制，使得多次局部变化可以合并为一次全局更新，避免重复执行中间状态计算。

### 4\.4\.5 传播带来的性能问题

脏标记传播机制虽然保证了 UI 状态的一致性，但也可能带来性能问题。

在复杂 UI 层级中，传播往往具有“放大效应”：

- 一个子节点变化 → 向上传播 → 触发整个布局树重建

- 一次布局变化 → 影响所有子节点 → 触发大量 Graphic 重建

常见问题包括：

- 深层嵌套 LayoutGroup 导致大范围重建

- 频繁修改 RectTransform 引发布局链反复计算

- 大规模子节点同时参与顶点重建

这些情况会显著增加：

- CPU 布局计算开销

- Mesh 生成成本

- Canvas 提交压力

因此，在 UI 设计中需要尽量：

- 减少不必要的布局层级嵌套

- 控制 LayoutGroup 的使用范围

- 避免高频触发布局变化

### 小结

脏标记传播路径本质上是一种“基于层级依赖的状态扩散机制”。

其中：

- 向上传播用于解决布局依赖关系

- 向下影响用于保证渲染结果一致性

- 统一调度用于完成最终状态收敛

理解这一传播模型，有助于准确判断 UI 更新范围，并为后续的性能优化提供理论基础。

## 4\.5 CanvasUpdateRegistry 调度机制

在 UGUI 的更新体系中，CanvasUpdateRegistry 是连接“脏标记阶段”与“实际重建阶段”的核心调度中心。它并不负责具体的 UI 计算，而是统一管理所有需要更新的 UI 对象，并在合适的时机按照既定顺序执行更新流程。

从系统架构角度来看，CanvasUpdateRegistry 是整个 UI 更新链路的“调度中枢”。

### 4\.5\.1 调度中心的职责与定位

UGUI 并没有让各个组件在状态变化时立即执行更新，而是将所有更新请求集中交由 CanvasUpdateRegistry 统一处理。

其核心职责包括：

- 收集所有需要更新的 UI 对象

- 按照规则组织更新顺序

- 在统一时机触发各类重建流程

这种设计将“分散更新”转化为“集中调度”，从而显著降低系统开销，并保证执行顺序的稳定性。

### 4\.5\.2 注册机制与更新队列

当 UI 组件调用 SetLayoutDirty 或 SetVerticesDirty 时，并不会立即执行重建逻辑，而是通过注册机制加入更新队列。

CanvasUpdateRegistry 内部维护了两个核心队列：

- Layout 重建队列（m\_LayoutRebuildQueue）

- Graphic 重建队列（m\_GraphicRebuildQueue）

注册过程通常包括：

- 检测当前对象是否已在队列中

- 若未注册，则加入对应队列

- 避免重复添加同一对象

Layout 类型的更新通常通过 LayoutRebuilder 注册，Graphic 类型的更新则由 Graphic 组件直接注册。

这些队列构成了后续调度执行的基础数据结构。

### 4\.5\.3 CanvasUpdate 阶段模型

CanvasUpdateRegistry 的执行并不是一次性完成，而是分多个阶段进行。

这一过程由 CanvasUpdate 枚举驱动，其典型阶段包括：

- Prelayout

- Layout

- PostLayout

- PreRender

- LatePreRender

每一个阶段对应不同类型的更新任务。

例如：

- Layout 相关计算主要发生在 Layout 阶段

- Graphic 顶点重建主要发生在 PreRender 阶段

系统会在每一个阶段遍历对应队列，并调用元素的统一接口：Rebuild\(CanvasUpdate update\)

这一设计使得所有 UI 元素可以通过统一入口参与不同阶段的更新。

### 4\.5\.4 调度执行流程

在 Unity 的帧更新过程中，CanvasUpdateRegistry 会在特定时机触发更新流程。

其典型执行步骤如下：

- 遍历 Layout 队列

- 按层级排序（确保父节点优先）

- 依次执行各阶段 Layout 重建

完成 Layout 后，进入 Graphic 阶段：

- 遍历 Graphic 队列

- 执行顶点重建与材质更新

- 生成最终 Mesh 数据

整个流程严格遵循：Layout → Graphic

这一顺序保证了布局结果能够正确影响后续渲染数据。

### 4\.5\.5 去重机制与更新合并

CanvasUpdateRegistry 内部具有重要的“去重机制”。

当同一个 UI 对象在一帧内被多次标记为 Dirty 时，不会重复加入队列，只会保留一次更新记录。

这一机制带来的直接效果是，多次状态变化被合并为一次重建，避免重复执行 Layout 或 Mesh 计算。

例如：同一帧内多次修改 Text 内容，最终只会触发一次 OnPopulateMesh。

这种“更新合并能力”是 UGUI 性能优化的重要基础。

### 4\.5\.6 顺序依赖与一致性保证

在 UI 更新过程中，不同类型的任务之间存在严格的依赖关系。

最关键的一点是Layout 更新必须先于 Graphic 更新执行

原因在于：

- Layout 决定 RectTransform 的最终结果

- Graphic 依赖 RectTransform 生成顶点数据

如果顺序颠倒，将导致Mesh 数据基于旧布局生成，渲染结果出现错位或错误。

CanvasUpdateRegistry 通过固定执行顺序与阶段划分，保证了这一依赖关系始终成立。

此外，队列排序（如按层级深度排序）也用于确保父节点先更新，子节点在正确的空间基础上计算。

### 小结

CanvasUpdateRegistry 是 UGUI 更新体系中的调度核心。

它通过注册机制收集 UI 更新请求，通过多阶段模型组织执行流程，并通过统一调度完成 Layout 与 Graphic 的批量重建。

其核心价值在于将零散的 UI 更新转化为可控的批处理流程，保证执行顺序与数据一致性，最大限度减少重复计算与性能开销。

理解 CanvasUpdateRegistry 的调度机制，是深入掌握 UGUI 更新流程与性能优化策略的关键。

## 4\.6 Layout 与 Graphic 更新顺序

在 UGUI 的整体更新体系中，Layout 与 Graphic 并不是并列执行的两个系统，而是具有严格先后关系的两级处理流程。

这一顺序的核心原则是：**布局计算必须先于渲染数据生成**

### 4\.6\.1 更新顺序的依赖模型

从系统职责来看：

- Layout 系统负责确定 UI 的空间结构

- Graphic 系统负责生成最终的渲染数据

两者之间存在明确的数据依赖关系：RectTransform（布局结果） → UIVertex（顶点数据）

Graphic 在生成 Mesh 时，必须依赖最终确定的 RectTransform。如果在布局未完成的情况下进行顶点生成，就会导致：

- 顶点坐标基于旧数据

- UI 出现错位或尺寸异常

因此，UGUI 将更新流程设计为先 Layout，后 Graphic。

这一顺序是系统稳定性的基础约束。

### 4\.6\.2 Layout 阶段执行流程

Layout 阶段本身并不是单一过程，而是可以进一步拆分为两个子阶段。

#### 一、Layout Input 收集阶段

在这一阶段，系统会调用：

- CalculateLayoutInputHorizontal

- CalculateLayoutInputVertical

用于收集各个 UI 元素的尺寸需求，例如：

- 最小尺寸

- 首选尺寸

- 弹性尺寸

这些数据不会立即改变布局，而是作为后续计算的输入。

#### 二、Layout Calculation 计算阶段

在收集完所有输入后，系统会执行：

- SetLayoutHorizontal

- SetLayoutVertical

由 LayoutGroup 或相关组件统一计算子节点的位置与尺寸，并更新 RectTransform。

该阶段的输出结果是所有 UI 元素的最终空间结构

这一结果将直接作为 Graphic 阶段的输入。

### 4\.6\.3 Graphic 阶段执行流程

在 Layout 阶段完成之后，系统进入 Graphic 更新阶段。

该阶段的核心任务是根据最新的 RectTransform 生成渲染数据。

对于所有标记为 Vertices Dirty 的 Graphic 组件，系统会执行：

- Graphic\.Rebuild

- OnPopulateMesh

在这一过程中：

- 根据 RectTransform 计算顶点位置

- 生成 UIVertex 数据

- 构建 Mesh 并提交给 CanvasRenderer

需要注意的是Graphic 阶段不参与布局计算，仅依赖 Layout 阶段的结果。

因此，它是一个纯粹的“数据生成阶段”。

### 4\.6\.4 分阶段执行模型（CanvasUpdate）

Layout 与 Graphic 的执行顺序，实际上由 CanvasUpdate 的多阶段模型所驱动。

典型执行顺序如下：

- Prelayout

- Layout

- PostLayout

- PreRender

- LatePreRender

其中：

- Layout 阶段负责完成所有布局计算

- PreRender 阶段负责执行 Graphic 重建

CanvasUpdateRegistry 会在每一个阶段调用ICanvasElement\.Rebuild\(CanvasUpdate update\)

从而让不同系统在正确的阶段参与更新。

这一机制保证了Layout 与 Graphic 在不同阶段执行且顺序始终稳定且可控。

### 4\.6\.5 跳过机制与性能优化

UGUI 的更新流程并不是每一帧都完整执行所有阶段，而是基于 Dirty 标记进行“按需执行”。

例如：如果没有 Layout Dirty，则可以跳过整个 Layout 阶段。如果只有 Vertices Dirty，系统将直接进入 Graphic 重建。

这种机制带来的效果是：

- 避免不必要的布局计算

- 减少 CPU 开销

同样，如果没有任何 Dirty 标记：CanvasUpdateRegistry 将不会执行任何重建逻辑

因此可以认为Layout 与 Graphic 的执行不仅有顺序约束，还具备条件触发特性。

合理利用这一点，可以显著优化 UI 性能。

### 小结

Layout 与 Graphic 的更新顺序，本质上是一种“先结构后渲染”的依赖模型。

其中：

- Layout 系统负责计算 UI 的空间结构

- Graphic 系统基于该结构生成顶点与 Mesh

- CanvasUpdateRegistry 通过分阶段调度保证执行顺序

这一设计既保证了数据一致性，又为按需更新与性能优化提供了基础。

## 4\.7 Rebuild 阶段详解

Rebuild 阶段是 UGUI 更新体系中，将“状态变化”转化为“可渲染数据”的核心执行环节。在此之前，系统仅完成了脏标记的收集与调度排序，而真正的计算与数据构建，全部发生在 Rebuild 阶段。

从系统结构来看，Rebuild 并不是单一过程，而是由多个阶段组成的执行流水线。

### 4\.7\.1 Rebuild 的职责与整体结构

UI 更新流程可以划分为两个部分：

1. 标记与调度阶段 —— 收集变化并组织执行顺序

2. Rebuild 阶段 —— 实际执行计算与数据生成

Rebuild 阶段的核心任务包括：

- 计算 UI 的最终布局结构

- 生成顶点数据与 Mesh

- 同步渲染所需的资源状态

从执行内容上看，Rebuild 可以分为两个主要分支：

1. Layout Rebuild

2. Graphic Rebuild

二者共同完成从“逻辑结构”到“渲染数据”的转换。

### 4\.7\.2 Rebuild 统一入口（ICanvasElement）

在源码层面，所有参与 UI 更新的对象，都会通过统一接口参与 Rebuild 流程：

ICanvasElement\.Rebuild\(CanvasUpdate update\)

CanvasUpdateRegistry 在调度阶段，会在不同的 CanvasUpdate 阶段调用该接口。

不同类型的组件会在不同阶段响应：

- Layout 组件在 Layout 阶段执行逻辑

- Graphic 组件在 PreRender 阶段执行逻辑

这种设计使得所有 UI 元素通过统一入口参与更新，不同系统在不同阶段解耦执行。

Rebuild 不再是分散调用，而是统一调度驱动。

### 4\.7\.3 Layout Rebuild 执行流程

Layout Rebuild 是整个 Rebuild 阶段的第一步。

在这一阶段，系统会处理所有被标记为 Layout Dirty 的对象，并执行完整的布局计算流程。

其执行可以分为两个子过程：

#### 一、布局输入收集

调用以下接口：

- CalculateLayoutInputHorizontal

- CalculateLayoutInputVertical

用于收集每个 UI 元素的尺寸需求，例如最小尺寸、首选尺寸等。

#### 二、布局计算执行

调用：

- SetLayoutHorizontal

- SetLayoutVertical

由 LayoutGroup 或相关组件统一计算子节点位置与尺寸，并更新 RectTransform。

在这一阶段中，不同组件承担不同职责：

- LayoutGroup —— 控制子节点排列

- ContentSizeFitter —— 根据内容调整自身尺寸

- RectTransform —— 根据 Anchor 规则计算最终位置

该阶段的最终结果是所有参与布局的 RectTransform 都达到最终状态。

需要注意的是Layout Rebuild 具有严格的层级依赖，父节点必须先于子节点执行。

因此系统通常会对队列进行排序，确保执行顺序正确。

### 4\.7\.4 Graphic Rebuild 执行流程

在 Layout Rebuild 完成之后，系统进入 Graphic Rebuild 阶段。

该阶段主要负责生成 UI 的渲染数据。

对于所有标记为 Vertices Dirty 的 Graphic 组件，系统会执行：

- Graphic\.Rebuild

- OnPopulateMesh

其核心流程包括：

- 读取最新的 RectTransform

- 计算顶点位置

- 生成 UIVertex 数据（包含位置、UV、颜色等）

- 构建 Mesh

在几何层面，一个矩形通常被拆分为两个三角形构成的四顶点结构。

这些数据最终会提交给 CanvasRenderer，用于 GPU 渲染。

此外，在该阶段中还会处理Material Dirty —— 更新材质与渲染状态。

### 4\.7\.5 执行链路与阶段映射

Rebuild 阶段并不是一次性执行，而是分布在 CanvasUpdate 的多个阶段中。

典型映射关系如下：

- Layout Rebuild —— 发生在 Layout 阶段

- Graphic Rebuild —— 发生在 PreRender 阶段

完整执行链路可以概括为：

CanvasUpdateRegistry 调度

→ 遍历队列

→ 调用 ICanvasElement\.Rebuild

→ 执行具体组件逻辑

这一链路保证了所有 UI 更新按照阶段顺序执行，不同系统之间不会相互干扰。

### 4\.7\.6 性能特征与局部重建机制

Rebuild 阶段是 UGUI 中性能开销最集中的部分。

其主要成本来源包括：

- Layout 计算（层级遍历与尺寸求解）

- 顶点生成（CPU 构建 Mesh 数据）

- 数据上传（提交至 CanvasRenderer）

为了控制开销，UGUI 采用了“局部重建机制”：

- 仅对被标记为 Dirty 的节点进行更新

- 通过传播机制扩展必要范围

- 避免整棵 UI 树的全量重建

此外，CanvasUpdateRegistry 的去重机制保证同一对象在一帧内只会被重建一次。

尽管如此，在复杂 UI 场景中频繁触发 Layout Rebuild或大量 Graphic 同时更新，仍然会带来显著性能压力。

因此，在实际开发中，需要重点控制：

- Rebuild 的触发频率

- Layout 层级复杂度

- 动态 UI 的更新范围

### 小结

Rebuild 阶段是 UGUI 更新体系的核心执行环节。

它通过统一接口驱动，在不同阶段完成 Layout 与 Graphic 的计算，并最终生成可供 GPU 渲染的 Mesh 数据。

从本质上看，Rebuild 是连接 UI 逻辑状态与最终视觉输出的关键桥梁，也是性能优化最需要关注的核心区域。

## 4\.8 Graphic 重建控制与优化机制

在 UGUI 的更新体系中，Rebuild 阶段负责执行实际计算，但在复杂 UI 场景下，仍然可能出现重复重建、无效更新以及重建范围失控的问题。

为了保证性能稳定，UGUI 在多个层面引入了重建控制机制，对 Graphic 的重建过程进行约束与优化。

### 4\.8\.1 重建问题与优化需求

在实际运行过程中，一个 Graphic 组件可能因为多种原因被标记为需要重建，例如：

- 顶点数据变化

- 材质变化

- 父级布局更新

这些变化往往是独立触发的，但最终会汇聚到同一帧的 Rebuild 阶段。

如果缺乏控制机制，就可能出现以下问题：

- 同一对象被多次注册到重建队列

- 同一帧内重复执行 Rebuild

- 布局变化引发大规模连锁更新

这些问题会直接导致 CPU 开销增加，并产生性能抖动。

### 4\.8\.2 重建去重机制

UGUI 的第一层优化来自 CanvasUpdateRegistry。

在注册阶段，系统会对重建队列进行去重处理：

- 同一个 ICanvasElement 只允许在队列中存在一次

- 重复注册请求会被忽略

这一机制保证无论同一帧内触发多少次 Dirty 标记，最终只会执行一次 Rebuild。

因此可以认为，CanvasUpdateRegistry 本身就承担了“重建去重控制器”的角色。

### 4\.8\.3 Graphic 层的自我约束

除了调度层的去重，Graphic 组件自身也具备状态控制能力。

其内部通过标记位控制更新行为：

- m\_VertsDirty

- m\_MaterialDirty

当调用 SetVerticesDirty 时：如果当前已经处于 Dirty 状态，则不会重复触发额外逻辑。

这种设计使得：多次相同的状态修改不会叠加成本，重复更新被收敛为一次有效标记。

从而在源头上减少无效操作。

### 4\.8\.4 Layout 与 Graphic 交界处的控制

在 Layout Rebuild 与 Graphic Rebuild 的交界处，重建控制尤为重要。

典型场景包括：

- Layout 修改 RectTransform

- 子节点 Graphic 需要重新生成顶点

如果没有控制机制：

- 子节点可能被多次标记为 Vertices Dirty

- 重复进入 Graphic 重建流程

UGUI 的处理方式是：

- 通过 Layout 阶段集中更新 RectTransform

- 在后续阶段统一触发 Graphic 重建

- 结合队列去重避免重复执行

这种“阶段隔离 \+ 队列收敛”的方式，本质上实现了对重建传播的控制。

### 4\.8\.5 重建风暴与性能控制策略

在复杂 UI 结构中，最常见的问题是“重建风暴”。

例如：

- ScrollRect 列表刷新

- 大量子节点同时变化

- 嵌套 LayoutGroup 触发级联计算

在这些场景下，虽然系统具备去重机制，但仍然可能出现：

- 大量对象同时参与 Rebuild

- 单帧 CPU 峰值过高

因此，在工程实践中，需要进一步进行优化控制：

- 减少深层 Layout 嵌套

- 避免频繁修改 RectTransform

- 合并 UI 状态更新操作

- 控制动态元素数量

这些措施可以从源头减少 Dirty 标记数量，从而降低 Rebuild 压力。

### 小结

UGUI 并没有依赖单一模块来控制 Graphic 重建，而是通过多层机制共同完成优化：

- CanvasUpdateRegistry 负责队列去重与统一调度

- Graphic 组件负责自身状态收敛

- 分阶段执行机制负责隔离更新影响

这些机制共同构成了一套“重建控制体系”。

其核心目标在于：

- 减少重复执行

- 控制更新范围

- 保证性能稳定

理解这一体系，有助于在复杂 UI 场景中合理控制更新粒度，从而实现高效的界面渲染。

## 本章小结

本章围绕 UGUI 的“更新与重建系统”展开，从整体生命周期出发，逐步深入到脏标记、调度机制以及最终的 Rebuild 执行流程，完整构建了 UI 从状态变化到渲染输出的内部工作链路。

在 UI 更新生命周期中，UGUI 并不会在状态变化时立即执行计算，而是采用“延迟处理”的设计，将变化收集并统一在特定阶段执行。这一机制将零散的更新操作转化为可控的批处理流程，是整个系统性能优化的基础。

Dirty 标记机制作为更新体系的核心入口，对 UI 的变化进行分类管理。通过 Layout Dirty、Vertices Dirty 与 Material Dirty 的划分，系统能够精确控制更新范围，并为后续调度提供依据。

在具体实现层面，SetLayoutDirty 与 SetVerticesDirty 分别连接 Layout 系统与 Graphic 系统，将不同类型的变化引导至对应的处理流程。二者不仅职责清晰，同时也构成了 UI 更新链路的关键触发点。

脏标记在层级结构中的传播，使得 UI 系统能够自动处理父子节点之间的依赖关系。其中，Layout 的向上传播保证布局一致性，而布局变化对 Graphic 的间接影响，则保证最终渲染结果的正确性。这种传播机制虽然提高了系统的健壮性，但也带来了潜在的性能开销。

CanvasUpdateRegistry 作为调度核心，将所有更新请求集中管理，并通过分阶段执行模型组织更新顺序。其内部的队列管理与去重机制，使得同一帧内的多次修改可以被合并处理，从而避免重复计算。

在执行顺序上，UGUI 采用“先 Layout，后 Graphic”的严格依赖模型。Layout 阶段负责确定 RectTransform 的最终结果，Graphic 阶段则基于该结果生成顶点数据与 Mesh。这一顺序保证了数据的一致性，也是 UI 正确渲染的前提。

Rebuild 阶段作为最终执行环节，将前述所有机制收敛为实际计算过程。Layout Rebuild 完成空间结构求解，Graphic Rebuild 完成渲染数据构建，二者共同实现从逻辑状态到 GPU 可渲染数据的转换。

在性能控制方面，UGUI 通过队列去重、状态收敛以及分阶段执行等机制，尽可能减少重复重建与无效计算。同时，局部重建策略确保系统仅处理必要的 UI 节点，从而降低整体开销。

总体来看，UGUI 的更新系统可以抽象为一个“状态收集、分类标记、统一调度、分阶段执行”的处理模型。该模型在保证 UI 一致性的同时，实现了较高的运行效率。

理解这一整套更新与重建机制，是进行 UI 性能优化与复杂界面设计的基础，也为后续深入分析渲染细节与底层实现奠定了理论基础。

