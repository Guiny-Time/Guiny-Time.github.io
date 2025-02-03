---
title: Hold-ON！复盘：物理材质
date: 2023-09-17 16:10:26
updated: 2023-09-17
tags: ['物理', '物理材质', '物理模拟','Unity','引擎','游戏开发']
categories: 引擎功能
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAe462e0c2d0477ea89497c1938dac9b73.png
---

# 什么是物理材质
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAa4c61ca03a695b11fd05c44a75f8ad24.png" alt="FEEL中的一个物理材质" />

![将该物理材质应用于地面的效果](pm.gif)

物理材质定义了**刚体表面的弹性和摩擦力**，包括以下几项：

- **Dynamic Friction**（动摩擦）
在移动时使用的摩擦力，动摩擦将尝试在与另一个对象接触时减慢对象的速度。`Dynamic Friction` 通常为 `0` 到 `1` 之间的值。
    - 值为 `0` 时表面就像冰一样滑
    - 值为 `1` 将使对象迅速静止（除非用很大的力或重力推动对象）。

- **Static Friction**（静摩擦）
当对象静止在表面上时使用的摩擦力，静摩擦力会阻止对象开始移动，如果向对象施加足够大的力，对象才会开始移动。`Static Friction` 通常为 `0` 到 `1` 之间的值。
    - 值为 `0` 时表面就像冰一样滑
    - 值为 `1` 时物体将很难移动。

- **Bounciness**（弹性）
表面的弹性如何？值为 0 将不会反弹。值为 1 将在反弹时不产生任何能量损失，预计会有一些近似值，但可能只会给模拟增加少量能量。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAba09b744bcfe0bb789265cf359009f92.png" alt="组合选项"/>

除此之外，还有一个 `Friction Combine` 和 `Bounce Combine`。这两个“组合”表示了**两个碰撞对象之间摩擦力/弹性的组合方式**，Unity 将根据所选择的模式对两个物体施加**相同**的弹性和摩擦力效果。以 `Friction Combine` 为例，上图中的四个组合选项的含义分别是：

- **Average**：对两个刚体的摩擦值求平均值。
- **Minimum**：使用两个值中的最小值。
- **Maximum**：使用两个值中的最大值。
- **Multiply**：两个刚体的摩擦值相乘。

对于弹性来说也适用。

{% note info %}
**如果两个对象的组合模式不同怎么办？**

在这种情况下，Unity 将使用优先级最高的组合模式，这四个模式的优先级是：`Average` < `Minimum` < `Multiply` < `Maximum`。
例如：如果一种材质设置了 `Average`，但另一种材质设置了 `Maximum`，那么要使用的组合函数是 `Maximum`，因为它具有更高的优先级。
{% endnote %}

{% note warning %}
**注意**

Nvidia PhysX 引擎使用的摩擦力模型针对模拟的性能和稳定性进行了调整，并不一定代表真实物理的高度近似值。具体而言，大于单个点的接触面（例如两个相互叠放在一起的盒子）将计算为具有两个接触点，其摩擦力将是现实世界物理学中的两倍。在这种情况下，可能希望将摩擦系数乘以 0.5 以获得更真实的结果。
同样的逻辑适用于弹性模型。由于各种模拟细节（如位置校正），Nvidia PhysX 无法保证完美的能量守恒效果。因此，比如说，如果受重力影响的对象的弹性值为 1，并与弹性为 1 的地面碰撞，则对象的弹跳高度应该会高于初始位置。
{% endnote %}

# 为项目设置默认物理材质
如果我们希望整个项目都采用某种物理材质的话，一个一个为刚体进行设置是非常麻烦的。为此，我们可以在 **Edit > Project Settings > Physics** 里找到 Default Material（默认材质）属性，为其选择默认物理材质。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAee930dc7d1511f1a31441ebc794b6e27.png" alt="设置默认物理材质" />

# 参考资料
https://docs.unity3d.com/cn/2019.4/Manual/class-PhysicMaterial.html