---
tags: ['UI','UGUI','NGUI','FGUI','Unity']
categories: 探索发现
title: UnityUI程序设计与UI性能优化（一）：基本概念
date: '2023-06-30'
updated: '2023-06-03T13:27:51.533Z'
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAa2bc4a06555293dfd21c7343d47b81cd.png
---

**NGUI**、**UGUI**、**FGUI**，这些“GUI”都是什么？GUI 通常指 graphic user interface（图形用户接口），是指采用图形方式显示的计算机操作用户界面。在这篇文章中我将简单介绍一下这几个常见名词，而在之后的章节中，笔者将从 UGUI 源码剖析、UI 框架的程序设计以及 UGUI 性能优化三个方面详细展开。

# UGUI (Unity Graphic User Interface)
这是我们最常听说的 GUI，也是 Unity 内置的用户界面系统，从 Unity 4.6 版本开始推出，可以在Github上找到它的<a href="https://github.com/Unity-Technologies/uGUI">开源仓库</a>。UGUI 以 Canvas 为核心，提供了丰富的 UI 组件（比如我们常用的button、slicer、text等）和可视化编辑工具，操作简单直观。

- **优点**
  - Unity 原生的 GUI 系统，和 Unity 的其他模块（例如事件、渲染、动画）深度集成，并且由**官方**更新和维护。
  - 社区中存在大量的教程和文档。

- **缺点**
  - 由于 Canvas 的重绘机制（特别是复杂或动态的 UI 元素）可能导致性能瓶颈，这部分会在 UI 性能优化的章节详细展开。
  - 开发效率不如第三方工具。

虽然存在上述缺点，但 UGUI 仍然是大多数中小型 Unity 项目开发者的首选（或者UI系统很简单时），主要是因为方便而且有官方支持。

# NGUI (Next-Gen UI)
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

# FGUI (FairyGUI)
<img src="https://www.fairygui.com/img/brand-logo-full.3e1cebcf.svg" />

FGUI 是一款**跨平台**的 UI 框架，它不仅支持 Unity，还支持诸如 Cocos Creater、Laya 等其他引擎。FGUI 提供了一个独立的 UI 编辑器（FairyGUI Editor），以数据驱动 UI 的方式，便于跨平台和资源管理。

FGUI 能在不改变最终显示效果的前提下，尽可能的把相同材质的物体调整到连续的 RenderingOrder 值上，以促使他们能够被Unity 的动态合批优化。有关于FGUI更详细的渲染优化原理，可以看检查<a href="https://www.fairygui.com/docs/unity/drawcall">官方文档</a>。

- **优点**
  - 轻量高效，内存占用低，渲染效率高，适合移动端游戏。
  - 多平台支持，FairyGUI Editor 可以快速创建复杂 UI，提升开发效率。
  - 功能丰富，内置的动画、动效支持强大，便于快速制作精美 UI。

- **缺点**
- 额外的学习成本。

FGUI 适合跨平台（尤其是 Web 和移动端游戏）或对 UI 动效需求高的项目，例如需要独立的 UI 工作流（减轻程序拼 UI 的工作负担）或更高效资源管理的团队。

# 参考资料
https://www.cg.com.tw/NGUI/
https://www.tasharen.com/?page_id=140
https://www.fairygui.com/docs/unity/drawcall
https://www.bilibili.com/opus/479970364929118811
