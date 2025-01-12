---
title: Hold-ON！复盘：物理关节
date: 2023-09-03 23:50:12
updated: 2023-09-05
tags: ['物理', '物理模拟','Unity','引擎','游戏开发']
categories: 引擎功能
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAccd21636323a23cada9dee7080c2e223.png
---

我们经常在游戏中看到下面的 gif 里所展示的“**可交互场景对象**”。这样的对象的存在增添了场景的可交互性，让场景不那么死板，显得更有生气和灵性。除了下图展示的游戏 **HUE** 之外，**GRIS**、**空洞骑士**等游戏也大量使用了可交换场景对象。

![HUE中的可交互场景对象-藤曼](joint.gif)

这种可交互的场景对象是怎么实现的呢？这就要引出文章的主题：Joints（关节）了。

# 什么是关节
Joints（关节）是一种用于**将两个刚体连接在一起**的物理组件，通常用于模拟现实世界中的连接、约束或机械行为。例如，你可以用关节连接一个物体到另一个物体、固定到某个位置，或者模拟摆锤、弹簧等效果。

## 固定关节/Fixed
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAe12a3c04a77fec083144ecea735dc9d3.png" alt="固定关节组件面板" width=350 />

**固定关节**（Fixed Joint）将两个对象**固定**在相对位置，适合用来模拟焊接、粘合等固定效果。例如将一个箱子固定在某个位置、或者连接两个不会相对移动的物体（例如把粘土炸弹固定到某个移动对象上）。

可使用固定关节面板上的 `Break Force` 和 `Break Torque` 属性来设置关节强度的限制。如果这些值小于无穷大，并对该对象施加大于这些限制的力/扭矩，则其固定关节将**被破坏**并将摆脱其约束的束缚。

<img src="fj3.gif" alt="固定关节断裂" width=400 />

另外，固定关节的刚体类型也会影响到展示的效果，例如：

<img src="fj.gif" alt="当Connected Body为运动学刚体时" width=400 />
<img src="fj2.gif" alt="二者均为物理学刚体时" width=400 />

## 弹簧关节/Spring
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAc0e63b6157e1bc47cc512ada46c38d33.png" alt="弹簧关节组件面板" width=350 />

**弹簧关节** (Spring Joint) 将两个刚体连接在一起，但允许两者之间的**距离改变**，就好像它们通过弹簧连接一样。`Anchor` 表示了弹簧连接点的位置，`Connected Anchor` 则是自动计算出的自身的连接锚点位置（也可以取消勾选自动计算）。锚点在检查器中以一个棕色小方块的形式展现：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA46ee291864a9aff8e9fcdf25398d729e.png" alt="位于上方的棕色锚点" />

弹簧就像一块弹性物，试图**将两个锚点一起拉到完全相同的位置**。拉力的强度与两个点之间的当前距离成比例，其中**每单位距离的力**由 `Spring` 属性设定。为了防止弹簧无休止振荡，可以设置 `Damper`（**阻尼**）值，从而根据与两个对象之间的相对速度按比例减小弹簧力。值越高，振荡消失的速度越快。

<img src="sj.gif" alt="Spring与Damper对弹簧关节的影响" width=500 />

`Min Distance` 和 `Max Distance` 值用于设置弹簧不施加任何力的距离范围。可以使用该距离范围允许对象进行少量的独立移动，但当对象之间的距离太大时将它们拉到一起。

## 铰链关节/Hinge
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAc35a496a012fa270843d8fa3c6668de7.png" alt="铰链关节组件面板" width=350 />

**铰链关节** (Hinge Joint) 将两个刚体组合在一起，对刚体进行约束，让它们就像通过**铰链**连接一样移动。铰链关节非常适合用于门，但也可用于模拟链条、钟摆等对象。我们在文章最开始提到的藤曼效果就是通过铰链实现的。`Anchor` 表示了铰链连接处的位置，`Axis` 则是铰链的轴，`Connected Anchor` 同样自动计算出的自身的连接锚点位置（也可以取消勾选自动计算）。铰链的锚点在检查器中以一个棕色小横杆的形式展现：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA1572e9cb253ae8197498a5455ec129ff.png" alt="位于上方的棕色横杆锚点" />

铰链支持弹簧效果，也可以通过 `Motor`（发动机）驱使着一直运动，不过 `Spring` 和 `Motor` 是互斥的，同时使用这两者会导致不可预测的结果。

对于 `Motor`，我们可以为其设置目标速度 `Target Velocity` 和力的大小 `Force`：

<img src="hj.gif" alt="Target Velocity与Force对铰链关节的影响" width=500 />

铰链还可以通过 `Use Limits` 来限制铰链的旋转角度在 `Min` 到 `Max` 值范围内。`Bounciness` 属性当对象达到了最小或最大停止限制时对象的**反弹力**大小。

<img src="hj2.gif" alt="Limits对铰链关节的影响" width=500 />

那如何通过铰链关节做出 HUE 中的藤曼效果呢？答案很简单，我们可以将多个铰链关节串在一起形成一条**链条**，例如下方所做的简单尝试：

<img src="hj3.gif" alt="铰链链条" width=400 />

## 角色关节/Character
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA9ad0a05f6ae2f8455d635fc925500c13.png" alt="角色关节组件面板" width=350 />

角色关节 (Character Joint) 主要用于**布娃娃**效果。此类关节是延长的**球窝关节**，可以用于模拟人体的**骨骼**连接（如肩膀或膝盖）。我们可以限制关节的旋转角度并添加弹性，让其更接近真实的效果。

<img src="cj.gif" alt="添加了角色关节的人物模型" width=400 />

一些很“物理”的游戏，比如人类一败涂地和猛兽派对的“摇摇晃晃”的行走效果，或许都在其中使用到了类似角色关节的设置。

## 可配置关节/Configurable
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA89aed3a23b0f73296424347ddc6bacb2.png" alt="可配置关节组件面板，一张图都放不下" width=350 />

可配置关节是所有关节中最复杂的一个。显然，从它的名字可以看出，可配置关节可以模拟出任意一种关节的效果，我们还可以通过配置可配置关节将关节修改为高度专业化的关节。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA17ddd20c9484bd66655893eb0ee1545e.png" width=300 />

`Anchor` 和 `Axis` 类似前面出现的设置，`XYZ Motion` / `Angular XYZ` 则代表在XYZ轴上的运动与旋转模式，可以改成 ：

- **Limit**：约束，允许在预定义的限制范围内自由移动。
- **Lock**：锁定，不允许任何运动。
- **Free**：自由，允许任意移动。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAe4c0a689e1fe3c73b243b68963ac07a8.png" width=300 />

这里有一系列 `Limit`，`Linear Limit` 可以**限制平移运动**，该属性定义了关节可从其原点移动的最大距离（分别沿每个轴进行测量）。`Angular Limit` 可以**限制旋转**，与线性限制不同的是此属性可以为每个轴指定不同的限制值。

通过观察面板可以发现 Limit 分为 **Spring** 和 **Bounciness** 两类。默认情况下，关节在达到约束时会直接停止移动。但这样的非弹性碰撞在现实世界中是罕见的，因此向受约束的关节添加一些弹跳感会很有用。**Spring** 限制定义了**拉回的弹簧力**和**阻尼**，而 **Bounciness** 限制定义了到达限制后的**反作用力**和**接触距离**。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAb865746dcb7af6c87d7276cb310367f6.png" width=300 />

目标位置、目标速度、目标旋转角度、目标扭矩速度和之前关节中出现的目标 xx（Target xx）类似，都是游戏对象希望到达的目标。面板中的 `Drive` 则可以给对象施加对应轴上的弹力（和阻尼设定），这样不管怎么拉扯对象，对象都会依照 `Drive` 设置的弹力和阻尼参数回到目标位置。

`Slerp Drive` 指定了驱动扭矩如何使关节围绕所有局部 x 轴旋转，仅当 `Rotation Drive Mode` 属性设置为 **Slerp** 时才可使用此属性。Slerp Drive
不同于其他驱动模式在不同的轴上施加力，它使用**四元数的球面插值**或 “Slerp” 功能来重新定向关节。Slerp 过程不会隔离单个轴，而是寻找将对象从当前方向带到目标的最小总旋转，并根据需要将该旋转应用于所有轴。Slerp Drive 和 X + Y/Z 轴 Drive 不能共存。

{% note orange 'fas fa-lightbulb' %}
**用可配置关节制作一个效果吧**

新建两个 Cube，为其增加可配置关节组件。将两个 Cube 摆放成上下位置关系，为上面的 Cube 可配置关节的 `Connected Body` 设置成下面的 Cube。
设置上方 Cube 的各项 `Driver` 的弹力为 20，运行游戏：

<img src="cj2.gif" width=200 />
{% endnote %}

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA1113e4eaf24be0b9e9d0ac37cdbec20f.png" />

`Projection` 定义了当关节意外地超过自身的约束（由于物理引擎无法协调模拟中当前的作用力组合）时如何快速恢复约束。选项为 None 和 Position and Rotation。

- `Projection Distance`：关节超过约束的距离，必须超过此距离才能让物理引擎尝试将关节拉回可接受位置。
- `Projection Angle`：关节超过约束的旋转角度，必须超过此角度才能让物理引擎尝试将关节拉回可接受位置。

# 参考资料
https://docs.unity3d.com/cn/2022.3/Manual
https://docs.unity3d.com/cn/2022.3/Manual/wizard-RagdollWizard.html
https://www.youtube.com/watch?v=MElbAwhMvTc
https://www.youtube.com/watch?v=iGlD3f-5JpA
