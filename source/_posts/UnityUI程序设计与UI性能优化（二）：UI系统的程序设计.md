---
tags: ['UI','系统','框架','设计','Unity']
categories: 探索发现
title: UnityUI程序设计与UI性能优化（二）：UI系统的程序设计
date: '2023-07-03T13:51:22.046Z'
updated: '2023-07-04T05:52:10.543Z'
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA63dffaa3b1762d6e793af5c10eb3503e.png
---

聊完了什么是 UGUI，现在可以看看作为开发者的我们应该如何设计 UI 系统的整体框架了。我在大二的时候曾经基于**栈**写过一个 <a href="https://github.com/Honghonglin/GameUsualFunction/pull/1">UIManager</a> ，原理很简单：一层一层打开的 UI 面板天然符合栈的数据结构思想，所以通过一个栈来维护打开的面板就可以了。这个模块由于和 hhl 学长的 UI 模块重复了所以没有合并进仓库，当然我的想法也并没有考虑到太多商业化场景中的复杂情况，因此在这里就不详细展开让大家见笑了。

本篇文章将聊聊 hhl 学长的 UI 模块和更商业化项目中的 UI 模块的设计思路。

# ProjectBase-UI分析
学长的代码在 <a href="https://github.com/Honghonglin/BaseProject/tree/main/Assets/Scripts/ProjectBase/UI">github</a> 仓库里，我对其中的UI部分拆开做了一张思维导图：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAd19ef10810f30e729f3dfa8870670dc6.png" />

整个 UI 管理器是一个单例的管理器，可以通过 UIManager 开启或关闭面板，也可以通过 UIManager 查找指定的开启的面板并调用其中 BasePanel 的方法。单例很好用，但它也存在增加耦合的风险，具体可以看我之前写的<a href="https://cattyhouse-guiny.xyz/2022/01/02/chapter1-%E5%8D%95%E4%BE%8B%E6%A8%A1%E5%BC%8F/">关于单例模式的文章</a>。

这个模块用起来很方便，但问题也很明显，比如说 UIManager 中隐藏面板的地方：
```C#
public void HidePanel(string panelName) {
    if(panelDic.ContainsKey(panelName)) {
        panelDic[panelName].HideMe();
        GameObject.Destroy(panelDic[panelName].gameObject);
        panelDic.Remove(panelName);
    }
}
```
HideMe()由继承了 BasePanel 的 XXPanel 类具体实现，一般用于关闭面板时的数据复位等回调操作。在这里代码直接将面板对象标记为销毁（Destory）并从字典中移除，实际上造成了潜在的 GC 开销，可能在未来导致大量内存碎片产生（由于 Unity 的 GC 特性，GC 只会标记清除而不会压缩空间）。

代码中实际上已经有了不同层级的 UI 根对象，实际上可以通过**对象池**的方式管理面板。

在 Panel 代码中实现 OnClick 和 OnValueChanged 略有些不必要，既然已经为不同面板实现了对应的 XXPanel 脚本，可以直接在脚本中实现相关的功能函数并在 Unity 的 Inspector 中添加绑定。一些开发者忌讳直接在检查器中绑定方法的行为，那么通过哈希表记录按钮名称与其对应的事件回调再动态调用也是没问题的。

# 复杂 UI 模块构思

<img src="https://assetstorev1-prd-cdn.unity3d.com/key-image/4514fb70-4b0b-4493-b7c7-dd5247d5b5be.jpg" alt="手机游戏的UI界面" width=400/>

在实际项目中，我们可能会碰到非常复杂的 UI 嵌套（比如一些微信小游戏或手机游戏），甚至可能有的游戏是完全基于 UI 的。如果我们使用前文中的 UI 模块，就很容易因为频繁创建/销毁面板造成 GC，使游戏变得卡顿。

我们依然可以考虑建立一个全局的UIManager负责管理所有面板，但UIManager需要避免产生前文提到的问题。

## 面板对象池与永久缓存
我们可以考虑为面板设置一个**缓存对象池** CachePanels，有了对象池就可以优化内存使用和性能、避免直接频繁地创建和销毁。对于手机游戏或者微信小游戏来说，缓存池太大会导致内存占用，所以我们可以设置一个对象池大小上限（从经验角度出发可以是3），当对象池满时，移除**最早**加入对象池的面板（因此可以考虑用**队列**的数据结构）。

有一些面板可能**经常被打开**检视（比如个人信息、背包），如果只是把它们简单的加入 CachePanels 可能对性能优化没多大帮助（因为可能需要不停挤出和载入对象池）。对于这些常用 UI  面板，可以额外增设一个 ForeverCachePanels 列表永久缓存它们。

> 这里说“永久”有些绝对，我们仍然可以通过代码设置将某个属性将“永久缓存面板”从 ForeverCachePanels 中移除

## 避免频繁操作
<img src="https://www.lifewire.com/thmb/sFSPR9f4HURmYarl8AKsNbRWVvc=/540x405/filters:no_upscale():max_bytes(150000):strip_icc()/GettyImages-2178782978-f4ed9e8f47b64c7b9ed4b7d0db58a292.jpg" width=300>

有的时候由于卡顿或者某些无法避免的神秘原因，用户点击一次开启面板按钮后游戏并没有反应，此时一些心急的玩家可能就会疯狂点击按钮试图打开某个面板。如果按照学长的 UI 模块，就可能因为在按钮点击回调中不停关闭自身而导致空指针异常。

为了避免暴躁用户高频点击导致出错，我们可以考虑维护一个等待开启列表 WaitOpenList 和开启列表 OpeningPanels。

- 等待开启列表：当面板被请求打开，且不在开启列表/缓存池时，存入等待开启列表中
- 开启列表：所有被打开的面板都在这个列表中

也就是说当我们打开一个面板时，
1. 先判断该面板**是否存在**（避免不小心打开非法面板导致空指针异常）
2. 判断该面板是否存在于**等待开启列表/开启列表/缓存池中**，如果在后两个那就直接打开
3. 如果都不存在，则将面板加入等待开启列表
4. 在下一帧对所有存在于等待开启列表中的面板执行**打开**操作，加入开启面板中

关闭面板就比较简单粗暴了，如果面板是打开或等待打开状态那就将其移除出当前结构并加入 Cache 中。

## 对所有面板的管理
学长的模块中只有对于指定面板的 Show 与 Hide 操作。实际上，在过场景或者某些特定条件下我们可能需要：

- 关闭当前所有打开的面板
- 关闭一系列不需要的面板
- 释放所有缓存

有了前面提到的几种数据结构，对于实现这三个功能就不是什么大问题了，可能需要用一些属性标注面板类型。

## 生命周期
在生命周期中，UIManager做的事情：

1. **Start**
类似学长的代码，我们仍需对UI进行分层，这样可以方便不同类型的UI的管理。
初始化前文提到的数据结构。

2. **Update**
遍历开启列表 OpeningPanels，Update 里面的面板

3. **LateUpdate**
当缓存池满时，释放超过上限的面板
LateUpdate 开启面板中的 Panel

