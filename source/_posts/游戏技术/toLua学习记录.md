---
title: toLua学习记录
date: 2025-03-04 21:34:24
tags: ['Lua','toLua','编程', '热更新', 'demo', 'Unity']
categories: 引擎功能
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAf4da76dd755ee1073f55b9f2d33af128.png
---

{% note danger %}
**注意：本文尚未施工完成。**
{% endnote %}

最近了解到很多项目组使用的技术栈是 Unity + ToLua（而非之前所了解的 xLua），因此想研究学习一下有关于 ToLua 框架的知识。类似于 xLua，我想从“C# 调用 Lua”、“Lua 调用 C#”两方面展开学习，并探讨一下 ToLua 跨语言调用背后的原理。

# 什么是ToLua
<a href="https://github.com/topameng/tolua">ToLua</a> 是 Unity **静态绑定** Lua 的一个解决方案，它通过在 C# 中集成 Lua 虚拟环境，可以自动生成用于在 Lua 中访问 Unity 的绑定（bind）代码，并把 C# 中的**常量**、**变量**、**函数**、**属性**、**类**以及**枚举**暴露给 Lua 使用。ToLua 通过 C/C++ 层优化减少了调用开销，运行效率比使用反射的 <a href="https://antunity.github.io/uLua-docs/">uLua</a>、xLua 更高（尤其是在安卓平台上）。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAf42bc00126ca625d06430f8a91bdcd01.png" alt="uLua" width=200 />



# 在Unity中使用
## C#调用Lua
- toLua解析器
- toLua解析器不同文件加载方式
- toLua解析器管理器
- 全局变量获取
- 全局函数获取
- 访问Lua中table表现List和Dictionary
- 访问Lua中table
- 使用toLua提供的协程

## Lua调用C#
- 类
- 枚举
- 数组、List和Dictionary
- 函数（拓展方法）
- 函数（ref和out）
- 函数（重载）
- 委托和事件
- 协程