---
title: 第5章 Graphic 系统
categories:
  - Unity
  - UGUI
tags:
  - UGUI
  - Unity
cover: /images/UGUI/UGUI.png
abbrlink: fcbee22c
date: 2026-08-26 00:00:05
---
# 第5章 Graphic 系统

在上一章中，我们已经完整分析了 UGUI 的更新与重建机制，理解了 UI 状态如何通过 Dirty 标记、调度系统以及 Rebuild 阶段被逐步处理。然而，这一流程最终只是完成了“何时更新”的问题，而真正决定 UI 呈现效果的，是 Graphic 系统。

在 UGUI 的整体架构中，Graphic 系统处于承上启下的关键位置。它上承 RectTransform 与 Layout 系统所生成的矩形空间结果，下接 Mesh 构建与 CanvasRenderer 的渲染提交流程，是 UI 从“结构描述”走向“图形数据”的核心转换层。

如果说 RectTransform 解决的是“UI 在哪里、占多大空间”的问题，那么 Graphic 系统解决的就是“UI 长什么样”的问题。它的核心职责，是将抽象的矩形区域转化为具体可渲染的几何数据，并通过统一接口输出给渲染管线。

从渲染链路的角度来看，UGUI 的数据流可以简化为：

RectTransform（空间结构）

→ Graphic（几何生成）

→ CanvasRenderer（提交与合批）

→ GPU（最终绘制）

在这一链路中，Graphic 正是连接“逻辑结构”与“渲染数据”的关键节点。

从设计上来看，Graphic 并不是一个单一的绘制模块，而是一个抽象基类体系。Image、RawImage、Text 等常见 UI 组件，都是基于 Graphic 扩展实现的具体表现形式。不同类型的 UI 元素，本质上都是通过重写同一套 Mesh 生成流程，来实现不同的视觉表现。

Graphic 系统的核心工作流程可以概括为三个阶段。

第一阶段是数据接收阶段。Graphic 会接收来自 RectTransform 与 Layout 系统的更新结果，包括尺寸、位置以及父子层级关系等信息。

第二阶段是 Mesh 构建阶段。通过 OnPopulateMesh 接口，将矩形空间转换为顶点数据，并构建对应的三角形结构。

第三阶段是渲染提交阶段。生成的 Mesh 数据会被提交给 CanvasRenderer，由其负责批处理与最终绘制。

此外，Graphic 系统还承担着 UI 状态变化响应的职责。当颜色、材质、纹理或顶点数据发生变化时，Graphic 会通过 Dirty 标记机制参与更新流程，并在 Rebuild 阶段重新生成渲染数据。

从整体架构来看，Graphic 系统本质上是 UI 渲染管线中的“几何生成层”。它将复杂的 UI 结构统一转换为标准 Mesh 数据，从而为 GPU 提供一致的输入格式。

本章将从 Graphic 类结构入手，逐步分析 Image、RawImage 与 Text 的实现差异，深入解析 OnPopulateMesh 与 VertexHelper 的内部机制，并进一步探讨自定义 Graphic 的扩展方式，从而完整理解 UGUI 的图形生成体系。

## 5\.1 Graphic 类结构

Graphic 是 UGUI 中所有可渲染 UI 组件的抽象基类，它定义了一套统一的“从 UI 数据到 Mesh 数据”的生成规范。无论是 Image、RawImage 还是 Text，本质上都是在 Graphic 提供的框架下完成各自的顶点构建逻辑。

从系统职责来看，Graphic 并不直接参与渲染，而是专注于生成渲染所需的数据，是 UI 渲染体系中的核心中间层。

### 5\.1\.1 类层级与接口实现

在类结构上，Graphic 继承自 UIBehaviour，这意味着它具备完整的 Unity 组件生命周期能力，例如：

- OnEnable

- OnDisable

- OnDestroy

同时，Graphic 实现了 ICanvasElement 接口，从而能够接入 CanvasUpdateRegistry 的调度系统。

该接口的核心方法为：Rebuild\(CanvasUpdate update\)

在 UI 更新过程中，CanvasUpdateRegistry 会在不同阶段调用该方法，从而驱动 Graphic 执行对应的重建逻辑。

通过这一接口，Graphic 成为 UI 更新体系中的标准参与者。

### 5\.1\.2 核心成员与状态数据

Graphic 内部维护了一组关键状态，用于描述当前 UI 的渲染属性与更新状态。

其中最重要的包括：

- m\_Color

用于控制 UI 的基础颜色，会影响最终顶点颜色

- m\_Material

用于指定渲染材质，影响 Shader 与合批行为

- m\_RaycastTarget

用于控制是否参与 UI 事件射线检测

- m\_VertsDirty 与 m\_MaterialDirty

用于标记当前对象是否需要进行顶点或材质重建

此外，Graphic 还会缓存对 CanvasRenderer 的引用，用于后续提交渲染数据。

这些状态共同决定了当前 UI 是否需要更新，更新时需要执行哪一类重建流程。

### 5\.1\.3 重建触发机制

Graphic 通过 Dirty 标记机制参与 UI 更新流程。

常见触发入口包括：

- SetVerticesDirty

- SetMaterialDirty

这些方法的作用是标记当前状态为需要更新，将对象注册到 CanvasUpdateRegistry。

需要强调的是这些方法不会立即执行计算，仅负责“上报更新请求”。

真正的执行发生在后续的 Rebuild 阶段。

### 5\.1\.4 Mesh 生成流程

Graphic 的核心功能是生成 Mesh 数据，其入口函数为：OnPopulateMesh

在 Rebuild 阶段中，系统会调用：Graphic\.Rebuild → OnPopulateMesh

在该过程中，Graphic 会：

- 读取当前 RectTransform 的矩形信息

- 根据自身逻辑生成顶点数据

- 通过 VertexHelper 构建三角形结构

不同 UI 组件的差异，正体现在 OnPopulateMesh 的实现方式上：

- Image 生成标准矩形并映射 Sprite UV

- Text 按字符生成顶点并排版

因此可以认为 OnPopulateMesh 是 UI 渲染差异化的核心来源

### 5\.1\.5 与 CanvasRenderer 的关系

Graphic 与 CanvasRenderer 之间是典型的“生产者与消费者”关系。

Graphic 的职责是生成 Mesh 数据，准备材质与纹理信息。

CanvasRenderer 的职责是接收 Mesh 数据，执行批处理与渲染提交。

Graphic 不直接参与绘制，也不控制渲染顺序。所有绘制行为最终由 CanvasRenderer 完成。

这种设计实现了数据生成与渲染执行的解耦

使得 UI 系统可以独立优化几何生成与渲染流程。

### 5\.1\.6 设计特点与架构定位

从整体架构来看，Graphic 可以被视为 UGUI 渲染体系中的“几何生成层”。

其核心特点包括：

- 统一接口

所有 UI 元素通过 OnPopulateMesh 生成数据

- 职责分离

不参与布局计算，也不参与渲染执行

- 状态驱动

通过 Dirty 标记控制更新流程

- 数据抽象

将 UI 元素统一为 Mesh 数据模型

通过这些设计，UGUI 将复杂的 UI 表现统一抽象为标准几何数据，从而为后续 CanvasRenderer 与 GPU 渲染提供稳定输入。

### 小结

Graphic 类结构构建了 UGUI 图形系统的基础框架。

它通过统一的接口与生命周期管理，将 UI 元素的视觉表现抽象为 Mesh 数据生成过程，并通过与 CanvasRenderer 的解耦设计，实现了灵活且高效的渲染体系。

## 5\.2 Image、RawImage、Text 对比

在 Graphic 体系中，Image、RawImage 与 Text 是最典型的三种实现。它们共享同一基类结构，但在数据来源、Mesh 构建方式以及性能特征上存在明显差异。

理解它们之间的区别，有助于从本质上把握 UGUI 的渲染组织方式。

### 5\.2\.1 统一渲染模型

从系统结构来看，这三种组件具有完全一致的渲染路径。

它们都继承自 Graphic，并通过统一流程完成数据生成：

- 接收 RectTransform 结果

- 生成顶点数据

- 构建 Mesh

- 提交至 CanvasRenderer

因此，它们的共同点可以概括为：

- 基于 Mesh 的渲染模型

- 通过 OnPopulateMesh 生成几何数据

- 最终进入统一渲染管线

差异并不在“如何渲染”，而在“如何生成 Mesh”。

### 5\.2\.2 Image 实现特征

Image 是 UGUI 中最常用的图形组件，其核心数据来源是 Sprite。

Sprite 本质上是纹理图集中的一个区域，包含：

- UV 坐标

- 边界信息

- 像素数据

在 Mesh 构建过程中，Image 会执行如下逻辑：

- 根据 RectTransform 生成矩形顶点

- 将 Sprite 的 UV 映射到顶点

- 构建标准四边形

在特定模式下，例如：

- Sliced

- Tiled

- Filled

Image 还会进一步调整顶点结构，例如九宫格拆分或扇形填充。

Image 的主要特点是：

- 结构简单

- 计算成本低

- 高度适配图集系统

因此，在绝大多数 UI 场景中，Image 具有最佳的性能表现。

### 5\.2\.3 RawImage 实现特征

RawImage 与 Image 的核心区别在于数据来源。

RawImage 直接使用 Texture，而不是 Sprite。

在 Mesh 构建过程中：

- 同样生成标准矩形

- 使用完整纹理或自定义 UV 区域

由于不依赖图集系统，RawImage 更适合以下场景：

- RenderTexture 显示

- 视频播放

- 动态生成纹理

但这种灵活性带来的代价是：

- 难以参与图集合批

- 材质与纹理切换成本较高

因此，在大规模 UI 中，RawImage 的性能通常弱于 Image。

### 5\.2\.4 Text 实现特征

Text 是三者中最复杂的实现，其核心差异在于 Mesh 的生成方式。

与 Image 的“单矩形模型”不同，Text 采用“逐字符生成”的策略。

在 OnPopulateMesh 阶段，系统会执行：

- 解析字符串

- 获取字体字形信息

- 逐字符生成四边形

- 计算 UV 与顶点位置

同时，还需要处理：

- 换行规则

- 对齐方式

- 字符间距与行距

因此，Text 的 Mesh 规模与字符数量直接相关。

一个包含 N 个字符的文本，其顶点数量通常为：4 × N

这一特性使得 Text 在以下场景中开销较高：

- 长文本

- 频繁更新文本

- 动态排版变化

此外，Text 同时承担：

- 布局计算

- 几何生成

这使其在系统层面具有更高复杂度。

### 5\.2\.5 三者核心差异总结

从本质上看，三者的差异可以归结为“Mesh 生成来源不同”。

- Image 基于 Sprite 区域映射生成矩形

- RawImage 基于原始纹理生成矩形

- Text 基于字符解析生成多矩形结构

进一步从性能与特性角度对比：

- Image

    - 最易合批

    - 性能最优

    - 适用于大多数静态 UI

- RawImage

    - 灵活性高

    - 合批能力弱

    - 适用于动态纹理场景

- Text

    - 功能最复杂

    - 生成成本最高

    - 适用于文本显示与排版

从系统设计角度来看，这三种组件体现了 Graphic 的核心思想：统一接口，差异实现

即通过同一套 Mesh 生成框架，支持多种不同的数据来源与表现形式。

### 小结

Image、RawImage 与 Text 的差异，本质上来源于它们各自的 Mesh 构建策略。

这种差异不仅决定了它们的视觉表现方式，也直接影响性能特征与适用场景。

理解这一点，是深入掌握 UGUI 渲染体系与优化策略的重要基础。

## 5\.3 OnPopulateMesh 机制

OnPopulateMesh 是 Graphic 系统中最核心的扩展入口，它决定了 UI 元素如何从“矩形空间描述”转化为“可渲染 Mesh 数据”。无论是 Image、Text 还是自定义 UI 组件，其几何结构生成都通过这一机制完成。

从系统定位来看，它是 UI 渲染管线中的几何生成接口。

### 5\.3\.1 机制定位与职责

Graphic 负责管理状态与调度更新，但并不关心具体几何如何生成。OnPopulateMesh 的职责正是填补这一空缺。

其核心作用可以概括为：根据 RectTransform 提供的矩形信息生成顶点数据与三角形结构，然后输出标准 Mesh 数据。

因此可以认为Graphic 定义“何时生成”，OnPopulateMesh 定义“生成什么”。

这一职责划分是 UGUI 可扩展性的基础。

### 5\.3\.2 调用链路分析

OnPopulateMesh 并不是独立调用的，而是处于完整的重建链路之中。

其典型调用路径如下：

Graphic\.Rebuild

→ UpdateGeometry

→ OnPopulateMesh

具体流程为：

当对象被标记为 Vertices Dirty，进入 CanvasUpdateRegistry 调度。

在 Graphic 阶段调用 Rebuild，最终触发 OnPopulateMesh。

这一链路说明OnPopulateMesh 只在“需要重建”时执行，不会在每一帧被调用。

其执行频率完全受 Dirty 标记控制。

### 5\.3\.3 顶点构建流程

在执行 OnPopulateMesh 时，系统会传入一个 VertexHelper 实例，用于构建顶点数据。

基本流程如下：

- 清空 VertexHelper 内部数据

- 根据 RectTransform 计算顶点位置

- 填充 UIVertex 数据

- 构建三角形索引

开发者并不直接操作 Mesh，而是通过 VertexHelper 提供的接口完成数据构建。

例如：

- AddVert 添加顶点

- AddTriangle 构建三角形

- AddUIVertexQuad 快速添加四边形

在 OnPopulateMesh 结束后，系统会将 VertexHelper 转换为 Mesh，并提交给 CanvasRenderer。

### 5\.3\.4 不同组件实现差异

不同 Graphic 子类的差异，主要体现在 OnPopulateMesh 的实现方式。

- Image

    - 生成单个矩形

    - 映射 Sprite UV

    - 支持九宫格与填充模式

- RawImage

    - 生成单个矩形

    - 使用完整纹理或自定义 UV

- Text

- 按字符生成多个矩形

- 处理排版与字形信息

- 顶点数量随字符数增长

因此可以认为 OnPopulateMesh 是 UI 表现差异的核心来源

### 5\.3\.5 性能特征与优化要点

OnPopulateMesh 是 UI 重建中最主要的 CPU 开销来源之一。

其成本主要来自：

- 顶点数据构建

- UV 计算

- 字符串解析（Text）

在高频更新场景中，可能产生明显性能压力。

常见优化方向包括：

- 减少不必要的 SetVerticesDirty 调用

- 避免在 Update 中频繁修改 UI

- 控制 Text 长度与更新频率

由于 OnPopulateMesh 仅在 Dirty 时触发，因此降低 Dirty 频率即等价于降低 Mesh 重建成本。

### 5\.3\.6 自定义实现注意事项

在自定义 Graphic 时，OnPopulateMesh 是最核心的扩展点。

常见应用包括：

- 圆形进度条

- 多边形 UI

- 渐变效果

- 特殊遮罩

在实现时需要注意：

- 始终基于 RectTransform 计算顶点

- 避免生成非法三角形结构

- 控制顶点数量，防止性能问题

- 正确设置 UV 与颜色信息

此外，应避免在 OnPopulateMesh 中执行复杂逻辑，例如：

- 资源加载

- 字符串处理

- 频繁分配内存

这些操作会直接影响 UI 重建性能。

### 小结

OnPopulateMesh 是 UGUI 图形系统的核心扩展接口。

它位于 Rebuild 阶段的关键位置，将 RectTransform 提供的空间信息转化为具体的 Mesh 数据，并定义了 UI 的最终几何形态。

理解该机制，不仅有助于掌握 UGUI 的渲染流程，也为实现自定义 UI 提供了基础能力。

## 5\.4 VertexHelper 详解

VertexHelper 是 UGUI 中用于构建 Mesh 数据的核心辅助结构，它位于 Graphic 与 Unity Mesh 之间，承担着“顶点数据组织与生成”的中间层职责。

其设计目标，是为 OnPopulateMesh 提供一个稳定且高层的顶点构建接口，从而屏蔽底层 Mesh 操作的复杂性。

### 5\.4\.1 设计目标与定位

在早期实现中，UI 组件需要直接操作 Mesh 的各类数组，包括：

- vertices

- uv

- colors

- triangles

这种方式不仅容易出错，而且维护成本较高。

VertexHelper 的引入，将这些操作统一封装为语义化接口，使顶点构建逻辑从“底层数组操作”转变为“结构化数据构建”。

因此可以认为 VertexHelper 是 UI Mesh 构建的抽象层

### 5\.4\.2 内部数据结构

VertexHelper 内部维护了一组顶点相关的数据列表，用于存储完整的 UI 几何信息。

核心数据包括：

- 顶点位置

- 颜色信息

- UV 坐标

- 法线与切线

- 三角形索引

这些数据以 UIVertex 为基本单位进行组织。

UIVertex 可以理解为 UI 系统中的“标准顶点结构”，其中包含：

- position

- color

- uv0

- normal

- tangent

在 OnPopulateMesh 阶段，所有顶点数据都会被转换为 UIVertex，并存入 VertexHelper。

### 5\.4\.3 顶点构建接口

VertexHelper 提供了一组高层接口，用于构建几何结构。

常用方法包括：

- AddVert

用于添加单个顶点

- AddTriangle

用于构建三角形索引

- AddUIVertexQuad

用于快速添加一个矩形

通过这些接口，可以方便地构建任意形状的 UI Mesh。

例如，一个标准矩形通常包含：

- 4 个顶点

- 2 个三角形

整个过程无需手动管理数组索引，只需按顺序调用接口即可完成。

### 5\.4\.4 数据提交流程

VertexHelper 本身并不是最终数据结构，它只是一个构建过程中的中间容器。

在 OnPopulateMesh 执行结束后，数据会进入提交流程：

- VertexHelper 内部数据被转换为 Mesh

- 调用 FillMesh 或等效方法写入 Mesh

- Mesh 被提交给 CanvasRenderer

这一流程完成了从“顶点构建”到“可渲染数据”的转换。

需要注意的是VertexHelper 不直接参与渲染，也不控制绘制行为。

它的职责仅限于数据组织与生成。

### 5\.4\.5 生命周期与性能特征

VertexHelper 的生命周期与一次 Mesh 重建过程一致。

在 Graphic Rebuild 阶段：

- 创建或复用 VertexHelper

- 填充顶点数据

- 提交至 Mesh

- 随后释放或进入复用池

这种短生命周期设计，使其更适合作为“临时缓冲结构”。

从性能角度来看，其优势体现在：

- 减少直接操作 Mesh 带来的开销

- 避免频繁分配顶点数组

- 降低 GC 压力

通过集中管理顶点数据，VertexHelper 提升了 UI 重建的整体效率。

### 5\.4\.6 设计意义与系统分层

VertexHelper 的引入，使 UGUI 的渲染体系进一步清晰分层：

- RectTransform 负责空间计算

- Graphic 负责逻辑组织与调度

- VertexHelper 负责顶点数据构建

- CanvasRenderer 负责渲染提交

这种分层设计带来了：

- 职责清晰

- 扩展性强

- 维护成本低

同时也使得开发者可以专注于几何生成逻辑，而无需关注底层 Mesh 实现细节。

### 小结

VertexHelper 是 UGUI 中连接 Graphic 与 Mesh 的关键桥梁。

它通过封装底层数据结构，提供统一的顶点构建接口，使复杂的 Mesh 生成过程变得可控且清晰。

理解 VertexHelper 的工作方式，是掌握 OnPopulateMesh 机制以及实现自定义 UI 的重要基础。

## 5\.5 材质与纹理绑定机制

在 UGUI 的 Graphic 系统中，Mesh 的生成只完成了几何数据构建，而最终视觉效果由材质与纹理共同决定。这一机制负责将 CPU 侧生成的数据与 GPU 渲染状态连接起来，是 UI 进入渲染管线的关键环节。

### 5\.5\.1 渲染资源组成

一个完整的 UI 渲染单元由三部分组成：

- Mesh

- Material

- Texture

其中：

- Mesh 定义几何结构

- Material 定义 Shader 与渲染方式

- Texture 提供像素内容

三者共同决定 UI 的最终显示结果。

### 5\.5\.2 Graphic 中的材质与纹理来源

在 Graphic 中，材质与纹理由两个核心属性管理：

- material

- mainTexture

material 用于定义渲染逻辑，通常来源于：

- 默认 UI 材质

- 组件自定义材质

mainTexture 用于提供图像数据，其来源根据组件类型不同而变化：

- Image 来自 Sprite 所引用的 Texture

- RawImage 直接使用 Texture

- Text 使用 Font 生成的字体图集

需要注意的是 Graphic 并不直接存储所有渲染资源，而是通过接口动态提供当前状态。

### 5\.5\.3 材质提交与 CanvasRenderer

在 Rebuild 阶段完成后，Graphic 会将渲染数据提交给 CanvasRenderer。

核心流程包括：

- 设置 Mesh 数据

- 设置 Material

- 设置 Texture

CanvasRenderer 内部接口类似：

- SetMaterial

- SetTexture

通过这些接口，Graphic 将 CPU 构建的数据传递到渲染系统中。

需要强调的是 CanvasRenderer 才是实际与渲染管线交互的对象，Graphic 仅负责数据准备。

### 5\.5\.4 materialForRendering 与材质实例化

在实际渲染中，Graphic 并不会直接使用 material，而是通过 materialForRendering 获取最终材质。

该过程可能包含以下处理：

- Mask 相关修改

- Stencil 参数写入

- 材质实例化

当 UI 存在以下情况时：

- 使用 Mask

- 修改材质属性

系统可能会生成新的材质实例，以保证不同 UI 元素之间互不影响。

这种机制保证了：

- 功能正确性

- 渲染状态隔离

但同时也会带来：

- 额外内存开销

- 潜在的 DrawCall 增加

### 5\.5\.5 合批与状态一致性

在 UGUI 中，是否能够合批，取决于渲染状态是否一致。

关键因素包括：

- Material 是否相同

- Texture 是否相同

- Shader 参数是否一致

只有在完全一致的情况下，多个 Graphic 才能合并为一个 DrawCall。

例如：

- 多个 Image 使用同一图集

- 共享同一材质

则可以高效合批。

而以下情况会打断合批：

- 不同材质

- 不同纹理

- 材质实例不同

因此，材质与纹理的管理直接影响 UI 性能。

### 5\.5\.6 Mask 与 Stencil 对材质的影响

在 UGUI 中，Mask 系统通过 Stencil Buffer 实现裁剪效果。

当 UI 使用 Mask 时，系统会修改材质的 Stencil 参数，生成带有裁剪逻辑的材质变体。

这一过程通常发生在 materialForRendering 阶段。

结果导致相同原始材质的 UI 在 Mask 环境下可能变为不同材质实例，从而影响合批效果。

因此，在复杂 UI 中 Mask 的使用需要权衡功能与性能。

### 5\.5\.7 设计意义与性能特征

从系统设计角度来看，材质与纹理绑定机制构成了 UGUI 的“资源绑定层”。

其核心作用是将 Mesh 数据与 GPU 状态绑定，统一管理渲染资源，控制批处理行为。

这一机制的设计特点包括：

- 数据与渲染分离

- 动态材质生成

- 状态驱动合批

在性能层面：

- 合理使用图集可以减少 Texture 切换

- 共享材质可以提高批处理效率

- 避免不必要的材质实例可以降低 DrawCall

### 小结

材质与纹理绑定机制决定了 UI 的最终视觉表现与渲染效率。

- Material 决定渲染方式

- Texture 决定显示内容

- CanvasRenderer 负责提交执行

三者与 Mesh 共同构成完整的 UI 渲染单元。

理解这一机制，是分析 UI 合批、Mask 行为以及渲染性能问题的关键基础。

## 5\.6 自定义 Graphic

在 UGUI 的设计中，Graphic 体系提供了高度可扩展的渲染能力。开发者可以通过继承 Graphic，自定义 UI 的几何生成逻辑，从而实现任意形状的界面元素。

这种扩展方式并不会破坏现有渲染管线，而是通过替换 Mesh 生成规则，实现对 UI 表现的完全控制。

### 5\.6\.1 实现目标

本节通过一个基础示例，构建一个圆形 UI 元素。其目标包括：

- 根据 RectTransform 生成圆形顶点

- 使用三角扇结构构建 Mesh

- 支持颜色与透明度控制

- 可直接作为标准 UI 组件使用

该示例覆盖自定义 Graphic 的完整流程。

### 5\.6\.2 示例代码

```C#
using UnityEngine;
using UnityEngine.UI;

public class CircleGraphic : Graphic
{
    [Range(3, 256)]
    public int segments = 64;

    protected override void OnPopulateMesh(VertexHelper vh)
    {
        vh.Clear();

        Rect rect = GetPixelAdjustedRect();

        Vector2 center = rect.center;
        float radius = Mathf.Min(rect.width, rect.height) * 0.5f;

        float angleStep = 2 * Mathf.PI / segments;

        UIVertex vertex = UIVertex.simpleVert;
        vertex.color = color;
        vertex.position = center;
        vh.AddVert(vertex);

        for (int i = 0; i <= segments; i++)
        {
            float angle = i * angleStep;

            float x = center.x + Mathf.Cos(angle) * radius;
            float y = center.y + Mathf.Sin(angle) * radius;

            vertex = UIVertex.simpleVert;
            vertex.color = color;
            vertex.position = new Vector2(x, y);

            vh.AddVert(vertex);
        }

        for (int i = 1; i <= segments; i++)
        {
            vh.AddTriangle(0, i, i + 1);
        }
    }
}
```

### 5\.6\.3 几何生成流程解析

该示例的核心在于 OnPopulateMesh 方法。

首先，通过 GetPixelAdjustedRect 获取当前 UI 的有效矩形区域，用于确保像素对齐。

随后计算圆心与半径。半径取宽高最小值的一半，从而保证生成结果始终为正圆。

在顶点构建阶段：先添加一个中心点，再按角度均匀生成外圈顶点。外圈顶点通过三角函数计算位置，从而形成圆周分布。

在索引构建阶段：以中心点为起点，逐个连接外圈顶点，构建三角扇结构，最终形成完整的圆形 Mesh。

### 5\.6\.4 运行时行为分析

该组件在运行时的行为与 Image 类似：

- 参与 CanvasUpdateRegistry 调度

- 在 Rebuild 阶段生成 Mesh

- 由 CanvasRenderer 提交渲染

但其核心差异在于不依赖 Sprite 完全由 OnPopulateMesh 控制几何结构。

当 RectTransform 和 color 发生变化时系统会触发 SetVerticesDirty，从而重新生成 Mesh。

在渲染层面，只要材质一致，该组件可以与其他 UI 元素合批。

### 5\.6\.5 自定义 Graphic 关键要点

在实现自定义 Graphic 时，需要关注以下几个关键点：

1. 始终基于 RectTransform 进行计算，避免使用世界坐标或屏幕坐标直接构建顶点。

2. 保证顶点顺序与三角形索引正确，错误的索引会导致渲染异常。

3. 合理控制顶点数量，过高的 segments 会显著增加 CPU 与 GPU 开销。

4. 避免在 OnPopulateMesh 中执行复杂逻辑，例如字符串处理或资源加载。

5. 正确使用 color，确保与 UI 统一的颜色控制体系兼容。

这些约束直接影响 UI 的性能与稳定性。

### 5\.6\.6 扩展能力与应用场景

通过自定义 Graphic，可以实现远超默认组件能力的 UI 表现，例如：

- 圆形进度条

- 雷达图

- 波形图

- 动态遮罩

- 数据可视化图形

这些效果本质上都是通过自定义 Mesh 生成规则实现的。

UGUI 提供的这一扩展机制，使其不仅是 UI 系统，同时也具备一定的二维图形生成能力。

### 小结

自定义 Graphic 的本质，是在 UGUI 渲染体系中替换几何生成逻辑。

通过 OnPopulateMesh，开发者可以完全控制 UI 的顶点结构，而无需修改底层渲染系统。

这一机制使 UGUI 在保持统一渲染管线的前提下，具备高度灵活的扩展能力。

## 本章小结

本章围绕 UGUI 的 Graphic 系统展开，从类结构到具体实现，再到渲染资源绑定与自定义扩展，系统性分析了 UI 从“几何生成”到“渲染提交”的完整过程。

在整体架构中，Graphic 位于 UI 渲染管线的核心位置。它承接 RectTransform 与 Layout 系统的空间计算结果，并将其转化为 Mesh 数据，最终交由 CanvasRenderer 提交至渲染管线。通过这种设计，UGUI 将“结构描述”与“渲染数据”有效分离，构建出清晰的分层体系。

在类结构层面，Graphic 通过继承 UIBehaviour 并实现 ICanvasElement 接口，接入了完整的生命周期与更新调度机制。同时，通过 Dirty 标记与 Rebuild 流程，控制 Mesh 的生成时机，从而避免不必要的重复计算。

在具体实现上，Image、RawImage 与 Text 展现了 Graphic 体系的三种典型扩展方式。它们共享统一的渲染流程，但在 Mesh 生成策略上存在本质差异。Image 基于 Sprite 映射生成矩形结构，RawImage 直接输出纹理内容，而 Text 则通过字符解析构建复杂的多顶点结构。这种差异不仅决定了视觉表现形式，也直接影响性能特征与合批行为。

OnPopulateMesh 作为 Graphic 的核心扩展接口，定义了 UI 几何生成的具体方式。所有可渲染 UI 元素，最终都通过该方法生成顶点数据。VertexHelper 则作为中间数据结构，封装了 Mesh 构建过程，使顶点生成从底层数组操作提升为结构化构建流程，从而降低复杂度并提升可维护性。

在渲染资源层面，材质与纹理绑定机制将 Mesh 数据与 GPU 渲染状态连接起来。Material 决定渲染方式，Texture 提供视觉内容，而 CanvasRenderer 负责最终提交。这一机制不仅影响 UI 的视觉表现，也直接决定合批效率与渲染性能。

在扩展能力方面，自定义 Graphic 提供了极高的灵活性。通过重写 OnPopulateMesh，开发者可以实现任意形状的 UI 元素，而无需修改底层系统。这种“统一接口、自由实现”的设计，使 UGUI 既具备稳定的渲染管线，又支持复杂的个性化表现。

总体来看，Graphic 系统可以抽象为一个“几何生成与资源绑定”的中间层。它通过统一接口连接 UI 结构与渲染管线，将所有 UI 元素转化为标准 Mesh 数据，并通过材质与纹理完成最终呈现。

理解 Graphic 系统的工作原理，不仅有助于掌握 UGUI 的渲染机制，也为 UI 性能优化与自定义效果实现提供了基础支撑。

