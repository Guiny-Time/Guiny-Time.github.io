---
title: Unity动画系统：性能优化
date: 2022-01-14 11:09:55
tags: ['动画','Unity','动画系统','性能优化','优化','经验分享','引擎功能']
categories: 引擎功能
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAea75700ebbbfbd809b975ead11e1517d.png
---

通过之前两篇文章的介绍，相信大家对 Unity 的动画系统已经有了更深刻的了解了吧！Unity 的动画系统功能十分强大，但在实际项目中随着动画数量的增加，我们有可能会遇到性能瓶颈，导致游戏帧率直线下降，因此**性能优化**在动画系统中同样举足轻重。如何让动画系统更高效地运作呢？今天我们就来聊聊 Unity 动画性能优化的一些技巧吧！

# Animation
## 简单动画
播放没有混合的**单个简单动画**(也就是状态机里头只有一个动画片段)会使 Unity 的速度比旧版动画系统更慢。如果动画较为简单，建议直接用 Legacy 下的 **Animation** 播放或尝试改用动画区间库例如 **Dotween** 来实现。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAf01a45ab816f8bbebace26fa9d28edc3.png" width=400 />
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA21ccfeb4d5bedf512b6ba88a754b000f.png" width=350 />

{% note info %}
**Animation 与 Animator 播放动画**

当动画在检查器中必须标记为“Legacy”时我们就可以通过 `Animation.Play` 来播放动画了。Animation 与 Animator 播放动画的效率取决于动画的**曲线条数**，当曲线条数小于某个临界点时 Animation 更快，当动画曲线条数大于这个临界点时 Animator 更快。一次你对于简单动画来说可以采用 Animation 来播放。
当 **CPU 核数较少**时，Animation 播放动画也有优势，反过来 Animator 表现会更好。
{% endnote %}

使用 Animation 时需要注意应始终通过将 **Culling Mode** 设置为 `Based on Renderers` 来优化动画。

## 动画片段
针对动画的**缩放**（Scale）曲线的开销比位移、旋转曲线的开销更大。为了改善性能，请避免使用缩放动画。

上述规则不适用于动画中的**常量曲线**（动画片段长度值相同的曲线）。常量曲线经过优化，成本低于普通的曲线。与默认场景值相同的常量曲线不会每帧写入场景。

## 动画事件
动画事件（Animation Events）虽然强大，但会对性能产生额外开销，可以通过 **Animators.FireAnimationEventsAndBehaviours** 查看动画事件的具体耗时。在使用动画事件时我们需要注意应当仅在**必要时**使用动画事件，并尽量减少复杂逻辑。

为什么会有这样的限制？因为动画事件会导致 CPU **中断**，并触发相应的脚本回调。如果动画事件过于频繁或逻辑复杂，可能会导致帧率下降。在开发过程中，我们应当避免**高频动画事件**的使用，特别是每帧调用动画事件。

# Animator
## 减少数量和实例化次数
动画系统最常见的性能杀手就是**大量**活跃的 Animator 组件。每个活跃的 Animator 组件都会消耗 CPU 资源来计算动画状态机的切换、动画权重混合等， 当场景中同时活跃的Animator 超过 `30` 个时，性能就可能开始吃紧，导致游戏帧率下降。

{% note info %}
未设置控制器的 Animator 不会花时间执行处理，这也是一个可以用来优化的方向。
{% endnote %}

在Active或实例化一个带有 Animator 组件的游戏对象时会触发 **Animator.Initialize** 函数，这个函数的耗时比较高，因此在战斗中不建议过于频繁地对含有 Animator 的游戏对象进行 Deactive / Active 的操作。

对于频繁实例化的角色，可尝试通过对象池的方式进行处理，在需要隐藏角色时，不直接 Deactive 游戏对象，而是**禁用 Animator 组件**，并把游戏对象**移到屏幕外**，从而降低 Animator.Initialize 的调用频率。

## 查询策略
在脚本中查询 Animator 的动画**片段**或动画**参数**时，使用**哈希**而不是字符串来查询 Animator，这样的速度会更快。动画机状态中的各个动画片段 / 动画参数具有**唯一性**，通过哈希查找的速度是O（1）；而使用字符串查找时，Unity 需要逐字符比较整个字符串才能找到匹配项。字符串具有**不可变性**，如果字符串较长或查询频率较高，这种逐字符比较会产生较大的性能开销（可能导致 **GC**）。

## 优化游戏对象
“**优化游戏对象**”即 **Optimize Game Objects** 选项，启用该选项后，Animator 会移除骨骼和动画目标上的所有 Transform 对象在运行时的直接访问，从而减少运行时的计算和内存开销。当我们给人形模型指定 Avatar 肌肉系统之后，就可以勾选该选项删除模型的骨骼，起到优化作用。

## 关闭RootMotion
Root Motion（动画驱动位移）功能会将动画中的位移应用到角色对象，但实际上这里存在一定的性能开销。滥用 RootMotion 是错误的，我们可以将一些没什么影响的信息**烘培**到动画中，或者只在必要时启用Root Motion。

如果游戏角色的一些移动逻辑已经在脚本中实现（比如跑酷类游戏，滑步问题没那么重要），那么 Root Motion 反而会显得多余，同时会增加 CPU 计算负担。可以直接在 Animator 组件中取消勾选“Apply Root Motion”，关闭 RootMotion 功能。

另外，在**通用类型**的动画中，如果动画没有使用根运动，请确保未指定根骨骼。

## 优化层级
大多数时间，Unity 都在估算动画，并将动画层和动画状态机的开销保持在最低水平。向 Animator 添加一个层的成本取决于层播放的**动画和混合树**。当一个层暂时没有用的时候，把层的**权重**设置为 `0` 可以让 Unity 跳过对该层的更新。

## 剔除模式
Animator 可能会更新一些在屏幕外的对象，导致性能浪费。我们应该始终通过将 Animator 的 **Culling Mode** 设置为 `Cull Update Transforms` 或 `Cull Completely` 来优化动画。`Cull Update Transforms` 适用于动画会产生位移的 Animator，`Cull Completely` 适用于动画不会产生位移的 Animator。

在之前的文章中我们介绍过 Culling Mode有以下几种:

- **Always Animate**：无论对象是否在视野内，都会更新动画。
- **Cull Update Transforms**：对象离开视野后，暂停动画更新，但仍会更新 Transform 位置。
- **Cull Completely**：对象离开视野后，暂停所有动画计算。

## 减少动画剪辑
Animator Controller 的状态机 Graph 中的所有动画状态的动画剪辑都会载入到**内存**中，因此当一个动画状态机中有海量状态节点时，内存开销就会比较大。

# 参考资料
https://docs.unity3d.com/cn/2021.3/Manual/MecanimPeformanceandOptimization.html
https://zhuanlan.zhihu.com/p/382656748
https://www.cnblogs.com/zerobeyond/p/17729863.html