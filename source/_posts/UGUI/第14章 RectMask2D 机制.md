---
title: 第14章 RectMask2D 机制
categories:
  - Unity
  - UGUI
tags:
  - UGUI
  - Unity
cover: /images/UGUI/UGUI.png
abbrlink: 1315b2ea
date: 2026-08-26 00:00:14
---
# 第14章 RectMask2D 机制

在UGUI系统中，除了基于Stencil Buffer实现的Mask之外，Unity还提供了一种更加轻量级的裁剪方案——RectMask2D。它主要用于矩形区域的UI裁剪，在ScrollView、列表视图以及窗口界面中被大量使用。

与Mask依赖GPU模板测试不同，RectMask2D采用的是基于矩形区域的裁剪机制，其核心思想是在渲染前限制UI元素的可见范围，从而减少不必要的像素处理开销。由于不依赖Stencil Buffer，因此它在批处理保持、DrawCall控制以及GPU状态切换方面具有更好的性能表现。

RectMask2D虽然无法实现任意形状裁剪，但其简单稳定的裁剪模型非常适合大多数UI界面需求。尤其是在大量动态列表、滚动区域以及复杂界面层级中，RectMask2D通常比Mask具有更高的运行效率。

本章将从RectMask2D的底层工作原理开始，分析其在Canvas系统中的裁剪实现方式，并对比其与Mask在渲染流程和性能层面的差异，最后结合实际项目场景分析RectMask2D的适用范围与使用策略。

## 14\.1 工作原理

RectMask2D是UGUI中用于实现矩形区域裁剪的一种轻量级机制。与Mask依赖Stencil Buffer进行像素级测试不同，RectMask2D采用的是基于矩形范围的裁剪方式，其核心目标是在尽可能减少GPU额外开销的前提下，实现对子UI可见区域的限制。

从整体实现来看，RectMask2D并不是通过修改UI几何结构来完成裁剪，而是通过向CanvasRenderer传递裁剪区域，并在渲染阶段结合GPU裁剪状态完成可见性控制。

### 14\.1\.1 RectMask2D的基本结构

一个RectMask2D通常由以下几个部分组成：

- 作为裁剪区域的RectTransform

- 挂载RectMask2D组件的节点

- 受到裁剪影响的子UI对象

- CanvasRenderer中的裁剪状态数据

其中，RectTransform决定裁剪矩形的空间范围，而RectMask2D负责将该范围转换为最终的裁剪区域数据，并通知所有受影响的子节点更新裁剪状态。

与Mask不同的是，RectMask2D不需要Image组件作为遮罩图形，因为它只支持规则矩形裁剪。

### 14\.1\.2 RectMask2D的裁剪区域计算

RectMask2D的核心首先是裁剪矩形的计算。

在Canvas更新阶段，RectMask2D会根据自身RectTransform计算出一个世界空间下的矩形区域。这个区域通常会经过以下步骤：

- 获取RectTransform四个顶点

- 转换到Canvas空间

- 计算最终包围矩形

- 与父级裁剪区域求交集

最终得到一个有效裁剪区域。

如果存在多层RectMask2D嵌套，则每一层都会与父级裁剪区域进行求交运算，从而不断缩小最终可见范围。

因此，RectMask2D本质上是一种“矩形区域传播机制”。

### 14\.1\.3 裁剪信息的传播过程

当RectMask2D更新完成后，会向所有受影响的子节点发送裁剪更新通知。

在UGUI源码中，这一过程主要通过以下接口完成：

- IClippable

- MaskableGraphic

- CanvasRenderer\.EnableRectClipping

其中：

- RectMask2D负责管理裁剪区域

- MaskableGraphic负责接收裁剪通知

- CanvasRenderer负责真正执行裁剪状态设置

整个流程如下：

1. RectMask2D计算裁剪矩形

2. 遍历所有可裁剪子节点

3. 调用SetClipRect更新裁剪区域

4. CanvasRenderer启用Rect Clipping状态

5. GPU根据裁剪矩形丢弃超出区域的片元

可以看到，RectMask2D本质上仍然会影响GPU片元输出，但它不需要Stencil Buffer参与。

### 14\.1\.4 CanvasRenderer中的裁剪机制

RectMask2D真正生效的位置是在CanvasRenderer阶段。

当CanvasRenderer接收到裁剪矩形后，会向底层渲染系统提交一个矩形裁剪区域。GPU在执行片元输出时，会自动判断当前像素是否位于该矩形内部。

如果片元超出裁剪范围，则直接被丢弃。

这一机制通常依赖GPU的Scissor Test（裁剪测试）或等价实现。

与Stencil测试相比，矩形裁剪具有以下特点：

- 无需Stencil Buffer读写

- 不需要额外模板状态

- 不会增加Stencil状态切换

- 裁剪判断逻辑更简单

因此RectMask2D的GPU成本明显更低。

### 14\.1\.5 RectMask2D与Transform变化

RectMask2D会持续监听自身以及父级Transform变化。

当以下情况发生时：

- RectTransform尺寸变化

- 锚点变化

- 父级移动

- Canvas缩放变化

RectMask2D都会重新计算裁剪区域，并通知子节点刷新裁剪状态。

因此，在动态布局频繁变化的界面中，RectMask2D会产生一定的CPU更新成本。

不过相比Stencil Mask带来的GPU成本，这部分CPU开销通常更容易控制。

### 14\.1\.6 RectMask2D对子节点的影响

RectMask2D不会真正修改子节点的顶点数据，也不会裁剪Mesh。

子节点仍然会正常生成完整几何结构，只是在最终片元输出阶段，超出矩形区域的像素被丢弃。

这意味着：

- 顶点数量不会减少

- UI布局不会改变

- Batch结构通常可以保持

- 片元填充区域会减少

因此RectMask2D虽然不能减少顶点处理成本，但能够有效降低无效像素输出。

### 14\.1\.7 多层RectMask2D嵌套

RectMask2D支持嵌套使用。

在嵌套情况下，每一层都会与父级区域进行矩形求交，最终形成一个更小的有效裁剪区域。

例如：

- 父级区域为500×500

- 子级区域为300×300

- 最终裁剪区域为两者重叠部分

由于整个过程始终基于矩形运算，因此即使多层嵌套，其计算复杂度仍然较低。

相比Stencil Mask的多层状态叠加，RectMask2D在嵌套场景中的性能更加稳定。

### 14\.1\.8 RectMask2D的局限性

RectMask2D虽然性能优秀，但存在明显限制。

首先，它只能支持矩形裁剪。

其次，它无法支持：

- 圆形遮罩

- 不规则图片遮罩

- 透明通道裁剪

- 复杂形状UI裁剪

此外，由于裁剪区域基于RectTransform计算，因此旋转后的矩形通常无法得到精确裁剪结果。

这也是为什么复杂视觉效果仍然需要使用Mask系统。

### 14\.1\.9 小结

RectMask2D是一种基于矩形区域裁剪的轻量级UI裁剪机制。它通过计算RectTransform裁剪区域，并利用CanvasRenderer向GPU提交矩形裁剪状态，实现对子UI可见范围的限制。由于不依赖Stencil Buffer，因此RectMask2D在Batch保持、DrawCall控制以及GPU状态切换方面具有明显性能优势，但同时也受限于只能处理规则矩形裁剪。

## 14\.2 与 Mask 性能对比

在UGUI系统中，Mask与RectMask2D虽然都用于实现UI裁剪，但两者的底层实现方式完全不同，因此在CPU开销、GPU开销、批处理影响以及适用场景等方面也存在明显差异。

理解二者的性能区别，不仅有助于优化UI渲染效率，也能够帮助开发者在项目中正确选择裁剪方案。

### 14\.2\.1 底层实现方式差异

Mask的核心依赖Stencil Buffer。

它通过在GPU模板缓冲区写入标记值，并在后续片元阶段进行Stencil测试，实现像素级裁剪。

而RectMask2D则完全不依赖Stencil Buffer。

它基于矩形裁剪区域，通过CanvasRenderer向GPU提交矩形裁剪状态，并利用Scissor Test或等价机制直接限制片元输出范围。

因此，两者最大的区别在于：

- Mask属于“模板测试型裁剪”

- RectMask2D属于“矩形区域型裁剪”

这一差异决定了后续所有性能表现。

### 14\.2\.2 GPU开销对比

从GPU角度来看，Mask的开销明显高于RectMask2D。

Mask在渲染过程中通常需要：

- 执行Stencil Buffer写入

- 执行Stencil Buffer读取

- 进行Stencil Comparison测试

- 维护Stencil状态切换

- 处理多层Stencil嵌套逻辑

这些操作都会增加片元阶段的负担。

尤其是在高分辨率UI界面中，Stencil测试会随着像素数量增加而明显放大成本。

相比之下，RectMask2D只需要执行简单的矩形范围判断。

GPU只需判断当前片元是否位于裁剪矩形内部，而无需进行模板缓冲区读写，因此其执行成本更低。

从片元阶段复杂度来看：RectMask2D \< 单层Mask \< 多层嵌套Mask

这是UGUI中非常典型的GPU性能层级关系。

### 14\.2\.3 对Batch合并的影响

批处理是UGUI性能优化中的核心机制之一。

Mask会直接影响Batch合并。

因为不同Mask层级通常意味着不同Stencil状态，而GPU状态不同会导致DrawCall拆分。

例如：

- Stencil Ref不同

- Comparison Function不同

- Write Mask不同

都会导致无法继续Batch合并。

因此在复杂Mask嵌套界面中，DrawCall数量通常会快速增长。

而RectMask2D由于不依赖Stencil状态，因此对Batch影响较小。

多数情况下：

- 相同材质

- 相同纹理

- 相同Shader

仍然可以保持合批。

这也是为什么ScrollView大量采用RectMask2D实现，而不是Stencil Mask。

### 14\.2\.4 CPU开销对比

在CPU侧，两者的性能结构则有所不同。

Mask本身主要依赖GPU，因此CPU开销相对较低。

它更多影响的是：

- 材质状态管理

- Stencil层级计算

- Canvas重建时的状态更新

而RectMask2D则需要频繁计算裁剪矩形。

尤其在以下情况下：

- 界面频繁移动

- RectTransform动态变化

- 滚动区域持续刷新

- 多层嵌套RectMask2D

都会导致裁剪区域重新计算。

因此RectMask2D的CPU开销通常高于Mask。

不过这种CPU开销一般远低于Stencil带来的GPU压力，因此整体仍然更高效。

### 14\.2\.5 FillRate影响对比

FillRate指GPU每帧能够处理的像素填充能力。

Mask由于依赖Stencil测试，因此即使像素最终被裁剪，GPU仍然需要：

- 执行片元着色

- 执行Stencil比较

- 再决定是否丢弃

这意味着大量无效像素仍然参与了部分GPU流程。

而RectMask2D由于矩形裁剪更早生效，因此很多超出区域的片元会更快被剔除。

因此：RectMask2D通常拥有更好的FillRate表现。

在移动端设备上，这一差异尤为明显。

### 14\.2\.6 嵌套场景性能差异

当出现多层裁剪嵌套时，两者差异会进一步放大。

Mask嵌套会导致：

- Stencil值递增

- 模板状态复杂化

- Shader状态切换增加

- DrawCall拆分加剧

嵌套越深，性能问题越明显。

而RectMask2D嵌套本质上只是矩形求交运算。

即使多层嵌套，其GPU状态依然相对稳定。

因此在复杂滚动列表中RectMask2D通常比Mask稳定得多。

### 14\.2\.7 内存与缓冲区影响

Mask依赖Stencil Buffer。

这意味着GPU必须维护额外模板缓冲区。

在部分平台上：

- Stencil与Depth共享缓冲

- Stencil格式可能影响显存占用

- 多重采样下Stencil成本会进一步增加

而RectMask2D不依赖Stencil Buffer，因此不会增加模板缓冲区压力。

这也是移动平台更倾向RectMask2D的重要原因。

### 14\.2\.8 实际项目中的选择策略

在实际项目中，选择Mask还是RectMask2D，本质上是在“表达能力”与“性能效率”之间做权衡。

适合使用Mask的场景：

- 不规则图片遮罩

- 圆形头像裁剪

- 复杂UI形状

- 特殊视觉效果

- 透明通道遮罩

适合使用RectMask2D的场景：

- ScrollView

- 聊天列表

- 背包列表

- 排行榜

- 普通窗口裁剪

- 大规模动态UI

通常情况下，只要矩形裁剪能够满足需求，就优先使用RectMask2D。

只有在确实需要复杂形状裁剪时，才使用Stencil Mask。

### 14\.2\.9 小结

Mask与RectMask2D虽然都用于UI裁剪，但两者在底层实现与性能结构上存在根本差异。Mask基于Stencil Buffer实现任意形状裁剪，拥有更强表达能力，但会增加GPU片元开销、Stencil状态切换以及DrawCall拆分。而RectMask2D基于矩形区域裁剪，不依赖Stencil Buffer，因此在Batch保持、FillRate控制以及嵌套稳定性方面具有明显优势，更适合高性能UI场景。

## 14\.3 使用场景

在UGUI实际开发中，Mask与RectMask2D并不存在绝对意义上的“谁更好”，二者的选择本质上取决于界面的视觉需求与性能目标。

由于RectMask2D与Mask在底层机制上的差异，它们分别适用于不同类型的UI结构。合理选择裁剪方案，不仅能够提升渲染效率，还能够降低Canvas重建与GPU状态切换带来的额外开销。

### 14\.3\.1 ScrollView中的RectMask2D

RectMask2D最典型的应用场景就是ScrollView。

在滚动列表中，通常只需要限制内容显示在一个矩形区域内部，并不需要复杂形状裁剪。因此RectMask2D天然适合这类场景。

例如：

- 聊天列表

- 排行榜

- 背包界面

- 邮件列表

- 任务列表

- 好友列表

这些界面的共同特点是：

- 内容数量较大

- 元素更新频繁

- 滚动操作频繁

- 裁剪区域规则稳定

如果使用Stencil Mask，在滚动过程中会不断产生额外的GPU片元测试开销，并影响Batch合并效率。

而RectMask2D由于只进行矩形裁剪，因此能够在保持较低DrawCall数量的同时，实现更稳定的滚动性能。

这也是Unity默认ScrollRect组件内部使用RectMask2D的重要原因。

### 14\.3\.2 大规模动态UI中的RectMask2D

在大型UI系统中，经常会存在大量动态刷新的界面。

例如：

- 商城界面

- 卡牌列表

- 技能树

- 成就系统

- 角色属性面板

这些界面通常包含：

- 大量UI元素

- 频繁激活与隐藏

- 持续布局变化

- 动态加载内容

在这种情况下，Stencil Mask的状态切换成本会被进一步放大。

尤其在移动端设备上：

- GPU FillRate压力

- Stencil Buffer读写

- DrawCall拆分

都可能成为性能瓶颈。

因此，只要界面允许使用矩形裁剪，就应优先采用RectMask2D。

### 14\.3\.3 不规则UI中的Mask

虽然RectMask2D性能优秀，但它只能处理规则矩形。

当UI需要复杂形状裁剪时，就必须使用Mask。

典型场景包括：

- 圆形头像

- 异形按钮

- 不规则窗口

- 图片透明区域遮罩

- 复杂视觉特效UI

例如圆形头像。

由于头像区域并不是矩形，因此RectMask2D无法实现真正的圆形裁剪，而Mask可以通过Image透明区域实现精确遮罩。

在这种情况下，Stencil Buffer提供的像素级控制能力是不可替代的。

### 14\.3\.4 特殊视觉效果中的Mask

某些UI效果本身就依赖Stencil机制。

例如：

- 动态镂空效果

- UI擦除效果

- 渐变遮罩

- 复杂动画过渡

- 多层视觉叠加效果

这些效果往往需要：

- 像素级裁剪控制

- 透明区域参与遮罩

- 动态变化的可见区域

RectMask2D无法处理这些复杂情况。

因此在高级UI表现中，Mask依然具有重要价值。

### 14\.3\.5 嵌套裁剪场景的选择

在复杂UI层级中，经常会出现多层裁剪嵌套。

例如：

- ScrollView内部嵌套子窗口

- 分页区域内嵌滚动列表

- 背包界面中的装备详情面板

此时需要特别注意裁剪方案的选择。

如果使用多层Stencil Mask：

- Stencil状态会持续叠加

- DrawCall拆分明显增加

- GPU状态切换频繁

性能会快速下降。

而RectMask2D的嵌套本质上只是矩形求交运算，因此更加稳定。

因此：多层嵌套列表优先使用RectMask2D，复杂视觉嵌套才使用Mask。

这是UI性能优化中的重要原则。

### 14\.3\.6 移动平台中的选择策略

在移动设备上，Mask与RectMask2D的差异会更加明显。

移动GPU通常具有以下特点：

- FillRate较低

- 带宽有限

- Stencil Buffer成本较高

- 状态切换敏感

因此Stencil Mask在移动端更容易产生性能问题。

尤其是在：

- 全屏UI

- 大量半透明元素

- 复杂HUD系统

- 高分辨率界面

中，Stencil带来的GPU压力会明显增加。

因此移动平台通常遵循以下策略：

- 能用RectMask2D就不用Mask

- 能减少嵌套就减少嵌套

- 能避免Stencil就避免Stencil

只有在视觉需求无法替代时，才使用Mask。

### 14\.3\.7 编辑器与运行时差异

需要注意的是，Mask在编辑器中的表现有时与真机不同。

由于PC GPU性能较强，因此Stencil开销往往不明显。但在移动设备上：

- DrawCall增长

- FillRate下降

- GPU占用升高

会更加明显。

因此很多UI在编辑器中运行流畅，但在移动端会出现明显掉帧。

RectMask2D由于结构更简单，因此跨平台稳定性通常更高。

### 14\.3\.8 实际项目中的推荐原则

在实际项目中，通常可以遵循以下原则：

- 普通矩形裁剪优先使用RectMask2D

- 滚动列表始终优先RectMask2D

- 复杂形状裁剪使用Mask

- 避免多层Stencil嵌套

- 减少复杂Mask层级

- 移动端尽量减少Stencil使用

很多成熟项目甚至会制定UI规范：

- 禁止在ScrollView中使用Mask

- 禁止超过两层Stencil嵌套

- 普通头像统一使用RectMask2D \+ 圆角Shader替代Mask

这些规范本质上都是为了降低GPU压力。

### 14\.3\.9 小结

RectMask2D与Mask分别适用于不同类型的UI场景。RectMask2D更适合矩形区域裁剪、大规模动态列表以及高性能UI系统，而Mask则适用于复杂形状裁剪与特殊视觉效果。在实际项目中，应优先选择RectMask2D以降低GPU负担，仅在确实需要像素级遮罩能力时使用Stencil Mask，从而在视觉效果与渲染性能之间取得合理平衡。

## 本章小结

本章围绕RectMask2D机制展开分析，重点讨论了其底层裁剪原理、与Stencil Mask之间的性能差异，以及在实际项目中的适用场景。

RectMask2D本质上是一种基于矩形区域的轻量级裁剪机制。它通过计算RectTransform对应的裁剪区域，并将该区域传递给CanvasRenderer，在GPU片元阶段利用矩形裁剪状态限制像素输出。整个过程不依赖Stencil Buffer，因此避免了模板缓冲区读写与Stencil状态切换带来的额外开销。

与Mask相比，RectMask2D最大的特点是性能稳定。由于其不依赖Stencil测试，因此不会显著破坏Batch合并结构，也不会产生复杂的DrawCall拆分问题。在大量动态UI、滚动列表以及高频刷新界面中，RectMask2D通常能够获得更好的GPU表现与更低的FillRate压力。

与此同时，RectMask2D也存在明显局限性。由于其裁剪模型完全基于矩形区域，因此无法支持圆形、不规则形状以及透明通道遮罩等复杂效果。这意味着它更适合规则界面裁剪，而复杂视觉表现仍然需要依赖Stencil Mask完成。

从实际项目角度来看，RectMask2D与Mask并不是互相替代关系，而是两种针对不同需求的裁剪方案。RectMask2D强调性能与稳定性，Mask强调表达能力与像素级控制。合理选择二者的使用场景，是UGUI性能优化中的重要组成部分。

通过本章内容可以看出，UGUI中的裁剪系统并不仅仅是简单的“显示或隐藏”逻辑，而是深度参与了Canvas渲染流程、GPU状态管理以及Batch结构组织的底层机制。理解RectMask2D的工作方式，对于后续分析ScrollView、大规模UI列表以及Canvas性能优化具有重要意义。

