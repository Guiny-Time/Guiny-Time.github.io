---
tags: ['UI','UGUI','NGUI','FGUI','Unity']
categories: 引擎功能
title: UnityUI程序设计与UI性能优化（一）：基本概念
date: '2023-06-30'
updated: '2023-06-03T13:27:51.533Z'
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAa2bc4a06555293dfd21c7343d47b81cd.png
---

# 常见UI框架
**NGUI**、**UGUI**、**FGUI**，这些“GUI”都是什么？GUI 通常指 graphic user interface（图形用户接口），是指采用图形方式显示的计算机操作用户界面。在这篇文章中我将简单介绍一下这几个常见名词，而在之后的章节中，笔者将从 UGUI 源码剖析、UI 框架的程序设计以及 UGUI 性能优化三个方面详细展开。

## UGUI (Unity Graphic User Interface)
这是我们最常听说的 GUI，也是 Unity 内置的用户界面系统，从 Unity 4.6 版本开始推出，可以在Github上找到它的<a href="https://github.com/Unity-Technologies/uGUI">开源仓库</a>。UGUI 以 Canvas 为核心，提供了丰富的 UI 组件（比如我们常用的button、slicer、text等）和可视化编辑工具，操作简单直观。

- **优点**
  - Unity 原生的 GUI 系统，和 Unity 的其他模块（例如事件、渲染、动画）深度集成，并且由**官方**更新和维护。
  - 社区中存在大量的教程和文档。

- **缺点**
  - 由于 Canvas 的重绘机制（特别是复杂或动态的 UI 元素）可能导致性能瓶颈，这部分会在 UI 性能优化的章节详细展开。
  - 开发效率不如第三方工具。

虽然存在上述缺点，但 UGUI 仍然是大多数中小型 Unity 项目开发者的首选（或者UI系统很简单时），主要是因为方便而且有官方支持。

## NGUI (Next-Gen UI)
<img src="https://cgioo-1310123335.cos.ap-nanjing.myqcloud.com/forum/201805/08/152829fonnwbjjobgklhmk.jpg" alt="好贵的嘞" />

<a href="https://assetstore.unity.com/packages/tools/gui/ngui-next-gen-ui-2413?srsltid=AfmBOorqdbt7YL9hQ7KFk3llxbkMS_t3klqmKq-sY8iPN41StOG4MJWj">NGUI</a> 是 Unity 上最早的第三方 UI 框架之一。NGUI 诞生于 UGUI 之前，是早期 Unity UI 开发的主流解决方案，因为早期 Unity 并不存在 GUI，想配置 UI 必须通过脚本的方式实现，非常麻烦。NGUI 就是在这样的契机下诞生的插件，它比 UGUI 更早支持复杂 UI 功能。

渲染优化方面，NGUI 把 UIPanel 中的 Widget（UI元素的基类）按**深度**进行排序，然后将相同材质的元素（例如使用相同图集的图片或者文字）的 Mesh 合并。网格合并的优点是合并后这些元素就只产生一个 drawcall（类似于 Unity 的合批）。但这个合并过程需要计算所有 UI 元素坐标相对于 Panel 的变换，而且如果 UI 元素的行为改变（例如平移，缩放等）都会触发网格重新合并，这会带来一定的 CPU 消耗。开发者需要谨慎组织 UI 元素到各个 UIPanel 上，并且对深度需要细致安排，否则达不到减少 drawcall 效果的同时更可能带来比较大的 CPU 消耗。

- **优点**
  - 性能优越：相比 UGUI NGUI 的 UI 元素使用独立的渲染方式，整个 UI 只有一个 drawcall，性能更好。
  - 开发效率高：提供直观的 UI 层级管理和编辑器扩展。
  - 功能齐全：拥有许多开箱即用的功能（如拖拽、动态布局等）。

- **缺点**
  - 付费工具，而且不便宜。
  - 自 UGUI 推出后，使用 NGUI 的开发者逐渐减少，官方维护也不如过去频繁（最近一次版本更新在2018年）。
  - 学习成本稍高，部分机制与 Unity 自带系统不一致。

从 NGUI 的优点出发，它适合那些对性能要求极高的项目（因为只有一个 drawcall）。

## FGUI (FairyGUI)
<img src="https://www.fairygui.com/img/brand-logo-full.3e1cebcf.svg" width=300 />

FGUI 是一款**跨平台**的 UI 框架，它不仅支持 Unity，还支持诸如 Cocos Creater、Laya 等其他引擎。FGUI 提供了一个独立的 UI 编辑器（FairyGUI Editor），以数据驱动 UI 的方式，便于跨平台和资源管理。

FGUI 能在不改变最终显示效果的前提下，尽可能的把相同材质的物体调整到连续的 RenderingOrder 值上，以促使他们能够被Unity 的动态合批优化。有关于FGUI更详细的渲染优化原理，可以看检查<a href="https://www.fairygui.com/docs/unity/drawcall">官方文档</a>。

- **优点**
  - 轻量高效，内存占用低，渲染效率高，适合移动端游戏。
  - 多平台支持，FairyGUI Editor 可以快速创建复杂 UI，提升开发效率。
  - 功能丰富，内置的动画、动效支持强大，便于快速制作精美 UI。

- **缺点**
- 额外的学习成本。

FGUI 适合跨平台（尤其是 Web 和移动端游戏）或对 UI 动效需求高的项目，例如需要独立的 UI 工作流（减轻程序拼 UI 的工作负担）或更高效资源管理的团队。

# UGUI的基本架构
作为 Unity 的原生 UI 框架，UGUI 的应用场景要远多余另外两种框架，因此本系列将重点介绍 UGUI 相关的内容。相信大多数初学者对 UGUI 的认识仅仅停留在使用层面，而对其背后的源码实现却鲜有深入了解。本着知其然而知其所以然的想法，本系列将以 UGUI 源码为切入点，介绍 UI 管理系统的设计、UGUI 的渲染过程、事件响应以及优化建议。

从源码角度来看，UGUI 的架构主要分为以下几个部分：

1. Canvas 与 CanvasRenderer
2. Graphic 与其派生类
3. Event System 与 Input 模块
4. 布局系统（Layout System）
5. 性能优化机制（Batching 和 Canvas Update）

接下来，我们逐一剖析这些模块的源码及其功能。

## Canvas 与 CanvasRenderer
### Canvas
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAbbe86f8984285a80ce46ffb6f5e2bec8.png" />

**Canvas**（画布）是 UGUI 系统的根基，负责定义一个 UI 渲染上下文。每个 UI 场景至少包含一个 Canvas，它决定了整个 UI 渲染的基本环境。Unity 官方并没有公开 Canvas 的源码，但通过组件所暴露出来的接口，我们可以看出 Canvas 承担了以下职责：

- **渲染模式**：通过 `RenderMode` 属性决定 UI 的渲染方式（屏幕空间-覆盖、屏幕空间-相机、世界空间）。
  - **屏幕空间-覆盖**（Screen Space-Overlay）
    该模式下的画布会填满整个屏幕空间，并将画布下面的所有UI元素置于屏幕最上方，此模式下画布会直接缩放来适应屏幕，然后直接渲染而不参考场景或者相机(即使场景中根本没有相机也会渲染UI)，如果屏幕的大小或者分辨率发生改变，画布将自动改变大小来自适应。
  - **屏幕空间-相机**（Screen Space-Camera）
    与 Screen Space-Overlay 相类似，画布放在指定摄像机的特定位置处。
    UI 元素是由**摄像机**来渲染的，UI 外观与摄像机的设置有关。如果摄像机设置为了正交视图，那么 UI 元素将会展示为透视图，透视失真量是由摄像机的视野来控制。
    如果屏幕的大小、分辨率或者是摄像机视锥体发生改变，画布也会自适应。场景中比 UI 平面更靠近摄像机的所有 3D 对象都将在 UI 前面渲染，而平面后的对象将被遮挡，因此这种模式可以用来实现**在 UI 上显示 3D 模型**的需求。
  - **世界空间**（World Space）
    画布的行为与场景中其他对象一样，类似于一张面片(Plane)，并且可以在通过`RectTransorm` 手动设置画布大小和位置及其旋转角度(前面两种模式不可以)，其 UI 元素是根据其 3D 位置所在的场景中其他对象前面或者后面渲染。适应于要称为世界一部分的 UI。这种界面也称之为”叙事界面”。
- **事件捕获**：为 UI 元素提供事件响应的基础。
- **层级管理**：通过 `sortingOrder`、`orderInLayer` 等属性控制 Canvas 在渲染队列中的顺序。

### CanvasRenderer
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA5094140d4eb67dd7464ed2637a7f9fa8.png" />

**CanvasRenderer**（画布渲染器）则是 Canvas 的底层**渲染工具**，直接负责将 UI 数据提交给渲染管线。其主要功能包括：

- **设置渲染数据**：通过 `SetMaterial`、`SetMesh` 等方法设置渲染所需的数据。
- **批处理优化**：通过合并顶点和减少绘制调用，提升渲染性能。
- **裁剪**：通过 `Cull` 方法裁剪不可见部分。

## Graphic 与其派生类
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA88f81fea9616098204b853c3caa39e69.png" width=400 />

**Graphic**（图形）是所有 UI 元素的基类，包含了 UI 元素的通用功能。常见派生类包括：

- **Image**：用于显示**图片**。
- **Text**：用于显示**文本**。
- **RawImage**：支持**原始纹理**显示。

Graphic 的源码位于 UI 文件夹下，主要功能包括：

- **材质与颜色**：提供了 UGUI **默认材质**的实现，管理 UI 元素的**材质**和**颜色**。
- **绘制调用**：通过 `OnPopulateMesh` 填充顶点数据、`SetAllDirty`（脏方法）让画布变脏（`Rebuild` 发起画布重建）等操作影响 UI 元素的绘制。

## Event System 与 Input 模块
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA3e9618b8cc56798b018d22785952946a.png" width=400 alt="EventSystem默认挂载的组件参数" />

UGUI 的事件系统由 EventSystem、StandaloneInputModule（或 TouchInputModule）和Raycaster 三个部分组成。在我们在场景中创建第一个 Canvas 时，Unity 会自动帮我们创建一个名为 EventSystem 的对象，上面就包含了其中的两个部分。

- **EventSystem**：事件分发中心，主要功能是**管理当前选中的对象**以及**分发事件到正确的组件**。
- **StandaloneInputModule** 和 **TouchInputModule**：不同平台的输入处理模块。
- **Raycaster**：负责将输入位置映射到 UI 元素。

在介绍 UGUI Button 的点击事件时，我们会详细讲讲 UGUI 的事件系统是怎么一回事。

## 布局系统（Layout System）

布局系统负责自动调整 UI 元素的位置和大小，常见组件包括：

- **Layout Group**：
  - `HorizontalLayoutGroup` 和 `VerticalLayoutGroup`：把子元素水平或垂直排列，按顺序摆放，就像码放积木。组内部会遍历子对象，依次设置它们的起始位置。
  - `GridLayoutGroup`：子元素按网格排列，可以指定每行每列的数量。适合制作像关卡选择界面这种规则分布的 UI。
- **Content Size Fitter**：根据子元素的大小动态调整父元素的尺寸。比如一个滚动列表，里面的内容有多大，外框就多大。
- **Aspect Ratio Fitter**：根据宽高比调整尺寸。

布局组的工作原理很简单，简单来说分为两步：

1. **计算布局输入**：分析子元素的大小和间距，计算出需要的总尺寸。
2. **设置布局**：根据计算结果摆放每个子元素。

以 HorizontalLayoutGroup 为例：

```C#
public class HorizontalLayoutGroup : LayoutGroup {
    public override void CalculateLayoutInputHorizontal() {
        // 计算子元素的宽度
    }
    public override void SetLayoutHorizontal() {
        // 设置子元素的位置
    }
}
```

布局系统的本质是一个递归计算过程，从根节点开始逐层调整子元素的位置和大小。因此如果一个布局组件嵌套另一个布局组件，调整子元素时，可能会引发多次重计算，导致帧率下降。

## 性能优化机制
UGUI 的性能优化，主要集中在减少 Draw Call（绘制调用） 和 Canvas 更新开销。以下是两个核心优化点：

- **批处理**（Batching）
批处理就是尽量把相邻的 UI 元素**合成一批**来渲染，减少 GPU 的工作量。UGUI 可以自动合并相邻的 UI 元素以减少绘制调用，其核心规则包括：
  - **材质合并**：相同材质的元素会被合并到同一批次。在开发中应尽量减少材质的切换（会打断合批），例如采用**图集**。
  - **相邻元素**：UI 元素在渲染顺序上需要相邻，否则无法合并。
- **Canvas 的更新机制**
Canvas 是 UI 的渲染核心，但 Canvas 的更新可能会拖慢性能，在介绍 UGUI 渲染过程时我们会详细展开聊一聊更新机制具体是怎么一回事。Canvas 的更新分为两种：
  - **布局更新**：调整 UI 元素的位置和大小。
  - **渲染更新**：重新生成顶点数据并提交到 GPU。

# 参考资料
https://www.cg.com.tw/NGUI/
https://www.tasharen.com/?page_id=140
https://www.fairygui.com/docs/unity/drawcall
https://www.bilibili.com/opus/479970364929118811
https://juejin.cn/post/6990184801869234190