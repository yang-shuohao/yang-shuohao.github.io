---
title: 第17章 UI Shader 机制
categories:
  - Unity
  - UGUI
tags:
  - UGUI
  - Unity
cover: /images/UGUI/UGUI.png
abbrlink: 925c6dfa
date: 2026-08-26 00:00:17
---
# 第17章 UI Shader 机制

在 UGUI 渲染体系中，Mesh 的生成仅完成了“几何数据”的构建，而真正决定 UI 最终视觉表现的，是 Shader 所定义的像素处理逻辑。如果说 Graphic 系统负责回答“UI 的形状是什么”，那么 Shader 系统则负责回答“这些形状最终如何被渲染”。

从整体流程来看，UI 渲染可以划分为两个阶段：CPU 侧的 Mesh 构建阶段，以及 GPU 侧的 Shader 渲染阶段。前者通过 VertexHelper 生成标准化的顶点数据，后者则通过 Shader 对这些数据进行光栅化与像素计算，从而输出最终图像。

UI Shader 在 Unity 中属于一类特殊的 Shader 类型。它并不追求复杂的光照计算，而是围绕“2D界面渲染”这一目标进行设计，其核心关注点包括透明混合、颜色叠加、纹理采样以及裁剪控制等。这使得 UI Shader 在结构上与传统 3D Shader 存在明显差异。

在 UGUI 的默认实现中，UI Shader 通常基于 Unlit 模型构建，不参与光照计算，而是直接使用顶点颜色与纹理结果进行输出。这种设计保证了 UI 渲染的稳定性与性能，同时也为合批提供了良好条件。

从数据流角度来看，UI Shader 的输入主要来自 UIVertex 结构，包括 Position、Color、UV 以及额外扩展数据。这些数据在顶点阶段被传入 Shader，并在片元阶段完成纹理采样与颜色混合，最终输出到屏幕。

此外，UI Shader 还承担着一些关键系统功能，例如 Mask 裁剪、Stencil 测试以及 Alpha 混合等。这些功能并不体现在 Mesh 结构中，而是通过 Shader 与 GPU 状态共同实现，因此理解 UI Shader，对于完整掌握 UGUI 渲染流程至关重要。

本章将从 UI Shader 的基础结构入手，深入分析 Default UI Shader 的实现方式，进一步探讨顶点数据在 Shader 中的传递路径，以及 Blend、Stencil 等关键机制在 UI 渲染中的作用，从而建立起 UGUI 从 Mesh 到像素输出的完整认知链路。

## 17\.1 UI Shader 结构

在 UGUI 的整个渲染体系中，Shader 并不仅仅只是一个简单的着色程序，它实际上决定了 UI 顶点数据最终如何被 GPU 解释与渲染。前面章节已经分析过，UGUI 会在 CPU 阶段生成并修改 Mesh，而 Shader 则负责在 GPU 阶段对这些顶点数据进行最终处理。因此如果说 Mesh 决定了 UI 的几何结构，Material 决定了 Shader 参数，那么 Shader 决定的则是 UI 最终如何显示。

UGUI 中所有的 Image、Text、RawImage 等组件，最终都会进入 UI Shader 的渲染流程。无论是透明混合、颜色叠加、Mask 裁剪，还是 RectMask2D 的软裁剪，其底层最终都建立在 UI Shader 体系之上。因此理解 UI Shader 的结构，是深入分析 UI 渲染原理、Stencil 裁剪、透明混合以及 UI 特效实现的重要前提。

### 17\.1\.1 UI Shader 在渲染流程中的位置

在 UGUI 中，一个 UI 元素从生成到最终显示，大致会经历如下流程：

```Plain Text
Graphic
    ↓
OnPopulateMesh
    ↓
VertexHelper
    ↓
Mesh
    ↓
CanvasRenderer
    ↓
Material
    ↓
Shader
    ↓
GPU
```

其中 Graphic 负责生成 UI 几何数据，VertexHelper 负责管理顶点结构，CanvasRenderer 负责提交渲染命令，而 Shader 则负责最终的 GPU 计算。

当 CanvasRenderer 提交 DrawCall 时，本质上会向 GPU 提交 Mesh、Material、Texture 与 Shader 等数据，而 Shader 会进一步完成顶点变换、UV 计算、颜色混合、Alpha 透明、Stencil 测试以及最终像素输出。因此从整个渲染链路来看，UI Shader 实际上处于 UGUI 渲染流程的最后阶段。

### 17\.1\.2 UI Shader 的基本组成

Unity 的 UI Shader 本质上仍然属于标准 Shader，因此其整体结构与普通 3D Shader 并没有本质区别。一个典型 UI Shader 通常包含 Properties、SubShader、Pass、Vertex Shader 与 Fragment Shader 等部分。

例如：

```Plain Text
Shader "UI/MyShader"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
        _Color ("Tint", Color) = (1,1,1,1)
    }

    SubShader
    {
        Tags
        {
            "Queue"="Transparent"
            "RenderType"="Transparent"
        }

        Pass
        {
            CGPROGRAM

            #pragma vertex vert
            #pragma fragment frag

            ENDCG
        }
    }
}
```

虽然 UI Shader 的整体结构与普通 Shader 十分接近，但 Unity 实际上针对 UI 渲染体系做了大量特殊设计。例如默认透明混合、关闭深度写入、支持 Stencil、支持 RectMask2D、支持 AlphaClip 以及适配 Canvas Batch 等机制，都属于 UI Shader 的特殊处理逻辑。因此 UI Shader 并不只是一个普通透明 Shader，而是 Unity 专门为 Canvas 渲染体系设计的一套特殊 Shader 结构。

### 17\.1\.3 UI Shader 的核心特点

相比普通 3D Shader，UI Shader 有几个非常明显的特征。

首先，UI Shader 默认工作在 Transparent 队列。由于 UI 天然需要支持半透明、层级叠加以及 Alpha 混合，因此绝大多数 UI 都不会进入 Opaque 渲染队列。这意味着 UI 基本都会参与透明排序。

其次，UI Shader 通常会关闭深度写入。

```Plain Text
ZWrite Off
```

这是因为 UI 更依赖 Canvas 的层级顺序，而不是深度缓冲。如果开启深度写入，那么前面的 UI 会遮挡后面的 UI，半透明关系也会出现错误，因此 UGUI 默认采用的是“层级排序 \+ 透明混合”的渲染方式，而不是传统 3D 的深度排序。

另外，UI Shader 几乎都会默认开启 Blend。

```Plain Text
Blend SrcAlpha OneMinusSrcAlpha
```

这意味着最终颜色会按照 Alpha 值进行透明混合。

其结果可以理解为：

```Plain Text
FinalColor = SrcColor * SrcAlpha + DstColor * (1 - SrcAlpha)
```

也正因为如此，UI 才能够实现半透明、平滑边缘以及抗锯齿等效果。

除此之外，UI Shader 还非常依赖顶点颜色。UGUI 中大量颜色数据并不是来自贴图，而是来自 Mesh 顶点。例如 Graphic\.color、Text 字体颜色、顶点渐变以及各种 Mesh Effect 修改后的颜色，最终都会写入 UIVertex\.color，然后传递给 Shader。

因此 UI Shader 中通常会存在如下逻辑：

```Plain Text
fixed4 color = tex2D(_MainTex, uv) * IN.color;
```

这里贴图颜色会与顶点颜色相乘，从而形成最终输出结果。

### 17\.1\.4 UI Shader 的顶点输入结构

由于 UI Shader 需要接收 UGUI 生成的 Mesh 数据，因此其顶点输入结构与普通模型 Shader 存在明显区别。

一个典型的 UI 顶点输入结构如下：

```Plain Text
struct appdata_t
{
    float4 vertex   : POSITION;
    float4 color    : COLOR;
    float2 texcoord : TEXCOORD0;
};
```

其中 vertex 来自 UIVertex\.position，color 来自 UIVertex\.color，而 texcoord 则来自 UIVertex\.uv0。这些数据最终会通过 VertexHelper、Mesh 与 CanvasRenderer 上传至 GPU。

因此 UI Shader 与 UGUI Mesh 系统实际上是强绑定关系。Shader 所接收到的所有顶点数据，本质上都来自 UGUI 在 CPU 阶段构建好的 UI Mesh。

### 17\.1\.5 UI Shader 的像素输出

UI Shader 的 Fragment 阶段通常相对简单，其核心逻辑一般只有纹理采样与颜色混合。

例如：

```Plain Text
fixed4 frag(v2f IN) : SV_Target
{
    fixed4 color = tex2D(_MainTex, IN.texcoord);
    return color * IN.color;
}
```

整个过程本质上只有两个步骤。首先通过 UV 采样贴图颜色，然后再与顶点颜色进行相乘，最终输出到 RenderTarget。

虽然逻辑看起来并不复杂，但实际上 Alpha 混合、Stencil 裁剪、RectMask2D、Soft Clip 等大量 UI 渲染功能，最终都会在 Fragment 阶段进一步参与运算。因此 UI Fragment Shader 实际上承担了大量最终像素处理工作。

### 17\.1\.6 UI Shader 与 UGUI 的关系

很多开发者会误认为 UGUI 与 Shader 是彼此独立的两个系统，但实际上并非如此。

UGUI 本身并不负责真正的像素渲染，它负责的是顶点生成、Mesh 构建、材质管理以及 DrawCall 提交，而真正决定像素最终如何显示的，始终是 Shader。

因此从本质上来看，UGUI 更像是一个 UI Mesh 生成系统，而 Shader 则是一个 UI 像素渲染系统。二者共同组成了完整的 UI 渲染体系。

## 17\.2 Default UI Shader 解析

在整个 UGUI 渲染体系中，Default UI Shader 是最核心的默认 Shader。Unity 中绝大多数 Image、Text、RawImage 等 UI 组件，在没有指定自定义材质时，最终都会使用这一 Shader 进行渲染。因此，理解 Default UI Shader 的内部结构，本质上就是理解 UGUI 默认渲染体系的核心实现。

相比普通 3D Shader，Default UI Shader 并不追求复杂光照或高级渲染效果，它的核心目标始终是高效、稳定以及适配 Canvas 渲染体系。因此它内部大量设计都围绕透明混合、Canvas Batch、Stencil Mask、RectMask2D 裁剪、顶点颜色支持以及 Alpha 裁剪等机制展开，而这些能力共同构成了 UGUI 默认渲染系统的基础。

### 17\.2\.1 Default UI Shader 的整体结构

Unity 内置的 Default UI Shader 本质上仍然属于标准 Vertex/Fragment Shader，其整体结构与普通 Shader 并没有本质区别。

例如：

```Plain Text
Shader "UI/Default"
{
    Properties
    {
        [PerRendererData] _MainTex ("Sprite Texture", 2D) = "white" {}
        _Color ("Tint", Color) = (1,1,1,1)
    }

    SubShader
    {
        Tags
        {
            "Queue"="Transparent"
            "RenderType"="Transparent"
            "CanUseSpriteAtlas"="True"
        }

        Stencil
        {
        }

        Cull Off
        Lighting Off
        ZWrite Off
        Blend SrcAlpha OneMinusSrcAlpha

        Pass
        {
            CGPROGRAM

            #pragma vertex vert
            #pragma fragment frag

            ENDCG
        }
    }
}
```

虽然整个 Shader 代码看起来并不复杂，但其中几乎每一个设置都与 UGUI 的底层渲染机制直接相关。实际上，Default UI Shader 的复杂度并不在视觉效果本身，而在于它需要高度适配整个 Canvas 渲染体系。

### 17\.2\.2 Transparent Queue 的作用

Default UI Shader 默认工作在 Transparent 渲染队列中。

```Plain Text
"Queue"="Transparent"
```

这一设置意味着 UI 会在所有不透明物体之后进行渲染。

由于 UI 天然需要支持半透明、层级叠加以及 Alpha 混合，因此几乎所有 UGUI 元素都工作在透明队列中。这也意味着 UI 的显示顺序主要依赖 Canvas 层级、Hierarchy 顺序以及 Sorting Order，而不是传统 3D 渲染中的深度缓冲。

因此在 UGUI 中，即使两个 UI 元素的 Z 坐标不同，也不一定会影响最终显示顺序，因为真正决定渲染先后的往往是 Canvas 排序系统。

### 17\.2\.3 ZWrite Off 的意义

Default UI Shader 默认关闭深度写入。

```Plain Text
ZWrite Off
```

这一设置对于 UI 系统至关重要。

因为 UI 大量存在透明区域，如果开启深度写入，那么前面的 UI 会提前写入 Depth Buffer，从而导致后面的 UI 即使本应显示，也会被深度测试错误遮挡。

例如，一个半透明 Image 如果写入了深度，那么后续 UI 即使透明区域本不应该遮挡，也可能被错误剔除。因此 UGUI 默认完全放弃基于深度缓冲的排序机制，而采用“透明混合 \+ Canvas 层级”的方式管理 UI 显示顺序。

这也是 UI 渲染与传统 3D 渲染之间最明显的区别之一。

### 17\.2\.4 Blend 混合模式

Default UI Shader 中最关键的一行配置之一就是：

```Plain Text
Blend SrcAlpha OneMinusSrcAlpha
```

这是标准的 Alpha Blend 混合模式。

其本质计算方式如下：

```Plain Text
FinalColor = SrcColor * SrcAlpha + DstColor * (1 - SrcAlpha)
```

其中 SrcColor 表示当前 UI 像素颜色，DstColor 表示屏幕已有颜色，而 SrcAlpha 则表示当前 UI 的透明度。

因此 Alpha 越低，当前 UI 就越透明。

UGUI 中所有半透明图片、字体抗锯齿、平滑边缘以及 Fade 动画，本质上都依赖这一 Blend 混合机制。如果关闭 Blend，那么 UI 将完全失去透明能力。

### 17\.2\.5 顶点颜色机制

Default UI Shader 非常依赖顶点颜色。

其顶点输入结构通常如下：

```OpenGL Shading Language
struct appdata_t
{
    float4 vertex   : POSITION;
    float4 color    : COLOR;
    float2 texcoord : TEXCOORD0;
};
```

其中 color 数据实际上来自 UIVertex\.color，而这个颜色又可能来自 Graphic\.color、Text 字体颜色、Mesh Effect 或顶点渐变等系统。

最终在 Fragment Shader 中，贴图颜色会与顶点颜色进行相乘。

```OpenGL Shading Language
color *= IN.color;
```

因此 UGUI 中大量颜色变化实际上并不是 Shader 内部动态计算的，而是 CPU 在 Mesh 阶段直接写入顶点颜色后，再由 Shader 完成最终混合。

这也是为什么 UGUI 的颜色修改通常不会增加额外 DrawCall 的原因。

### 17\.2\.6 顶点变换过程

Default UI Shader 的 Vertex Shader 本身非常轻量，其核心流程通常只有顶点坐标转换、UV 传递以及颜色传递。

例如：

```OpenGL Shading Language
OUT.vertex = UnityObjectToClipPos(IN.vertex);
OUT.texcoord = IN.texcoord;
OUT.color = IN.color;
```

由于 UGUI 已经在 CPU 阶段完成了绝大多数几何计算，因此 UI Shader 的 Vertex 阶段并不需要像 3D Shader 那样进行复杂骨骼、法线或光照运算。

这也是 UGUI 能够保持高批处理效率的重要原因之一。

### 17\.2\.7 Fragment Shader 的核心逻辑

Default UI Shader 的 Fragment 阶段主要负责纹理采样、颜色混合以及最终 Alpha 输出。

其核心逻辑通常如下：

```OpenGL Shading Language
fixed4 color = tex2D(_MainTex, IN.texcoord);
color *= IN.color;
return color;
```

整个过程本质上只有两步，首先根据 UV 采样贴图颜色，然后再与顶点颜色相乘，最终输出到 RenderTarget。

虽然逻辑看起来非常简单，但实际上 Alpha 混合、Stencil 裁剪、RectMask2D、Soft Clip 等大量 UI 功能，最终都会在 Fragment 阶段进一步参与计算。因此 UI Fragment Shader 实际上承担了大量最终像素处理工作。

### 17\.2\.8 CanUseSpriteAtlas 的作用

Default UI Shader 中有一个非常容易被忽略的 Tag。

```OpenGL Shading Language
"CanUseSpriteAtlas"="True"
```

这一标记表示当前 Shader 支持 SpriteAtlas。

如果一个自定义 UI Shader 不包含这一 Tag，那么 SpriteAtlas 很可能无法正常工作，同时还可能导致图集合批失效、动态 Atlas 无法正确绑定以及 DrawCall 数量暴增等问题。

因此在实际开发中，自定义 UI Shader 通常必须保留这一配置。

### 17\.2\.9 Default UI Shader 的真正定位

很多开发者第一次阅读 Default UI Shader 时，会觉得它非常简单，因为其内部既没有复杂光照，也没有高级 PBR 计算。

但实际上，Default UI Shader 真正复杂的地方并不在渲染算法，而在于它需要同时兼容整个 UGUI 渲染体系，包括 Canvas Batch、Stencil Mask、RectMask2D、SpriteAtlas、顶点颜色系统以及多平台兼容等大量功能。

因此从本质上来看，Default UI Shader 并不是一个追求高级视觉效果的 Shader，而是一个高度适配 UGUI 架构、强调稳定性与批处理效率的基础渲染 Shader。

## 17\.3 顶点数据传递

在 UGUI 的整个渲染体系中，顶点数据传递是连接 CPU Mesh 构建与 GPU Shader 渲染的核心环节。前面章节已经分析过，UGUI 会在 CPU 阶段通过 VertexHelper 构建 UI Mesh，而 Shader 则会在 GPU 阶段对这些顶点数据进行最终处理。因此，理解 UI Shader 的运行机制，本质上首先需要理解顶点数据究竟是如何从 UGUI 传递到 Shader 的。

从整体流程来看，一个 UI 顶点从生成到最终进入 Fragment Shader，大致会经历 UIVertex、VertexHelper、Mesh、CanvasRenderer、Vertex Shader 以及 Fragment Shader 等多个阶段，而这些阶段共同构成了一条完整的 UI 顶点数据流水线。

### 17\.3\.1 UIVertex 数据结构

UGUI 中所有 UI 顶点最终都会被表示为 UIVertex 结构。

其定义大致如下：

```C#
public struct UIVertex
{
    public Vector3 position;
    public Vector3 normal;
    public Vector4 tangent;
    public Color32 color;
    public Vector4 uv0;
    public Vector4 uv1;
    public Vector4 uv2;
    public Vector4 uv3;
}
```

虽然 UIVertex 中包含多个字段，但绝大多数 UI 实际只会使用 position、color 与 uv0。

其中 position 表示顶点坐标，color 表示顶点颜色，而 uv0 则表示主纹理 UV 坐标。例如一个普通的 Image，本质上只是生成了四个 UIVertex，然后组合为两个三角形，从而形成一个矩形区域。

这些顶点数据最初全部位于 CPU 内存中，并不会立即上传 GPU。

### 17\.3\.2 VertexHelper 顶点管理

当 Graphic 调用 OnPopulateMesh 时，UGUI 并不会直接生成 Mesh，而是先将顶点数据写入 VertexHelper。

例如：

```C#
vh.AddVert(position, color, uv);
```

VertexHelper 本质上是一个专门用于构建 UI Mesh 的顶点管理器，它内部维护了顶点列表、UV 列表、颜色列表以及三角形索引列表。

所有 UI Mesh 最终都会先进入 VertexHelper，然后再统一转换为真正的 Mesh 数据。

这一设计最大的意义在于，UGUI 可以在 Mesh 真正生成之前，继续对顶点数据进行修改。例如 Shadow、Outline、Gradient 等 Mesh Effect，本质上都是直接操作 VertexHelper 中的顶点数据。

因此，VertexHelper 实际上是整个 UGUI Mesh 扩展体系的核心中间层。

### 17\.3\.3 VertexHelper 到 Mesh 的转换

当所有 ModifyMesh 操作执行完成后，VertexHelper 会将内部数据真正写入 Mesh。

其核心流程如下：

```Plain Text
VertexHelper
    ↓
FillMesh(mesh)
    ↓
Mesh
```

在这一阶段中：

- position 会写入 Vertex Buffer 

- uv 会写入 TEXCOORD 

- color 会写入 COLOR 

- triangle 会写入 Index Buffer 

最终形成 GPU 可识别的 Mesh 数据结构。

这里非常重要的一点在于，UGUI 中的所有顶点修改，本质上都发生在 Mesh 上传 GPU 之前。因此 ModifyMesh 属于 CPU 阶段逻辑，而不是 Shader 阶段逻辑。

### 17\.3\.4 CanvasRenderer 的数据提交

当 Mesh 构建完成后，CanvasRenderer 会负责真正提交渲染数据。

例如：

```C#
canvasRenderer.SetMesh(mesh);
canvasRenderer.SetMaterial(material, texture);
```

CanvasRenderer 内部会进一步完成：

- Mesh 绑定 

- Material 绑定 

- Texture 绑定 

- DrawCall 提交 

从 GPU 角度来看，此时真正提交的数据包括 Vertex Buffer、Index Buffer、Texture、Material 与 Shader。

随后 GPU 才会正式开始执行 Vertex Shader。

### 17\.3\.5 顶点数据进入 Shader

当 GPU 开始执行 Vertex Shader 时，Mesh 中的顶点数据会自动映射到 Shader 输入结构。

例如：

```OpenGL Shading Language
struct appdata_t
{
    float4 vertex   : POSITION;
    float4 color    : COLOR;
    float2 texcoord : TEXCOORD0;
};
```

这里的 POSITION 对应 UIVertex\.position，COLOR 对应 UIVertex\.color，而 TEXCOORD0 则对应 UIVertex\.uv0。

GPU 会自动将 Vertex Buffer 中的数据填充到对应语义，因此 Shader 本身并不知道 UIVertex 的存在，它只知道当前顶点包含 POSITION、COLOR 与 TEXCOORD 等标准 GPU 顶点语义。

这也是 GPU 顶点管线的标准工作方式。

### 17\.3\.6 Vertex Shader 中的数据传递

Vertex Shader 的核心职责之一，就是继续向 Fragment Shader 传递数据。

例如：

```OpenGL Shading Language
struct v2f
{
    float4 vertex : SV_POSITION;
    fixed4 color  : COLOR;
    float2 uv     : TEXCOORD0;
};
```

然后在 Vertex Shader 中：

```OpenGL Shading Language
v2f vert(appdata_t IN)
{
    v2f OUT;

    OUT.vertex = UnityObjectToClipPos(IN.vertex);
    OUT.color = IN.color;
    OUT.uv = IN.texcoord;

    return OUT;
}
```

这里主要完成了三个步骤。

首先将 UI 顶点从对象空间转换到裁剪空间，然后将颜色继续向后传递，最后再将 UV 传递给 Fragment Shader。

因此 Vertex Shader 本质上更像是 GPU 顶点数据的中转处理阶段。

### 17\.3\.7 Fragment Shader 的插值数据

当三角形进入光栅化阶段后，GPU 会自动对 Vertex Shader 输出的数据进行插值。

例如顶点 A、B、C 的颜色数据，会在三角形内部自动生成平滑过渡的像素颜色。

因此 Fragment Shader 实际接收到的，并不是原始顶点数据，而是 GPU 插值后的像素级数据。

例如：

```OpenGL Shading Language
fixed4 frag(v2f IN) : SV_Target
{
    fixed4 color = tex2D(_MainTex, IN.uv);
    return color * IN.color;
}
```

这里的 IN\.uv 与 IN\.color 都已经是插值后的结果。

这也是为什么顶点渐变能够形成平滑颜色过渡的根本原因，因为 GPU 在光栅化阶段自动完成了颜色插值。

### 17\.3\.8 UI 顶点数据传递的本质

理解完整个流程后会发现，UGUI 的顶点数据传递，本质上是一条“CPU 构建顶点 → GPU 接收顶点 → GPU 插值顶点 → GPU 输出像素”的完整数据流水线。

其中 CPU 主要负责：

- 几何结构生成 

- 顶点扩展 

- Mesh 修改 

- Vertex 数据组织 

而 GPU 则主要负责：

- 顶点坐标变换 

- UV 传递 

- 插值计算 

- Blend 混合 

- Stencil 测试 

- 最终像素输出 

这种 CPU 与 GPU 明确分工的结构，也是 UGUI 能够同时兼顾灵活性与性能的重要原因。

## 17\.4 Blend 与透明度

在整个 UGUI 渲染体系中，Blend 是最核心的 GPU 渲染机制之一。UI 之所以能够实现半透明、平滑边缘、渐隐动画以及多层颜色叠加，本质上都依赖 Blend 混合机制。前面章节已经分析过，UGUI 默认工作在 Transparent 渲染队列，因此绝大多数 UI 元素都会参与透明混合。而透明混合真正发生的位置，并不在 CPU，也不在 Mesh 阶段，而是在 Fragment Shader 输出像素之后，由 GPU 的 Blend 阶段完成最终颜色合成。

因此从本质上来看，UI 的透明度并不是简单的“颜色变淡”，而是“当前像素与屏幕已有像素进行混合”。

### 17\.4\.1 什么是 Blend

Blend 的本质，是 GPU 将当前像素颜色与屏幕已有颜色进行混合计算。

在 GPU 渲染过程中，当 Fragment Shader 输出颜色后，并不会立即覆盖屏幕，而是会进入 Blend 阶段。此时 GPU 会读取当前 RenderTarget 中已经存在的像素颜色，然后再根据 Blend 规则与当前输出颜色进行计算，最终生成真正写入屏幕的结果。

其中当前输出颜色通常称为 SrcColor，而屏幕已有颜色则称为 DstColor。

Blend 的核心作用，本质上就是决定当前像素应该以什么比例覆盖已有像素。

### 17\.4\.2 Default UI Shader 的 Blend 模式

UGUI 默认使用如下 Blend 设置：

```OpenGL Shading Language
Blend SrcAlpha OneMinusSrcAlpha
```

这是最标准的 Alpha Blend 模式。

其最终计算公式如下：

```OpenGL Shading Language
FinalColor = SrcColor * SrcAlpha + DstColor * (1 - SrcAlpha)
```

其中 SrcColor 表示当前 UI 输出颜色，SrcAlpha 表示当前 UI 的透明度，而 DstColor 则表示屏幕已有颜色。

因此当前 UI 的 Alpha 越高，对屏幕颜色的覆盖能力就越强。

例如：Alpha = 1，表示完全不透明。而Alpha = 0，则表示完全透明。

因此 UI 的透明效果，本质上来自 Blend 对颜色的插值混合。

### 17\.4\.3 UI 为什么必须使用 Blend

传统 3D 模型很多时候可以不使用 Blend，因为不透明物体能够直接覆盖屏幕颜色。

但 UI 不同，UI 天然需要支持半透明、抗锯齿边缘、字体平滑、渐隐动画以及多层颜色叠加，而这些能力全部依赖 Alpha Blend。

例如 Text 字体，其边缘实际上并不是真正的硬边，而是由大量半透明像素组成。如果关闭 Blend：

```OpenGL Shading Language
Blend Off
```

那么字体边缘会立即变得锯齿严重。

因此对于 UI 而言，Blend 并不是附加功能，而是最基础的渲染能力之一。

### 17\.4\.4 Alpha 的真正来源

很多开发者会误认为 UI 的透明度只来自 Image\.color\.a，但实际上并不是。

UI 中的 Alpha 可能来自多个系统，例如 Texture Alpha、Graphic\.color\.a、CanvasGroup Alpha、顶点颜色 Alpha 以及 Shader 内部 Alpha。

最终这些 Alpha 通常会相互叠加。

例如：

```OpenGL Shading Language
finalAlpha = textureAlpha * vertexAlpha * materialAlpha
```

因此一个 UI 元素最终的透明度，往往是多个系统共同作用后的结果。

这也是为什么 CanvasGroup 能够统一控制整个 UI 层级透明度的根本原因，因为它本质上是在继续修改顶点颜色中的 Alpha。

### 17\.4\.5 顶点颜色与透明度

UGUI 中大量透明控制实际上来自顶点颜色系统。

例如：

```OpenGL Shading Language
graphic.color = new Color(1,1,1,0.5f);
```

这里最终会修改：

```OpenGL Shading Language
UIVertex.color.a
```

随后这一 Alpha 会继续传递给 Shader。

最终在 Fragment Shader 中：

```OpenGL Shading Language
color *= IN.color;
```

贴图 Alpha 会与顶点 Alpha 相乘，从而形成最终透明度。

因此 UGUI 中大量透明变化，本质上都依赖顶点颜色系统，而不是 Shader 内部动态计算。

这也是为什么修改 UI 透明度通常不会增加额外 DrawCall。

### 17\.4\.6 Blend 与渲染顺序

Blend 最大的问题之一，在于它强依赖渲染顺序。

因为透明混合需要读取屏幕已有颜色，所以如果绘制顺序错误，那么最终混合结果也会错误。

例如两个半透明 UI：

- 红色 UI

- 蓝色 UI

如果红色先绘制，再绘制蓝色，与蓝色先绘制，再绘制红色，最终结果通常并不相同。

因此透明物体无法像 Opaque 那样随意调整绘制顺序，而这也是 UGUI 必须严格维护 Canvas 顺序与 Hierarchy 顺序的重要原因。

### 17\.4\.7 Premultiplied Alpha

除了标准 Alpha Blend 外，UI 中还有一种非常重要的模式，称为 Premultiplied Alpha。

其 Blend 设置通常如下：

```OpenGL Shading Language
Blend One OneMinusSrcAlpha
```

与普通 Alpha Blend 的区别在于，颜色已经提前乘过 Alpha。

例如普通模式下：

```Plain Text
RGB = (1,1,1)
Alpha = 0.5
```

而 Premultiplied 模式下：

```Plain Text
RGB = (0.5,0.5,0.5)
Alpha = 0.5
```

这种方式能够有效减少：

- 黑边 

- 白边 

- Alpha 锯齿 

因此很多高质量 UI 特效都会采用 Premultiplied Alpha，例如 Spine、粒子系统以及高级 UI 特效。

### 17\.4\.8 Additive Blend

除了普通透明混合外，UI 中还经常使用 Additive Blend。

例如：

```OpenGL Shading Language
Blend One One
```

其计算方式如下：

```OpenGL Shading Language
FinalColor = SrcColor + DstColor
```

这种模式不会降低亮度，而是不断叠加颜色，因此非常适合发光、流光、能量特效以及高亮 UI。

但 Additive Blend 也存在明显问题，因为颜色会不断累加，所以很容易导致：

- 过曝 

- UI 发白 

- 细节丢失 

因此通常只适用于局部视觉特效。

### 17\.4\.9 Blend 与性能

很多开发者会误认为 Blend 的性能消耗很低，但实际上透明混合会带来明显 GPU 压力。

因为 Blend 必须读取屏幕已有颜色，所以 GPU 无法像 Opaque 那样直接覆盖像素，因此透明渲染通常无法充分利用 Early\-Z。

与此同时，大量半透明 UI 重叠还会产生严重 Overdraw。

例如：

- 半透明背景

- 半透明 Glow

- 半透明 Mask

- 半透明粒子

同一区域可能会被反复绘制很多次。

因此 UI 的 GPU 性能瓶颈，很多时候并不是 Shader 本身复杂，而是透明 Overdraw 过高。

### 17\.4\.10 Blend 的本质

理解完整个 Blend 机制后会发现，UI 渲染并不是简单地不断绘制矩形，而是不断将新的颜色混合到已有屏幕颜色之中。

因此：

- 半透明 

- Fade 

- 字体平滑 

- Mask 边缘 

- 发光 

- Additive 特效 

- UI 动态渐变 

这些能力最终全部建立在 GPU Blend Pipeline 之上。

而 Blend 机制，也正是整个 UGUI 透明渲染体系最核心的底层基础。

## 17\.5 Stencil 应用

在整个 UGUI 渲染体系中，Stencil 是最核心的 GPU 裁剪机制之一。Unity 中的 Mask、嵌套裁剪以及大量 UI 特效，本质上都建立在 Stencil Buffer 的基础之上。很多开发者第一次接触 Mask 时，往往会认为它只是一个简单的“区域隐藏”功能，但实际上 UGUI 并不是通过 CPU 删除 Mesh，也不是通过 Shader 动态切割几何结构来实现裁剪，而是利用 GPU 的 Stencil Buffer 完成像素级过滤。

因此从本质上来看，Mask 并不是简单地隐藏 UI，而是阻止某些像素通过 GPU 的渲染测试。

### 17\.5\.1 什么是 Stencil Buffer

Stencil Buffer 本质上是一块额外的 GPU 缓冲区。

它与 Color Buffer、Depth Buffer 类似，但存储的并不是颜色或深度，而是一个整数值。通常情况下，Stencil Buffer 每个像素会存储一个 8 bit 数据，也就是：0 \~ 255

GPU 在渲染过程中，可以对这块缓冲区进行读取、写入、比较以及修改。

因此从本质上来看，Stencil Buffer 更像是一套 GPU 像素标记系统。

### 17\.5\.2 Stencil 的基本工作流程

Stencil 的工作流程通常分为两个阶段。

第一阶段负责写入标记。

第二阶段负责读取标记并决定当前像素是否允许渲染。

例如：

```Plain Text
Mask 写入 Stencil = 1
        ↓
子 UI 检测 Stencil == 1
        ↓
允许渲染
```

而不在 Mask 区域内的像素，由于：

```Plain Text
Stencil != 1
```

因此会被 GPU 丢弃。

所以从本质上来看，Mask 实际上是一种 GPU 像素过滤机制。

### 17\.5\.3 UGUI 中的 Mask 原理

Unity 中的 Mask 组件，本质上就是基于 Stencil 实现的。

当一个 UI 添加 Mask 后，Unity 会自动修改其 Material 的 Stencil 参数。

例如：

```Plain Text
Stencil
{
    Ref 1
    Comp Always
    Pass Replace
}
```

这里：

- Ref 1 表示写入值 1 

- Comp Always 表示始终通过测试 

- Pass Replace 表示将当前值替换为 Ref 

因此 Mask 在渲染时，会将自身区域对应的 Stencil Buffer 写入：1

但需要注意的是，Mask 本身很多时候并不负责真正显示内容，它更重要的职责其实是向 GPU 写入一块可用于后续测试的区域标记。

### 17\.5\.4 子 UI 的 Stencil 测试

当 Mask 的子节点开始渲染时，其 Shader 会自动增加对应的 Stencil 测试。

例如：

```Plain Text
Stencil
{
    Ref 1
    Comp Equal
}
```

这里表示：

只有当：

```Plain Text
Stencil == 1
```

当前像素才允许绘制。

因此只有位于 Mask 区域内部的像素，才能真正显示，而区域外部的像素即使 Fragment Shader 已经输出颜色，也会在最终写入屏幕之前被 GPU 丢弃。

### 17\.5\.5 Stencil 的执行位置

Stencil Test 发生在 Fragment Shader 之后，但在最终写入 RenderTarget 之前。

其整体流程大致如下：

```Plain Text
Vertex Shader
        ↓
Fragment Shader
        ↓
Stencil Test
        ↓
Blend
        ↓
写入屏幕
```

因此即使 Fragment Shader 已经完成颜色计算，只要 Stencil Test 不通过，当前像素依然不会真正显示。

这也是为什么 Stencil 能够实现高效 GPU 裁剪，因为它根本不需要 CPU 修改 Mesh。

### 17\.5\.6 为什么 Stencil 比 Mesh 裁剪更高效

很多开发者会误认为 Mask 是通过动态裁剪 Mesh 实现的，但实际上 UGUI 并不会真正切割 UI Mesh。

因为动态 Mesh 裁剪成本极高。

例如：

- 需要重新生成顶点 

- 需要重新计算 UV 

- 需要重新构建三角形 

而 Stencil 完全工作在 GPU 像素阶段，它不修改 Mesh，也不修改顶点，而只是决定当前像素是否允许写入屏幕。

因此相比动态 Mesh 裁剪，Stencil 的效率通常更高。

### 17\.5\.7 嵌套 Mask 的实现

UGUI 支持多层 Mask 嵌套。

例如：

```Plain Text
Mask A
    └── Mask B
            └── UI
```

其底层本质上仍然依赖 Stencil Buffer。

Unity 会不断增加 Ref 值。

例如：

```Plain Text
Mask A → Ref 1
Mask B → Ref 2
```

随后子节点必须满足：

```Plain Text
Stencil == 2
```

才允许渲染。

因此 Stencil Buffer 本质上形成了一套层级式 GPU 裁剪系统。

### 17\.5\.8 Stencil 的位运算机制

由于 Stencil Buffer 通常只有 8 bit，因此 UGUI 实际上会通过位运算管理 Mask 层级。

例如：

```Plain Text
00000001
00000010
00000100
```

每一层 Mask 会占用一个 bit。

因此 UGUI Mask 理论最大嵌套层级通常为：8 层

超过后，由于 Stencil 位数不足，Unity 就无法继续正确管理 Mask。

这也是为什么 UGUI 深层嵌套 Mask 容易出现异常显示。

### 17\.5\.9 RectMask2D 与 Stencil 的区别

很多开发者会误认为 RectMask2D 同样基于 Stencil 实现，但实际上并不是。

RectMask2D 通常基于 Rect 范围裁剪实现，它不会真正写入 Stencil Buffer，而是在 Shader 中直接判断当前像素是否位于矩形区域内。

因此 RectMask2D 通常比普通 Mask 更高效，因为它避免了：

- Stencil 写入 

- Stencil Test 

- 多层 Stencil 嵌套 

但 RectMask2D 也存在明显限制，因为它只能裁剪矩形区域，而无法实现：

- 圆形裁剪 

- 不规则 Mask 

- 图片形状裁剪 

因此普通 Mask 与 RectMask2D 实际上适用于不同场景。

### 17\.5\.10 Stencil 与 UI 特效

Stencil 不仅仅用于 Mask。

实际上大量 UI 特效也会依赖 Stencil。

例如：

- 镂空遮罩 

- 引导高亮 

- 区域模糊 

- 聚光效果 

- 局部显示 

这些效果本质上都是先写入 Stencil，然后再基于 Stencil 控制后续区域是否允许渲染。

因此从本质上来看，Stencil 更像是一套 GPU 区域控制系统。

### 17\.5\.11 Stencil 的性能问题

虽然 Stencil 非常高效，但它仍然存在一定性能成本。

因为 Stencil Test 属于 GPU 像素阶段操作，因此大量：

- 半透明 UI 

- 多层 Mask 

- 大面积覆盖 

仍然可能导致严重 Overdraw。

与此同时，不同的 Stencil 状态还会影响 Batch。

例如：

```Plain Text
Ref
Comp
Pass
```

只要这些状态不同，就可能导致 DrawCall 被拆分。

因此复杂的 Mask 系统通常会明显增加 UI DrawCall 数量。

### 17\.5\.12 Stencil 的本质

理解完整个 Stencil 机制后会发现，UGUI 的 Mask 系统本质上并不是“UI 裁剪系统”，而是一套 GPU 像素过滤系统。

它不会真正修改 Mesh，也不会真正删除顶点，而是利用 GPU 的 Stencil Buffer，在最终输出阶段决定哪些像素允许显示。

因此：

- Mask 

- 镂空 

- 引导遮罩 

- 区域高亮 

- 局部显示 

- UI 聚光效果 

这些能力，本质上全部建立在 Stencil Buffer 之上。

而 Stencil，也正是整个 UGUI 裁剪体系最核心的底层机制之一。

## 17\.6 UI 与 3D Shader 差异

虽然 UGUI Shader 与传统 3D Shader 在底层都运行于 GPU 渲染管线之中，并且同样由 Vertex Shader 与 Fragment Shader 组成，但两者的设计目标却存在本质区别。3D Shader 更关注空间表现、光照计算以及物理渲染，而 UI Shader 则更强调透明混合、批处理效率以及二维界面表现。因此从整体架构上来看，UI Shader 与 3D Shader 虽然共享同一套 GPU 渲染体系，但它们实际解决的是两类完全不同的问题。

### 17\.6\.1 渲染目标差异

3D Shader 的核心目标是构建真实空间中的物体表现，因此它通常需要处理光照、阴影、法线、反射、PBR 以及深度关系等问题。例如一个 3D 模型，其 Shader 需要不断计算光线方向、法线方向、阴影衰减以及 BRDF，从而尽可能还原真实物理效果。

而 UI Shader 不同。

UI Shader 的核心目标并不是还原真实世界，而是保证界面元素的正确显示与高效渲染。因此 UI Shader 更关注透明混合、Mask 裁剪、顶点颜色、图集支持以及 Canvas Batch 等问题。

所以从本质上来看，3D Shader 更偏向空间渲染，而 UI Shader 更偏向界面合成。

### 17\.6\.2 顶点结构差异

3D Shader 通常需要大量顶点数据。

例如：

```Plain Text
POSITION
NORMAL
TANGENT
TEXCOORD
COLOR
BONEWEIGHT
BONEINDEX
```

因为 3D 渲染需要进行法线计算、切线空间计算、骨骼动画以及复杂光照运算。

而 UGUI Shader 的顶点结构通常非常简单。

例如：

```Plain Text
POSITION
COLOR
TEXCOORD0
```

大多数 UI 根本不需要法线、骨骼以及切线空间，因为 UI 本质上只是二维平面。

因此 UI 顶点数据通常远小于 3D Mesh。

### 17\.6\.3 Vertex Shader 计算差异

3D Shader 的 Vertex 阶段通常非常复杂。

例如：

- 骨骼矩阵计算 

- BlendShape 

- 法线变换 

- 阴影坐标计算 

- 世界空间转换 

这些都会明显增加 Vertex Cost。

而 UI Shader 的 Vertex 阶段通常极其轻量。

例如：

```OpenGL Shading Language
OUT.vertex = UnityObjectToClipPos(IN.vertex);
OUT.uv = IN.texcoord;
OUT.color = IN.color;
```

很多情况下，UI Vertex Shader 甚至只负责坐标转换、UV 传递以及颜色传递。

因为 UGUI 的大量几何处理，其实已经在 CPU 阶段完成。

因此 UI Shader 的 Vertex Cost 通常远低于 3D Shader。

### 17\.6\.4 Fragment Shader 差异

3D Shader 的 Fragment 阶段通常会进行复杂光照计算。

例如：

- PBR 

- GI 

- Shadow 

- Reflection 

- Fresnel 

而 UI Shader 的 Fragment 阶段则通常更加简单。

例如：

```OpenGL Shading Language
fixed4 color = tex2D(_MainTex, IN.uv);
return color * IN.color;
```

UI Fragment Shader 更多负责纹理采样、Alpha 混合、Mask 裁剪以及顶点颜色处理，而不是复杂光照。

因此 UI Shader 的像素计算通常比 3D Shader 更轻。

但需要注意的是，UI 往往会产生极高 Overdraw，因此 UI 的 GPU 压力很多时候来自像素重复绘制，而不是 Shader 本身复杂。

### 17\.6\.5 深度缓冲差异

3D Shader 极度依赖 Depth Buffer，因为 3D 世界必须正确处理前后遮挡、空间深度以及 Early\-Z。

因此大多数 3D Shader 默认会开启：

```OpenGL Shading Language
ZWrite On
ZTest LEqual
```

而 UI Shader 则通常关闭深度写入。

例如：

```OpenGL Shading Language
ZWrite Off
```

因为 UI 更依赖 Canvas 顺序、Hierarchy 顺序以及 Sorting Order，而不是空间深度。

因此 UGUI 本质上是一套基于绘制顺序的渲染系统，而不是基于空间深度的渲染系统。

### 17\.6\.6 Blend 使用差异

3D Shader 中，大量物体属于 Opaque，因此不需要 Blend，可以直接覆盖像素，同时还能充分利用 Early\-Z。

而 UI Shader 几乎天然依赖 Blend。

例如：

```OpenGL Shading Language
Blend SrcAlpha OneMinusSrcAlpha
```

因为字体抗锯齿、半透明 Image、Fade 动画以及 UI 特效全部依赖 Alpha Blend。

因此 Blend 对 UI 来说属于基础能力，而对 3D Shader 来说更多属于特殊效果。

### 17\.6\.7 Stencil 使用差异

3D Shader 中虽然也会使用 Stencil，但通常只用于延迟渲染、后处理或者特殊效果。

而在 UGUI 中，Stencil 属于核心系统。

因为：

- Mask 

- UI 裁剪 

- 引导遮罩 

- 镂空区域 

全部依赖 Stencil。

因此 Stencil 在 UI 中的重要性远高于普通 3D 渲染。

### 17\.6\.8 Batch 机制差异

3D 渲染中的 Batch 通常依赖 Static Batch、Dynamic Batch 以及 GPU Instancing。

而 UGUI 更依赖 Canvas Batch、Texture Atlas 以及 Material 合并。

因此 UI Shader 必须尽可能避免：

- 材质切换 

- Texture 切换 

- Stencil 状态变化 

否则 DrawCall 会快速增加。

所以 UI Shader 会更加强调统一渲染状态。

### 17\.6\.9 坐标体系差异

3D Shader 主要处理：

- Object Space

- World Space

- View Space

- Clip Space

而 UGUI 大多数情况下只需要处理：

- Local Space

- Canvas Space

- Clip Space

因为 UI 本身通常位于二维平面，因此 UI Shader 很少需要复杂空间变换。

### 17\.6\.10 性能瓶颈差异

3D Shader 的性能瓶颈通常来自：

- 光照计算 

- 阴影 

- PBR 

- 顶点数量 

- 后处理 

而 UI Shader 的性能瓶颈则通常来自：

- Overdraw 

- DrawCall 

- Mask 

- Alpha Blend 

- 大面积透明覆盖 

因此 UI 性能优化与 3D 性能优化，往往是两套完全不同的思路。

### 17\.6\.11 UI Shader 的本质定位

理解完整个差异后会发现，UI Shader 本质上并不是简化版 3D Shader，因为两者从设计目标开始就完全不同。

3D Shader 更关注如何真实渲染空间物体，而 UI Shader 更关注如何高效合成二维界面。

因此：

- Blend 

- Stencil 

- Canvas Batch 

- 顶点颜色 

- 图集 

这些系统才真正构成了 UI Shader 的核心。

而这也意味着，很多适用于 3D Shader 的优化方案，并不一定适用于 UI Shader。整个 UGUI Shader 体系，本质上是一套专门为二维界面渲染而设计的 GPU 渲染架构。

## 本章小结

本章围绕 UGUI 的 Shader 机制展开，重点分析了 UI Shader 在整个渲染体系中的结构组成、数据传递方式以及透明混合与裁剪机制，并进一步对比了 UI Shader 与传统 3D Shader 在底层设计上的核心差异。

首先，在 UI Shader 结构部分，可以看到 UGUI Shader 并不是一个独立于 Unity 渲染体系之外的特殊系统，它本质上仍然建立在标准 GPU 渲染管线之上，同样由 Vertex Shader 与 Fragment Shader 构成。但由于 UI 系统更强调透明混合、批处理效率以及二维界面显示，因此其 Shader 结构相比传统 3D Shader 更加轻量。

在 Default UI Shader 的分析中，可以进一步看到 UGUI 默认 Shader 的核心目标并不是实现复杂光照，而是高度适配 Canvas 渲染体系。其内部大量设计都围绕 Blend、Stencil、顶点颜色、SpriteAtlas 以及 Batch 合并展开，因此它本质上是一套专门服务于 UI 渲染的基础 Shader。

在顶点数据传递部分，可以看到 UGUI 的整个数据流实际上是一条从 CPU 到 GPU 的完整顶点流水线。UIVertex 在 CPU 阶段生成后，会经过 VertexHelper、Mesh 与 CanvasRenderer 逐步提交到 GPU，随后再进入 Vertex Shader 与 Fragment Shader 进行最终处理。这种 CPU 与 GPU 明确分工的结构，也是 UGUI 能够同时兼顾灵活性与渲染效率的重要原因。

Blend 与透明度部分则揭示了 UI 半透明渲染的底层本质。UI 的透明效果并不是简单的颜色变化，而是 GPU 在 Blend 阶段对当前像素与屏幕已有像素进行混合计算的结果。无论是字体抗锯齿、Fade 动画还是 Additive 特效，本质上都建立在透明混合机制之上。同时也可以看到，UI 渲染中的大量性能问题实际上并不是来自 Shader 本身，而是来自透明 Overdraw。

在 Stencil 应用部分，可以看到 UGUI 的 Mask 系统本质上是一套 GPU 像素过滤机制。Unity 并不会真正裁剪 Mesh，而是通过 Stencil Buffer 在最终输出阶段决定哪些像素允许显示。因此无论是普通 Mask、嵌套裁剪还是引导遮罩，其底层都建立在 Stencil Buffer 的区域控制能力之上。

最后，在 UI 与 3D Shader 的差异部分，可以明确看到两者虽然共享同一套 GPU 渲染管线，但其设计目标却完全不同。3D Shader 更强调空间渲染、物理光照以及深度关系，而 UI Shader 更强调透明合成、Batch 效率以及界面表达。因此很多适用于 3D 渲染的优化方案，并不一定适用于 UI 系统。

从整体架构来看，UGUI Shader 体系的核心价值，在于它将 GPU 渲染能力高度适配到了二维界面系统之中，使 UI 不再只是简单的纹理绘制，而成为一套具备透明混合、像素裁剪、顶点控制以及 GPU 合成能力的完整渲染体系。

