---
title: 第9章 UI 与渲染管线
categories:
  - Unity
  - UGUI
tags:
  - UGUI
  - Unity
cover: /images/UGUI/UGUI.png
abbrlink: 1dacfce4
date: 2026-08-26 00:00:09
---
# 第9章 UI 与渲染管线

UGUI 的渲染过程并不是独立存在的，它始终运行在 Unity 整体渲染管线之上。

很多开发者会将 UI 理解为“直接绘制到屏幕上的二维元素”，但从底层实现来看，UI 本质上仍然属于渲染系统的一部分。无论是 Image、Text 还是复杂的动态界面，最终都需要进入 Render Pipeline，由 GPU 完成真正的绘制。

因此，UI 的显示顺序、遮挡关系、透明混合以及性能开销，本质上都与渲染管线密切相关。

从执行流程来看，UGUI 的渲染路径大致如下：

UI 顶点生成 → Canvas Batch 构建 → CanvasRenderer 提交 → RenderPipeline 执行 → GPU DrawCall 输出

在这个过程中：

- Graphic 负责生成顶点数据 

- Canvas 负责批处理与合并 

- CanvasRenderer 负责向渲染系统提交绘制命令 

- RenderPipeline 负责决定真正的绘制阶段 

- GPU 最终完成像素输出 

因此，UI 并不是一个脱离渲染体系的“特殊系统”，而是 Unity 图形渲染架构中的一个组成部分。

在 Built\-in Render Pipeline 中，UI 通常会在场景渲染结束后进行绘制，以保证其覆盖在三维场景之上。而在 Scriptable Render Pipeline（SRP）体系下，UI 的绘制阶段则由 RenderPass 控制，其行为更加显式，也更加可编程。

Canvas 的三种渲染模式，本质上也是 UI 接入渲染管线的不同方式。

Overlay 模式虽然不依赖 Camera，但仍然会进入最终渲染流程；Camera 模式会参与摄像机排序与渲染阶段；World Space 模式则完全融入三维场景体系。

除此之外，UI 的最终绘制结果还会受到 RenderQueue、Sorting Layer、Depth 以及材质状态等多个系统共同影响。

在 URP 等现代渲染管线中，UI 还涉及 SRP Batcher 的兼容问题。Shader、材质状态以及 Pass 配置，都会影响 UI 是否能够进入 SRP 批处理，从而影响最终性能表现。

本章将从 Built\-in Render Pipeline 入手，逐步分析 UI 在 URP 中的渲染方式、Camera Stack 对 UI 的影响、RenderQueue 排序机制，以及 UI 在 GPU 渲染阶段中的实际位置，从而完整还原 UGUI 在整个渲染管线中的执行流程。

## 9\.1 Built\-in Pipeline 渲染流程

Built\-in Render Pipeline 是 Unity 早期默认使用的固定渲染管线。整个渲染流程由 Unity 内部预先定义，开发者虽然能够通过 Shader、CommandBuffer 等方式进行部分扩展，但整体渲染顺序依然由引擎控制。

UGUI 在 Built\-in Pipeline 中，同样属于整个渲染体系的一部分。

很多开发者会认为 UI 是“最后直接绘制到屏幕上的二维元素”，但实际上，UGUI 仍然需要经过完整的渲染流程，包括顶点生成、Batch 构建、DrawCall 提交以及 GPU 执行等阶段。

因此，UI 的显示顺序、遮挡关系以及性能表现，本质上都与渲染管线密切相关。

### 9\.1\.1 Built\-in Pipeline 中的 UI 渲染位置

在 Built\-in Pipeline 中，一帧画面的渲染流程大致如下：

1. Camera 开始渲染 

2. 场景剔除（Culling） 

3. Shadow Pass 

4. 不透明物体渲染 

5. Skybox 渲染 

6. 透明物体渲染 

7. Image Effect 

8. UI 渲染 

9. FrameBuffer 输出 

对于大多数 Screen Space \- Overlay 模式的 UI 来说，Unity 通常会在场景渲染结束后再统一绘制 UI，因此 UI 看起来总是覆盖在场景最上层。

但这并不意味着 UI 绕过了渲染管线。

实际上，UI 仍然会：

- 生成 Mesh 

- 提交 DrawCall 

- 使用 Shader 

- 参与 GPU 渲染 

只是它被安排在更靠后的渲染阶段执行。

这一点非常重要。

因为很多 UI 排序问题，本质上都与 UI 插入 Render Pipeline 的阶段有关。

### 9\.1\.2 UGUI 的底层渲染流程

一个 UGUI 元素从生成到最终显示，大致会经历如下流程：

Graphic 生成顶点数据 → Canvas 构建 Batch → CanvasRenderer 提交渲染命令 → RenderPipeline 执行 DrawCall → GPU 输出像素

整个过程中，最核心的几个部分分别是 Graphic、Canvas 与 CanvasRenderer。

Graphic 是所有 UGUI 渲染组件的基类，例如 Image、RawImage、Text 等组件，本质上都属于 Graphic 的派生类。

Graphic 的核心职责是生成 UI 的顶点数据，包括：

- 顶点坐标 

- UV 

- 顶点颜色 

- 三角形索引 

这些数据最终会组成 UI Mesh。

Canvas 则负责整个 UI 系统的批处理管理。

它会收集当前 Canvas 下的所有 Graphic，并尝试进行：

- 材质合并 

- 纹理合并 

- DrawCall Batch 

当 UI 内容发生变化时，Canvas 会重新构建 Batch，这也是 UI Rebuild 开销的来源之一。

CanvasRenderer 位于 UGUI 与底层渲染系统之间。

它负责维护：

- Mesh 数据 

- Material 状态 

- Texture 信息 

- 渲染命令 

随后再将这些数据提交给底层 RenderPipeline。

需要注意的是，CanvasRenderer 并不真正负责绘制。

真正执行 DrawCall 的阶段，仍然属于 RenderPipeline 与 GPU。

### 9\.1\.3 Overlay 模式的渲染特点

Screen Space \- Overlay 是最常见的 UI 模式。

在该模式下，UI 不依赖 Camera，而是直接进入最终屏幕空间。

很多开发者因此误认为 Overlay UI “不走渲染管线”，实际上这是错误的。

Overlay UI 仍然属于 RenderPipeline 的一部分。

它依然需要：

- 使用 Shader 

- 提交 DrawCall 

- 参与 Batch 

- 消耗 GPU 

真正的区别在于 Overlay UI 不参与 Camera 的空间计算与深度排序。

Unity 通常会在所有 Camera 渲染结束后，再统一绘制 Overlay UI，因此它能够稳定覆盖在场景最上层。

从底层角度来看，其本质只是“UI 被插入到了最终屏幕输出阶段。”

### 9\.1\.4 Camera 模式与 World Space 模式

与 Overlay 模式不同，Screen Space \- Camera 与 World Space 都会进入 Camera 渲染体系。

Screen Space \- Camera 模式会将 UI 投影到指定摄像机空间中，因此它会受到 Camera 渲染流程影响。

例如：

- Camera Depth 

- Clear Flags 

- Post Processing 

- RenderTexture 

- Sorting Order 

都可能影响最终 UI 的显示结果。

这也是为什么某些后处理特效会作用于 Camera UI，而不会作用于 Overlay UI。

World Space 模式则更加特殊。

在该模式下，UI 已经完全进入三维世界坐标体系，其行为与普通 Mesh Renderer 十分接近。

它会：

- 参与深度测试 

- 被场景遮挡 

- 参与三维空间排序 

- 受到透视投影影响 

因此，World Space UI 本质上已经属于三维场景的一部分。

### 9\.1\.5 Built\-in Pipeline 中的 UI 排序机制

在 Built\-in Pipeline 中，UI 的最终绘制顺序并不仅仅由 Hierarchy 决定。

真正影响 UI 排序的因素包括：

- Canvas Sorting Order 

- Sorting Layer 

- RenderQueue 

- Depth 

- Shader Queue 

- Camera Depth 

这些系统会共同决定最终 DrawCall 的提交顺序。

例如：两个 Overlay Canvas 即使 Hierarchy 顺序不同，只要 Sorting Order 不同，最终绘制结果也可能完全改变。

而在 Camera UI 与 World Space UI 中，Depth 与 RenderQueue 的影响会更加明显。

因此，UI 排序问题本质上属于渲染管线问题，而不仅仅只是 UI 系统内部问题。

### 9\.1\.6 Built\-in Pipeline 中 UI 的局限性

Built\-in Pipeline 虽然实现简单，但其渲染流程本质上仍然属于固定管线。

对于 UGUI 而言，这意味着：

- UI 插入阶段难以自由控制 

- 渲染流程不够显式 

- 很难精确干预 RenderPass 

- 后处理与 UI 的协作有限 

- 渲染扩展能力较弱 

随着 SRP 体系的出现，Unity 开始逐渐将 UI 的渲染流程显式化。

在 URP 与 HDRP 中，UI 不再只是“固定阶段中的最后一步”，而是开始真正进入可编程 RenderPass 体系。

这也是现代 Unity 渲染架构与 Built\-in Pipeline 之间最本质的区别。

## 9\.2 URP 中的 UI 渲染

URP（Universal Render Pipeline）是 Unity 基于 Scriptable Render Pipeline（SRP）构建的新一代渲染管线。

与 Built\-in Render Pipeline 最大的不同在于，URP 不再使用 Unity 内部固定的渲染流程，而是将整个渲染过程拆分为多个可编程 RenderPass，并由 RenderPipeline 统一调度执行。

这意味着 UI 在 URP 中的渲染方式，也与 Built\-in Pipeline 存在明显差异。

在 Built\-in Pipeline 中，UI 的绘制阶段更多属于“引擎内部固定逻辑”，而在 URP 中，UI 会明确进入 RenderPass 体系，其执行顺序开始变得更加显式。

因此，理解 URP 中 UI 的渲染流程，本质上就是理解 UI 是如何被插入到 SRP 渲染阶段中的。

### 9\.2\.1 URP 的整体渲染结构

在 URP 中，一帧画面的渲染流程通常如下：

1. Camera 数据准备 

2. Culling 

3. Shadow Pass 

4. Depth PrePass 

5. Opaque Pass 

6. Skybox Pass 

7. Transparent Pass 

8. Post Processing 

9. Final Blit 

10. UI 渲染 

与 Built\-in Pipeline 相比，最大的变化在于 URP 会将整个渲染过程拆分为多个 RenderPass。

每个 Pass 都拥有：

- 独立执行阶段 

- 独立 RenderTarget 

- 独立 RenderState 

- 独立 Shader Pass 

因此，UI 不再只是“最后统一绘制”的固定流程，而是会被明确插入到某个 RenderPass 阶段。

### 9\.2\.2 UI 在 URP 中的插入位置

在默认情况下，URP 中的 Overlay UI 通常会在场景渲染结束后执行。

但与 Built\-in Pipeline 不同的是这个“最后阶段”本质上已经属于一个明确的 RenderPass。

Unity 内部会通过 ScriptableRenderer 调度 UI 渲染流程，并在最终输出阶段插入 UI DrawCall。

因此，从 SRP 视角来看，UI 不再是一个“特殊系统”，而是 RenderPass 体系中的一个普通渲染阶段。

这也是为什么：

- UI 可以与 Post Processing 协同工作 

- UI 可以输出到 RenderTexture 

- UI 可以被自定义 RendererFeature 干预 

- UI 可以进入自定义 RenderPass 

在 URP 中，UI 的可扩展性明显高于 Built\-in Pipeline。

### 9\.2\.3 Overlay UI 在 URP 中的行为变化

很多开发者在从 Built\-in Pipeline 切换到 URP 时，会发现 Overlay UI 的行为与过去并不完全一致。

例如：

- 某些后处理开始影响 UI 

- UI 与 RenderTexture 的关系发生变化 

- UI 与 Camera Stack 出现新的排序行为 

- 某些特效开始覆盖 UI 

这些问题的根本原因在于 URP 将 UI 纳入了更完整的 RenderPass 管理体系。

在 Built\-in Pipeline 中，Overlay UI 更像是“场景结束后的额外绘制阶段”。

而在 URP 中，UI 已经成为整个 RenderGraph 中的一部分。

因此，UI 会受到更多渲染阶段影响。

不过需要注意的是 Overlay UI 虽然不依赖 Camera 空间，但它仍然依赖最终 RenderTarget。

也就是说 Overlay UI 本质上仍然属于 GPU 渲染流程的一部分。

### 9\.2\.4 Camera UI 在 URP 中的特点

Screen Space \- Camera 模式在 URP 中会更加依赖 Camera 渲染体系。

因为 URP 中的大部分渲染行为，本质上都围绕 Camera 与 Renderer 展开。

当 UI 使用 Camera 模式时，它会：

- 参与 Camera 的渲染顺序 

- 参与 Camera Stack 

- 受到 Post Processing 影响 

- 受到 RenderTexture 影响 

- 参与透明阶段排序 

因此，Camera UI 在 URP 中已经不仅仅只是“屏幕 UI”，而更接近一个特殊的透明渲染对象。

特别是在多个 Camera 同时存在时，UI 的行为会明显复杂化。

例如：

- Base Camera 与 Overlay Camera 的 UI 排序 

- 不同 Camera 的 UI 覆盖关系 

- RenderTexture Camera 的 UI 输出 

- UI 与后处理的先后顺序 

这些都属于 URP 中非常常见的问题。

### 9\.2\.5 World Space UI 在 URP 中的本质

World Space UI 在 URP 中，本质上与普通 Mesh Renderer 已经非常接近。

它会：

- 进入透明物体阶段 

- 参与深度测试 

- 参与透明排序 

- 受到光照与后处理影响 

- 使用 Camera 矩阵进行投影 

因此，从 GPU 视角来看 World Space UI 与普通透明 Mesh 的差异已经非常小。

其真正特殊的地方，仅仅在于它的数据来源于 Canvas 系统，而不是 MeshFilter。

这也是为什么World Space UI 经常会与透明物体产生排序问题。

因为它本质上已经属于透明渲染体系的一部分。

### 9\.2\.6 URP 中 UI 的 RenderPass 特性

URP 最重要的特点之一，就是 RenderPass 显式化。

在 Built\-in Pipeline 中，开发者很难准确控制 UI 的插入阶段。

但在 URP 中，可以通过：

- ScriptableRendererFeature 

- ScriptableRenderPass 

- RenderPassEvent 

主动控制渲染阶段。

例如：

开发者可以选择：

- 在 UI 之前插入特效 

- 在 UI 之后绘制全屏 Pass 

- 将 UI 输出到指定 RenderTarget 

- 在透明阶段插入自定义 UI 效果 

这意味着 URP 中的 UI 已经真正进入了“可编程渲染体系”。

因此，现代 Unity UI 渲染的核心，不再只是 Canvas 系统本身，而是 Canvas 如何接入 SRP RenderPass。

## 9\.3 Camera Stack 与 UI

Camera Stack 是 URP 中非常重要的一套摄像机叠加机制。

在 Built\-in Render Pipeline 中，多个 Camera 通常依靠 Depth 顺序进行覆盖渲染，而在 URP 中，Unity 将多个摄像机的组合关系进一步结构化，形成了 Base Camera 与 Overlay Camera 的堆叠体系。

这一机制对 UI 渲染有非常直接的影响。

尤其是在以下场景中：

- 主场景与 UI 分离渲染 

- 小地图 Camera 

- RenderTexture Camera 

- 特效 Camera 

- 多层 UI 叠加 

- 后处理与 UI 混合 

如果不理解 Camera Stack 的执行顺序，就很容易出现：

- UI 被覆盖 

- UI 消失 

- UI 后处理异常 

- UI 排序错误 

- RenderTexture UI 不显示 

等问题。

因此，Camera Stack 本质上属于 UI 渲染流程中的重要组成部分。

### 9\.3\.1 Camera Stack 的基本结构

在 URP 中，摄像机主要分为两种类型：

1. Base Camera 

2. Overlay Camera 

其中，Base Camera 负责整个渲染链路的开始。

它通常负责：

- 创建 RenderTarget 

- 清除颜色缓冲 

- 清除深度缓冲 

- 执行场景渲染 

- 驱动后续 Overlay Camera 

Overlay Camera 则不会单独创建完整渲染目标，而是依附于 Base Camera 的渲染结果继续绘制。

多个 Overlay Camera 可以按照顺序依次叠加到同一个最终画面中。

其本质可以理解为“多个 Camera 共享同一个最终 FrameBuffer。”

因此，Camera Stack 的核心实际上是多个 Camera 共同参与同一帧的渲染流程。

### 9\.3\.2 UI 与 Camera Stack 的关系

在 UGUI 中，只有以下两种模式会真正参与 Camera Stack：

- Screen Space \- Camera 

- World Space 

而 Screen Space \- Overlay 不属于 Camera 渲染体系，因此不会直接参与 Camera Stack。

这是一个非常重要的区别。

很多开发者会误认为“所有 UI 都属于 Camera 渲染的一部分。”

实际上，Overlay UI 往往是在 Camera Stack 执行完成后，最后再统一绘制。

因此：

- Overlay UI 通常不会被 Camera 后处理影响 

- Overlay UI 不参与 Camera Depth 排序 

- Overlay UI 不属于 Camera Stack 的渲染链路 

而 Camera UI 与 World Space UI，则会真正进入 Camera Stack 流程。

### 9\.3\.3 Base Camera 与 UI

当 UI 绑定到 Base Camera 时，其渲染会直接进入主渲染流程。

例如：

- 场景渲染 

- 透明物体渲染 

- UI 渲染 

- 后处理 

都可能发生在同一个 Camera 渲染链路中。

此时 UI 的行为会受到多个 Camera 参数影响，包括：

- Clear Flags 

- Post Processing 

- HDR 

- MSAA 

- Render Scale 

例如：如果 Base Camera 开启了后处理，那么 Camera UI 很可能也会受到 Bloom、Color Grading 等效果影响。

这也是 Camera UI 与 Overlay UI 最明显的区别之一。

### 9\.3\.4 Overlay Camera 与 UI

Overlay Camera 通常用于额外叠加渲染内容。

例如：

- 武器 Camera 

- UI Camera 

- 特效 Camera 

- 小地图 Camera 

在很多项目中，会专门创建一个 UI Camera，并将其作为 Overlay Camera 叠加到主 Camera 上。

这种方式有几个明显优势。

首先，可以将 UI 与场景渲染分离。

例如：

主 Camera 负责三维场景。

UI Camera 只负责：

- UI 

- 特效 

- HUD 

这样能够避免部分场景后处理影响 UI。

其次，可以单独控制 UI 的渲染参数。

例如：

- 独立后处理 

- 独立 Culling Mask 

- 独立 RenderTexture 

- 独立 Clear Depth 

这在复杂项目中非常常见。

不过需要注意 Overlay Camera 不会重新清除颜色缓冲。

因此，如果 Clear Flags 或 Depth 配置不正确，很容易导致：

- UI 残留 

- UI 深度错误 

- UI 被前一层 Camera 遮挡 

### 9\.3\.5 Camera Stack 中的 UI 排序问题

在 Camera Stack 中，UI 排序会变得比传统 Built\-in Pipeline 更复杂。

因为此时最终显示结果不仅取决于：

- Canvas Sorting Order 

- Sorting Layer 

还会受到：

- Camera Stack 顺序 

- Camera Depth 

- RenderPass 执行阶段 

- Clear Depth 

- RenderTarget 

等多个系统共同影响。

例如：即使一个 UI 的 Sorting Order 更高，如果它所属的 Camera 更早执行，最终仍然可能被后续 Camera 覆盖。

因此，在 Camera Stack 中“Camera 顺序”的优先级往往高于“Canvas 排序”。

这是很多 UI 排序问题的根源。

### 9\.3\.6 Camera Stack 与后处理

URP 中的后处理与 Camera Stack 关系非常紧密。

因为后处理通常是在 Camera 渲染结束阶段执行。

因此，UI 到底是在后处理之前还是之后绘制，会直接影响最终视觉效果。

例如：如果 UI 属于 Base Camera ，则UI 可能会受到：

- Bloom 

- Motion Blur 

- Color Grading 

等效果影响。

而如果 UI 位于单独 Overlay Camera 中，则可以避免部分后处理作用于 UI。

因此，在实际项目中，经常会采用“场景 Camera \+ UI Overlay Camera”的结构。

其目的就是将 UI 从场景后处理中分离出来。

这也是现代 URP 项目中非常常见的一种 UI 渲染架构。

### 9\.3\.7 Camera Stack 的性能影响

虽然 Camera Stack 提供了更灵活的渲染结构，但它并不是没有代价的。

每增加一个 Camera，都可能额外产生：

- Culling 开销 

- RenderPass 开销 

- RenderTarget 切换 

- GPU 状态切换 

- 后处理开销 

因此，过度拆分 Camera 会明显增加渲染成本。

尤其是在 UI 系统中，如果：

- UI Camera 过多 

- RenderTexture 过多 

- 后处理叠加过多 

都会导致 Frame Debugger 中出现大量额外 Pass。

因此，在实际项目中，需要平衡：

- UI 分层结构 

- Camera Stack 数量 

- 后处理复杂度 

- GPU 提交成本 

否则 UI 系统本身也可能成为渲染性能瓶颈。

## 9\.4 RenderQueue 与排序

在 UGUI 中，很多开发者会将“排序”简单理解为 Hierarchy 顺序或者 Canvas 的 Sorting Order。

但从底层渲染角度来看，真正决定最终绘制顺序的，并不仅仅只有 UI 系统本身。

实际上，UI 的排序本质上属于 Render Pipeline 的一部分。

最终的绘制结果，会同时受到：

- RenderQueue 

- Sorting Layer 

- Sorting Order 

- Depth 

- 渲染阶段 

- Shader Queue 

- Camera 顺序 

等多个系统共同影响。

因此，UI 排序问题本质上并不是单一系统的问题，而是整个渲染管线协同工作的结果。

### 9\.4\.1 什么是 RenderQueue

RenderQueue 是 Unity 用于控制物体绘制顺序的一套渲染队列机制。

每个材质最终都会对应一个 RenderQueue 值。

Unity 会根据该值决定 DrawCall 的提交顺序。

RenderQueue 的数值越小，通常越早绘制。

Built\-in Pipeline 中最常见的几个队列如下：

UGUI 默认通常使用 Transparent 队列。

这是因为 UI 大量依赖 Alpha Blend。

因此，UI 本质上属于透明渲染体系的一部分。

### 9\.4\.2 UI 为什么属于透明队列

在 GPU 渲染中，不透明物体与透明物体的处理方式完全不同。

不透明物体通常会：

- 开启深度写入 

- 提前进行 ZTest 

- 使用 Early\-Z 优化 

而透明物体通常需要：

- 关闭深度写入 

- 保持混合顺序 

- 按绘制顺序进行 Alpha Blend 

UGUI 的大部分组件，例如：

- Image 

- Text 

- RawImage 

- TMP\_Text 

都需要进行透明混合。

因此，UI Shader 通常会：

- 使用 Blend 

- 关闭 ZWrite 

- 进入 Transparent Queue 

例如，默认 UI Shader 中经常可以看到：

```Plain Text
Tags
{
    "Queue"="Transparent"
}
```

这意味着 UI 会在大多数不透明物体之后进行绘制。

这也是 UI 能够覆盖场景的重要原因之一。

### 9\.4\.3 RenderQueue 对 UI 的影响

由于 UI 属于透明队列，因此它会受到透明渲染排序规则影响。

例如：透明物体之间通常无法像不透明物体一样依赖深度缓冲进行完全正确的遮挡。

因此绘制顺序本身会直接影响最终结果。

在 UGUI 中，如果两个 UI：

- 使用不同材质 

- 使用不同 Shader 

- 使用不同 RenderQueue 

那么最终排序结果可能会完全不同。

例如：一个 RenderQueue 为 3100 的 UI，即使 Sorting Order 更低，也可能后绘制。

这是因为RenderQueue 的优先级通常高于部分 UI 内部排序规则。

因此，很多“UI 明明层级更高却被覆盖”的问题，本质上都与 RenderQueue 有关。

### 9\.4\.4 Canvas 排序与 RenderQueue 的关系

UGUI 本身还提供了一套独立排序系统，包括：

- Sorting Layer 

- Order in Layer 

- Override Sorting 

这些系统主要作用于 Canvas 层级。

当多个 Canvas 同时存在时，Unity 会先根据：

1. Sorting Layer 

2. Order in Layer 

3. Hierarchy 

进行 Canvas 排序。

随后，进入真正的渲染阶段时，RenderQueue 又会进一步影响 DrawCall 的提交顺序。

因此 Canvas 排序并不是最终排序。

真正的最终排序，还会受到 RenderPipeline 影响。

这一点在以下场景中特别明显：

- World Space UI 

- Camera UI 

- UI 与粒子混合 

- UI 与透明物体混合 

- 多 Shader UI 系统 

这些情况下，RenderQueue 往往比 Canvas 层级更加重要。

### 9\.4\.5 World Space UI 的排序问题

World Space UI 是最容易出现 RenderQueue 问题的模式。

因为它已经完全进入三维透明渲染体系。

此时 UI 会：

- 参与透明排序 

- 参与深度测试 

- 参与 Camera 距离排序 

因此两个 World Space UI 即使 Canvas 排序正确，也仍然可能出现遮挡错误。

尤其是在以下情况下：

- 大量半透明 UI 

- 粒子与 UI 混合 

- 透明特效覆盖 UI 

- 多个透明 Shader 混合 

都可能出现排序异常。

其根本原因在于透明渲染本身并不存在真正完美的排序方案。

GPU 通常只能依赖：

- RenderQueue 

- 距离排序 

- DrawCall 顺序 

近似完成透明混合。

因此，World Space UI 的排序复杂度会远高于 Overlay UI。

### 9\.4\.6 UI Shader 与 RenderQueue

UGUI 的最终排序行为，与 Shader 有非常直接的关系。

因为 Shader 中的 Queue 标签会直接决定 RenderQueue。

例如：

```Plain Text
Tags
{
    "Queue"="Transparent"
}
```

Unity 会根据该标签自动计算材质队列。

开发者也可以手动修改材质 Queue。

例如：

```Plain Text
material.renderQueue = 3100;
```

但需要注意：

强行修改 UI RenderQueue，可能导致：

- Batch 被打断 

- 排序异常 

- Mask 行为异常 

- Stencil 错误 

因为 UGUI 的很多内部逻辑，本身默认建立在 Transparent Queue 基础之上。

因此，除非明确理解渲染顺序，否则通常不建议随意修改 UI RenderQueue。

### 9\.4\.7 UI 与粒子的排序问题

UI 与粒子的混合，是项目中非常常见的问题。

尤其是在：

- 技能特效 

- UI 动效 

- 全屏特效 

- 飘字系统 

中最为明显。

问题的根源在于粒子系统通常也属于 Transparent Queue。

因此 UI 与粒子实际上属于同一个透明渲染体系。

此时最终结果会同时受到：

- RenderQueue 

- Camera 距离 

- Sorting Layer 

- DrawCall 顺序 

影响。

因此，经常会出现：

- 粒子覆盖 UI 

- UI 穿插粒子 

- 半透明混合错误 

等问题。

很多项目会采用以下方案解决：

- 单独 UI Camera 

- 提高 UI RenderQueue 

- 分离 RenderPass 

- 使用 Overlay UI 

- RenderTexture 合成 

本质上都是在重新控制透明渲染顺序。

### 9\.4\.8 RenderQueue 的本质

从底层角度来看，RenderQueue 的本质其实非常简单：

它决定的是“DrawCall 进入 GPU 的先后顺序。”

而 GPU 对透明物体的最终结果，又高度依赖绘制顺序。

因此RenderQueue 本质上属于 GPU 渲染阶段的重要排序规则。

UGUI 虽然是 UI 系统，但它最终仍然属于 GPU 透明渲染体系的一部分。

这也是为什么真正复杂的 UI 排序问题，最终都必须从 RenderPipeline 与 GPU 渲染顺序角度分析。

## 9\.5 SRP Batcher 支持情况

SRP Batcher 是 Unity 在 Scriptable Render Pipeline（SRP）体系下提供的一套 GPU 提交优化机制。

它的核心目标，是减少 CPU 向 GPU 提交 DrawCall 时产生的状态切换开销。

在传统渲染流程中，即使多个物体使用相同 Shader，CPU 在提交 DrawCall 时，仍然需要频繁切换：

- Shader 常量缓冲 

- Material 状态 

- GPU Uniform 数据 

这些状态切换会带来明显的 CPU 开销。

而 SRP Batcher 的核心思想则是将 Shader 常量缓冲结构标准化，从而减少 DrawCall 之间的状态切换成本。

对于大量物体同时渲染的场景而言，SRP Batcher 能够显著降低 CPU RenderThread 压力。

但在 UGUI 中，SRP Batcher 的支持情况却相对特殊。

很多开发者会发现即使项目已经开启 URP，并启用了 SRP Batcher，UI 的 DrawCall 数量与性能表现仍然没有明显变化。

其根本原因在于 UGUI 的渲染方式，与 SRP Batcher 的优化目标并不完全一致。

### 9\.5\.1 SRP Batcher 的工作原理

SRP Batcher 本质上是一种“减少 Material 状态切换”的优化机制。

在传统渲染中，每个 DrawCall 提交前，CPU 通常都需要重新上传：

- UnityPerMaterial 

- UnityPerDraw 

- Shader Uniform 

等常量数据。

而 SRP Batcher 会将这些数据缓存在 GPU 常量缓冲区中。

如果多个 DrawCall：

- 使用相同 Shader Variant 

- 使用兼容的 CBUFFER 结构 

那么 GPU 就能够复用已有状态，而不需要重新上传整套 Shader 数据。

因此，SRP Batcher 的优化重点并不是减少 DrawCall 数量，而是减少 DrawCall 之间的 CPU 状态切换。

这一点非常重要。

很多开发者会误认为开启 SRP Batcher 后 DrawCall 会减少。

实际上并不会。DrawCall 数量通常不会发生明显变化。

真正减少的是 CPU RenderThread 的提交开销。

### 9\.5\.2 UGUI 为什么难以充分利用 SRP Batcher

UGUI 的渲染结构，与普通 Mesh Renderer 有明显区别。

普通 Renderer 通常具有：

- 稳定 Mesh 

- 稳定 Material 

- 固定 Shader 状态 

因此更容易进入 SRP Batcher。

而 UGUI 的特点则是：

- 动态 Batch 

- 顶点频繁变化 

- 材质切换频繁 

- Stencil 状态变化频繁 

- Mask 会生成额外材质 

- Graphic 会动态修改 Mesh 

这些行为会导致UI 的渲染状态很难保持稳定。

而 SRP Batcher 最依赖的，恰恰就是稳定的 Shader 与 Material 状态。

因此，即使 UI Shader 本身兼容 SRP Batcher，UGUI 也未必能够真正获得明显收益。

### 9\.5\.3 UI Shader 与 SRP Batcher

一个 Shader 是否兼容 SRP Batcher，主要取决于 CBUFFER 是否符合 SRP 规范。

例如：

```Plain Text
CBUFFER_START(UnityPerMaterial)
float4 _Color;
float4 _MainTex_ST;
CBUFFER_END
```

Unity 要求所有 Material 属性都必须进入统一的 CBUFFER。

否则 Shader 将无法进入 SRP Batcher。

在 URP 中，大部分官方 Lit Shader 都已经兼容 SRP Batcher。

但 UGUI 中经常存在以下问题：

- 使用旧版 UI Shader 

- 使用内置 Default UI Shader 

- 使用第三方 UI 特效 Shader 

- 使用 GrabPass 

- 使用多 Pass Shader 

这些情况都可能导致 UI 无法进入 SRP Batcher。

尤其是很多旧版 UI Shader，本身并不是基于 SRP 架构设计的。

因此在 Frame Debugger 中，经常能够看到“SRP Batcher：Not Compatible”

### 9\.5\.4 Mask 对 SRP Batcher 的影响

Mask 是 UGUI 中最容易破坏 SRP Batcher 的系统之一。

因为 Mask 会引入：

- Stencil State 修改 

- Material 实例化 

- 不同 RenderState 

例如：一个普通 Image 与一个 Mask 下的 Image，即使使用同一个 Shader，也很可能生成不同材质。

这是因为Stencil Ref 与 Stencil Compare 通常会发生变化。

而 SRP Batcher 要求 DrawCall 之间尽可能保持一致的 Material 状态。

因此，大量 Mask 往往会导致：

- Batch 被打断 

- Material 数量增加 

- SRP Batcher 失效 

这也是复杂 UI 中 CPU RenderThread 开销上升的重要原因之一。

### 9\.5\.5 动态材质对 SRP Batcher 的影响

UGUI 中还有一个非常常见的问题：动态材质实例化。

例如：

- 修改 Material 属性 

- 使用 MaterialForRendering 

- 使用 Outline 

- 使用 Shadow 

- 使用 UI 特效插件 

都可能导致新的 Material Instance 被创建。

而 SRP Batcher 对 Material 一致性要求非常高。

一旦材质实例过多，即使 Shader 相同，也可能无法有效复用 GPU 常量缓冲。

因此大量动态材质通常意味着 SRP Batcher 收益迅速下降。

### 9\.5\.6 UGUI 更依赖 Canvas Batch

对于 UGUI 而言，真正最重要的优化机制，其实并不是 SRP Batcher，而是 Canvas Batch。

因为 UGUI 本身最大的性能瓶颈，通常并不在 Shader 状态切换，而在于：

- Canvas Rebuild 

- Batch 重建 

- 顶点上传 

- DrawCall 拆分 

因此相比 SRP Batcher，UGUI 往往更依赖：

- 合理拆分 Canvas 

- 减少材质切换 

- 减少 Mask 

- 合理使用图集 

- 降低 Rebuild 范围 

这些优化手段。

这也是为什么很多 UI 项目即使 SRP Batcher 完全失效，性能仍然可能正常。

因为 UI 的核心瓶颈通常并不在 SRP Batcher。

### 9\.5\.7 Frame Debugger 中的 SRP Batcher

在实际项目中，可以通过 Frame Debugger 查看 UI 是否进入 SRP Batcher。

如果兼容，通常会看到：

- SRP Batch 

- Compatible 

- Batcher Active 

等标记。

如果不兼容，则可能显示：

- Node use different shader keywords 

- Material property mismatch 

- Different CBUFFER layout 

- Different RenderState 

等信息。

这些提示本质上都意味着当前 DrawCall 无法共享 GPU 状态。

因此 Frame Debugger 是分析 UI SRP Batcher 问题最重要的工具之一。

### 9\.5\.8 SRP Batcher 对 UI 的真实意义

从整体角度来看，SRP Batcher 对 UGUI 并不是无意义的。

但它的重要性，通常低于：

- Canvas Batch 

- DrawCall 合并 

- Rebuild 控制 

因为 UGUI 本身属于高动态渲染体系。

它与 SRP Batcher 最擅长优化的“稳定 Mesh Renderer 场景”并不完全一致。

因此，SRP Batcher 对 UI 更像是额外优化项，而不是决定 UI 性能的核心机制。

真正影响 UI 性能的关键，仍然是 UI 如何组织 Batch，以及 UI 如何减少重建与状态切换。

## 9\.6 UI 在 GPU 渲染阶段的位置

从 Unity 上层架构来看，UGUI 属于一个 UI 系统。

但从 GPU 角度来看，UI 与普通 Mesh、粒子、透明物体之间，其实并不存在本质区别。

GPU 并不理解：

- Button 

- Image 

- Text 

- Canvas 

这些高层概念。

GPU 真正能够识别的，只有：

- Vertex Buffer 

- Index Buffer 

- Texture 

- Shader 

- RenderState 

- DrawCall 

因此，当 UI 进入 RenderPipeline 后，它本质上已经被转换为了标准 GPU 渲染数据。

理解 UI 在 GPU 渲染阶段中的真实位置，对于理解以下问题非常重要：

- UI 为什么会产生 DrawCall 

- UI 为什么会消耗 FillRate 

- UI 为什么会导致 Overdraw 

- UI 为什么会出现透明排序问题 

- UI 为什么无法完全绕过 GPU 

这些问题的根源，最终都属于 GPU 渲染行为。

### 9\.6\.1 UI 如何进入 GPU

在 UGUI 中，Graphic 会首先生成顶点数据。

这些数据通常包括：

- 顶点坐标 

- UV 

- 顶点颜色 

- 三角形索引 

随后，Canvas 会将多个 UI 元素进行 Batch 合并。

合并完成后，CanvasRenderer 会生成真正的 Mesh 数据，并向 RenderPipeline 提交 DrawCall。

此时，UI 已经不再是“Image” 或 “Text”，而是“GPU 可渲染 Mesh”。

最终，这些 DrawCall 会进入 GPU Command Buffer，等待 GPU 执行。

因此，从 GPU 视角来看 UI 本质上只是另一种透明 Mesh。

### 9\.6\.2 UI 在 GPU Pipeline 中的位置

在 GPU 渲染阶段，UI 通常会经历如下流程：

1. Vertex Shader 

2. Clipping 

3. Rasterization 

4. Fragment Shader 

5. Blending 

6. FrameBuffer 输出 

这一流程与普通透明物体几乎完全一致。

其中 Vertex Shader 负责处理 UI 顶点变换。

例如：

- RectTransform 顶点坐标 

- MVP 矩阵变换 

- UV 计算 

随后 GPU 会进行三角形裁剪与光栅化。

Rasterization 阶段会将三角形转换为像素片段。

Fragment Shader 则负责：

- 纹理采样 

- Alpha 计算 

- Mask 计算 

- 颜色混合 

最终再通过 Blend 阶段输出到 FrameBuffer。

因此 UI 本质上仍然属于 GPU 的透明渲染流程。

### 9\.6\.3 UI 为什么容易产生 Overdraw

UI 是最容易产生 Overdraw 的系统之一。

因为大多数 UI：

- 使用透明混合 

- 不写入深度 

- 互相层层覆盖 

这意味着 GPU 无法像不透明物体一样提前剔除被遮挡像素。

例如：一个完全被覆盖的 UI Image，即使最终不可见，其 Fragment Shader 仍然可能已经执行。

如果多个半透明 UI 互相覆盖：

GPU 就会反复执行：

- Fragment Shader 

- Texture Sampling 

- Alpha Blend 

这会显著增加 FillRate 开销。

尤其是在移动平台中 UI 往往比三维场景更容易成为 GPU 瓶颈。

因为大量全屏透明 UI 会迅速提高 Overdraw。

### 9\.6\.4 UI 的 FillRate 问题

FillRate 指的是 GPU 每秒能够填充的像素数量。

UI 对 FillRate 的消耗通常非常明显。

原因在于：

UI 往往具有以下特点：

- 大面积透明区域 

- 全屏覆盖 

- 多层半透明叠加 

- 高频 Fragment Shader 执行 

例如：一个全屏半透明 Image，即使纹理本身非常简单，也仍然需要 GPU 对整块屏幕执行 Fragment Shader。

如果：

- 弹窗过多 

- 特效过多 

- 半透明遮罩过多 

都会导致 FillRate 压力迅速增加。

因此 UI 的 GPU 开销，很多时候并不来自 Vertex，而来自 Fragment。

这也是 UI 优化中非常重要的一点。

### 9\.6\.5 UI 的 Blend 阶段

UGUI 大量依赖 Alpha Blend。

典型 UI Shader 通常会使用：

```Plain Text
Blend SrcAlpha OneMinusSrcAlpha
```

这意味着最终颜色会与屏幕已有颜色进行混合。

Blend 的存在，会导致 GPU 很难进行 Early\-Z 优化。

因为 GPU 必须先读取当前像素颜色，再完成混合计算。

因此透明 UI 的 GPU 成本通常高于不透明物体。

尤其是：

- 大量半透明 UI 

- 动态特效 UI 

- 模糊 UI 

- 发光 UI 

都会进一步提高 Blend 成本。

### 9\.6\.6 UI 与深度缓冲

大多数 Overlay UI 默认不会写入深度缓冲。

例如：

```Plain Text
ZWrite Off
```

因此 UI 通常无法利用深度缓冲进行遮挡优化。

这也是 UI 容易产生 Overdraw 的根本原因之一。

不过在 World Space UI 中，情况会有所不同。

部分 World Space UI 会参与：

- ZTest 

- 深度比较 

- 深度排序 

因此它可能会被场景遮挡。

但即便如此，大部分 UI Shader 依然会关闭 ZWrite。

因为 UI 更依赖透明混合，而不是深度写入。

### 9\.6\.7 UI 的 DrawCall 本质

很多开发者会认为“UI 是二维系统，因此 DrawCall 开销应该很低。”

实际上并非如此。

UI 的每一次 Batch 拆分，本质上都会生成新的 DrawCall。

例如：

- 材质变化 

- Texture 变化 

- Mask 变化 

- Stencil 变化 

- Shader Keyword 变化 

都会导致新的 GPU 提交。

而 GPU 并不会关心这些 DrawCall 是来自 UI 还是 Mesh。

因此UGUI 的 DrawCall 本质上与普通渲染 DrawCall 完全一致。

它们都会：

- 切换 RenderState 

- 切换 Shader 

- 切换 Texture 

- 提交 GPU 命令 

这也是为什么复杂 UI 系统同样可能产生非常高的 GPU 开销。

### 9\.6\.8 UI 在 GPU 中的真实本质

从 GPU 角度来看，UGUI 本质上并不是“特殊 UI 系统”。

它最终只是：

- 一组透明 Mesh 

- 一组 DrawCall 

- 一组 Fragment Shader 

- 一组 Blend 操作 

UI 与普通透明物体最大的区别，仅仅在于它的数据来源于 Canvas 系统。

但当数据真正进入 GPU 后 UI 与透明 Mesh Renderer 的本质差异已经非常小。

因此，真正理解 UGUI 渲染，本质上必须理解：

- DrawCall 

- RenderState 

- Blend 

- FillRate 

- Overdraw 

- GPU Pipeline 

因为 UI 的最终执行位置，从来都不在 Canvas，而是在 GPU。

## 本章小结

本章从 Unity 渲染管线的整体结构出发，系统梳理了 UGUI 在 Built\-in 与 URP 两种渲染体系中的执行路径，完整还原了 UI 从 CPU 提交到 GPU 输出的全过程。

在 Built\-in Pipeline 中，UI 作为渲染流程的末端阶段被统一执行，通过固定的插槽式机制接入最终屏幕输出流程，使其天然具备覆盖场景内容的表现特性。这种模式整体结构清晰，执行路径固定，但 UI 与渲染流程之间的耦合较强，可调度性较弱，难以进行精细化控制。

在 URP 中，渲染流程被重构为基于 RenderPass 的可编排体系，UI 不再作为隐式末端步骤存在，而是作为明确的渲染阶段被 ScriptableRenderer 统一调度。这种结构变化使 UI 从固定流程中的“默认输出结果”，转变为 RenderPass 链路中的一个可控节点，从而增强了 UI 与现代渲染架构之间的融合能力与扩展能力。

Camera Stack 的引入进一步改变了 UI 的渲染语义，使 UI 不再局限于单一屏幕空间的最终输出结果，而是可以嵌入到多 Camera 级联渲染链路中参与整体画面构建。在该机制下，UI 的最终表现同时受到 Camera 拓扑结构与 Canvas 排序体系的共同影响，使 UI 的层级模型从单维排序转变为多维渲染关系的组合结果。

RenderQueue 则从材质层面定义了 GPU 提交顺序，是 UI 最终绘制顺序的重要底层保障机制。它与 Sorting Layer、Canvas 层级以及 Camera 渲染顺序共同构成了一套完整的 UI 排序系统，使 UI 的最终输出不再仅由单一规则决定，而是由多个渲染阶段协同控制。

SRP Batcher 的引入进一步优化了 UI 在 CPU 侧的渲染状态切换成本，使大量 UI 在频繁更新的场景下具备更稳定的性能表现。然而其优化效果高度依赖 Shader 与 Material 的规范化程度，一旦材质状态复杂化或产生大量实例化行为，其收益会显著下降。

最终在 GPU 渲染阶段，UI 被统一转换为标准透明 Mesh 数据参与光栅化与片元计算流程。此时 UI 已不再具有系统级语义差异，其性能瓶颈主要集中在 Overdraw、透明混合以及 FillRate 消耗，而非 UI 逻辑本身。

总体来看，本章建立了一条完整的认知链路：UI 并不是独立运行的渲染系统，而是深度嵌入 Unity 渲染管线中的一个渲染参与者。从 CPU 侧的批处理构建，到渲染管线的阶段调度，再到 GPU 端的像素输出，每一层机制都共同决定了 UI 的最终性能表现与视觉结果。

