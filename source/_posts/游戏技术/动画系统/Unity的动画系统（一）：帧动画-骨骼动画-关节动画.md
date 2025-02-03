---
title: Unity动画系统：动画系统基础
date: 2021-11-20
tags: ['动画','Unity','动画系统','教程','帧动画','骨骼动画','状态机','动画事件','经验分享','引擎功能']
categories: 引擎功能
cover: https://docs.unity3d.com/cn/2023.2/uploads/Main/AnimationIntroPic.jpg
---

{% flink %}
- class_name: Hi~ o(*￣▽￣*)ブ
  class_desc: 哈喽大家好，我是时光！经过将近一年的 Unity 学习，我已经对 Unity 的各个系统有了不少认知啦~欢迎大家看看我之前发布的一些文章哦：
  link_list:
    - name: 合成大西瓜
      link: https://cattyhouse-guiny.xyz/2021/01/30/%E7%94%A8Unity%E7%AE%80%E5%8D%95%E5%AE%9E%E7%8E%B0%E5%90%88%E6%88%90%E5%A4%A7%E8%A5%BF%E7%93%9C/
      avatar: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA4ba3c038673dd0ed730ddf48bec5eb09.png
      descr: Unity2D制作的合成大西瓜小游戏
    - name: 明日方舟
      link: https://cattyhouse-guiny.xyz/2021/04/03/%E7%94%A8Unity%E7%AE%80%E5%8D%95%E5%AE%9E%E7%8E%B0%E6%98%8E%E6%97%A5%E6%96%B9%E8%88%9F/
      avatar: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAc6d357b42abe814f212b5d5291989b44.png
      descr: Unity实现的明日方舟战斗系统
{% endflink %}

我打算开一个系列介绍一下 Unity 动画系统的基础知识，也算是对我这段时间使用 Unity 动画系统的一个总结，欢迎大家评论转发φ(゜▽゜*)♪！

本文将从 Unity 动画系统的基本概念出发，介绍动画机的工作原理，并为动画添加动画事件。话不多说，那我们就正式开始吧！

# 什么是Unity动画系统？
Unity 的动画系统是引擎内置的一个强大的系统，可以帮助用户创建、导入和管理动画资产，为 Unity 的所有元素（包括对象、角色和属性）提供简单的动画工作流程和动画设置。

动画系统中有三个重要的概念：**Animator**（动画机）、**Animation Clip**（动画剪辑）和 **Animation Controller**（动画控制器），这三者缺一不可。

![三者之间的关系](https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA5b3909275ddc04172a0c7206773f0b85.png)

## Animation Clip
**动画剪辑**（Animation Clip）是 Unity 动画系统的核心元素之一，它包含某些对象应如何随时间改变其**位置、旋转或其他属性（包括自定义脚本中的属性）** 的相关信息。每个剪辑可视为单个线性录制。除了通过编辑器中的 **Animation** 窗口创建和制作动画之外，Unity 还支持从**外部源**导入动画。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA80a67fc9cd8c12368ad44f65ffd31208.png" alt="点击Create就可以为选中的对象创建动画" />

{% note orange 'fas fa-lightbulb' %}
**为什么要从外部导入动画？**

在实际的游戏开发过程中，动画资产往往由美术人员制作。对于 2D 美术来说，**Spine** 或**龙骨**的界面更友好，制作出的动画效果也更好（而在 Unity 中想做出这样的效果可能就要使用插件来完成了），导出之后再导入 Unity 中。这样也可以减少直接在 Unity 中制作动画资产带来的潜在冲突问题。
对 3D 美术动画资产来说，往往在建模软件（例如 **Maya**、**ZBrush** 等）中绑定骨骼并制作动画，导出 `.fbx` 模型时 Unity 将从中提取出动画，无需在引擎中创建。
还有一些动画是通过专业的**动捕**软件制作的，这些动画也需要从外部导入 Unity。
{% endnote %}

一些动画可能是特定模型所特有的，不能在其他模型上复用（例如，某些模型具有独特的骨骼，因此有自己的一组动画），但多数时候美术人员会对资源进行划分（例如按照体型将人物骨骼划分成成男、成女、少男、少女、儿童五套），并为每种类型的资源制作动画库，这样动画剪辑就是**可以复用**的。

## Animation Controller
Animator Controller 的本质是一个**状态机**，负责决定**当前应该播放哪个动画剪辑**以及动画应该何时改变或如何**混合**（这会在后续的文章中提到）。当我们为某一游戏对象创建动画剪辑时，Unity 会**自动**为其创建一个 Animation Controller，Animation Controller 中会包含这一动画剪辑并将其设置为默认播放的动画。

我们也可以手动创建一个 Animation Controller，其中只包含 `Any State`、`Entry` 和 `Exit` 三个状态。我们可以将制作好的动画剪辑拖进窗口内，然后为不同的动画剪辑设置过渡关系（例如下图）。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAbfb0756e567314cd011c6e596ad7fa1a.png" alt="双击一个AnimationController可以看到的界面" />

{% note danger %}
一个复杂的 Animator Controller 可包含用于主角所有动作的**几十段**人形动画。
这毫不夸张，因此也需要做一些额外的措施来管理海量的动画，例如**分层动画**。
{% endnote %}

双击 Animation Controller 会进入 **Animator 窗口**，它也可以通过选项卡的 `Windows/Animation/Animator` 打开：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA4677eb5260efff9a54d3c6b84ca69f31.png" />

通过 Animator，我们可以为动画机设置**分层**（Layer）和**动画参数**（Parameters）：

- **分层**：Animator的分层功能允许你同时播放多个动画，比如角色的下半身在跑步，上半身在挥手（这两个动画位于不同的层）。
- **动画参数**：参数是动画逻辑的核心，可以通过设置 `Bool`、`Float`、`Int` 等类型的参数来动态切换动画，比如通过一个 `Bool` 类型的参数 `isJumping` 来控制玩家跳跃。

我们会在下一篇文章中介绍这两个概念的~

## Animator
![Animator](https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA9078e15de979dadee44b6aec03347cec.png)

这里的 Animator 和上一小节中所提到的“Animator 窗口”并不一样，是挂载在要播放动画的游戏对象身上的**组件**。开发者可以为该组件挂载对应的 Animator Controller，然后就可以播放动画了。除了动画控制器之外，Animator 组件还有以下几个参数：

1. **Avatar**
指的是“人形动画”，Unity 有专门为人形动画制作一套配置，开发者可以相对轻松地将同一组动画应用于各种角色模型。关于这个概念可以参考在 **Mixamo** 上为不同的人形模型绑定 Avatar，然后使用动画的过程，它们本质上差不多。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA35289757cba5dfcc761fa442d1b1f911.png" alt="mixamo logo" width=300 />

2. **Root Motion**
绑定该组件的游戏对象的位置是否可以由动画进行改变。

3. **Update Mode**
表示动画的更新模式，有三种选项：
    - **Normal**：跟随游戏的 `Update` 循环
    - **Animate Physics**：跟随游戏的 `Fixed Update` 循环，以固定时间步长计算，适合物理模拟
    - **Unscaled Time**：也属于 `Update`，但不受 **TimeScale** 影响，在 TimeScale = 0 的情况下仍然可以播放动画

4. **Culling Mode**
表示当游戏对象被**剔除**（摄像机看不见）时动画的播放状态，有三种选项：
    - **Always Animate**：总是播放动画
    - **Cull Update Transform**：停止动画播放，但是位置会继续更新（但会剔除 **IK**），视觉上和上一个没有区别
    - **Cull Completely**：完全禁用动画

# 创建一个简单的动画
看到这里想必各位读者已经跃跃欲试，想要创建第一个动画了吧~本小节我就会介绍一下如何用 Animation 窗口在 Unity 中直接制作和调试动画。

## 创建动画剪辑
我们可以通过选中要创建动画的游戏对象（例如相机），点击 **Create** 按钮为其创建一个动画剪辑和 Animator Controller。Unity 会弹出一个窗口询问你新动画剪辑的名称和你想要存放的文件目录。
![在这里命名](https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAcc4e932f6c132633ebc29027ca822db9.png)

然后我们就可以在 Animation 面板上看到新建的动画剪辑了。面板上有几个要素：
- **录制按钮**（红色圆圈）：点击后进入录制状态，所作的修改会被记录到动画中。
- **帧导航控件**：包含快退到底、后退一个关键帧、播放、前进一个关键帧、前进到底。
![](https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA1af42be6c49b72041e47ea0b5ca08edb.png)
- **当前所在的帧**
- **动画剪辑名称**：目前所编辑的动画剪辑，点击后可以通过底部的 `Create New Clip` 为该 Animator Controller 添加一个新的空白动画剪辑，也可以在这里切换想要编辑/察看的动画剪辑。
![动画剪辑们](https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAbb1e5c039550e99dadb4ecc2c10ad857.png)
- **采样率**：默认为 60，可以自行修改。默认的动画窗口可能没有展示采样率，可以在设置里打开。
- **过滤选择**：
- **添加关键帧**：在时间轴当前位置添加一个关键帧。
- **添加动画事件**：在时间轴当前位置添加一个动画事件，动画播放到该位置时会执行事件。
- **游戏对象（或其子对象）的动画化属性**：顾名思义，另外可以点击小三角形展开看详细的数据：
![相机旋转参数的详细数据（具体到轴）](https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA09af3dcca9607e395d39a8373728f86b.png)
- **曲线**：即 `Curves` 按钮，位于动画窗口的左下角。点击后将切换视图为曲线，可以点击 `Dopesheet` 按钮切换回来。
![曲线模式下的动画](https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAe7412903d1e45d0a186e7e613ba960ce.png)

点击红色的录制按钮，我们就可以在时间轴上为相机添加关键帧了（相信有过 Flash 制作经验的人都不会觉得陌生），例如改变相机的旋转数据：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAf2b46059e8e503abffa502ad6450f9c3.png" alt="白色点就是关键帧" />

录制好之后，记得再点一下录制按钮退出录制模式。Unity 会自动为这些关键帧之间的普通帧做插值处理，得到关键帧与关键帧之间的平滑过渡。想要看一看动画效果如何的话，直接点击 Animator 视窗内的播放按钮就可以看到了，无需运行游戏。

![相机动画试运行效果](camera.gif)

## 动画曲线
**动画曲线**就是我们在点击 Animation 窗口左下角的 `Curves` 按钮时所展示的界面，它是一个非常强大的功能，不同于关键帧，我们可以查看和编辑动画属性的值以及它们之间的**插值方式**。

![调整曲线](curve.gif)

我们可以在取信中找到很多**点**，这些点其实就是参数的各个**关键帧**。曲线的变化描述了参数的数据在两个关键帧之间的变化情况。我们可以通过点边上的两条伸出的**左右两条杠**（曲线的切线）来调整参数的**变化程度**。

右键关键点可以看到几个选项，默认情况下一般如下图所示：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAe18aa8f025e09d6dc6aaf1c064751305.png" width=200 />

上图显示了关键点的**切线类型**：
- **Clamped Auto**： 默认切线类型。
    - 自动设置切线，使曲线平滑地经过该关键点。当编辑该关键点的位置或时间时，自动调整切线，以避免曲线被过度调整。在 Clamped Auto 模式下，如果你手动调整了该关键点的切线，切线类型会被切换为 Free Smooth 模式。
    <img src="https://docs.unity3d.com/uploads/Main/AnimationClampedAutoTangents.gif" width=300 />
- **Auto**：这是一个遗留切线模式，保留该模式是为了向后兼容旧项目。
    - 除非你有特殊原因必须使用该模式，否则请使用默认的 Clamped Auto。当关键点被设置为该模式时，切线被自动设置，使曲线平滑地经过该关键点。但是，相比 Clamped Auto 模式，有两点差异：
        - 当编辑该关键点的位置或时间时，切线不会自动调整；切线仅仅在该关键点被第一次设置为 自动（Auto） 模式（初始化）时才会进行调整。
        - Unity 在计算切线时，不会考虑避免过度调整（即可能会导致过度调整）。
    <img src="https://docs.unity3d.com/uploads/Main/AnimationAuto.png" width=300 />
- **Free Smooth**: 拖动切线图柄来自由地设置切线。
    - 切线被锁定为**共线**，以确保平滑度。
    <img src="https://docs.unity3d.com/uploads/Main/AnimationFreeSmooth.png" width=300 />
- **Flat**: 切线被设置为水平（自由平滑的一种特殊情况）。
    <img src="https://docs.unity3d.com/uploads/Main/AnimationFlat.png" width=300 />
- **Broken**: 有时你可能不希望曲线平滑地通过某个关键点。想要在曲线中**创建剧烈的变化**，请选择 Broken 切线模式。
    - **Broken-Free**: 拖动切线图柄来自由地设置切线。
        <img src="https://docs.unity3d.com/uploads/Main/AnimationBrokenFree.png" width=300 />
    - **Broken - Linear**: 切线指向相邻的关键点。
        - 要创建直线型的曲线片段，请将切换的两边都设置为 Linear。下面的 3 个关键点都被设置为 Broken - Linear，以实现关键点到关键点的直线。
        <img src="https://docs.unity3d.com/uploads/Main/AnimationBrokenLinear.png" width=300 />
    - **Broken - Constant**: 曲线在两个关键点之间保持为一个常量值。左侧关键点的值决定了曲线片段的值。
    <img src="https://docs.unity3d.com/uploads/Main/AnimationBrokenConstant.png" width=300 />

如果觉得曲线形状不满意，也可以双击曲线，在理想的位置添加一个新的**点**，调整点的位置、切线类型和变化程度拟合出想要的形状。

## 骨骼动画
关于骨骼动画，我感觉麦扣老师的这一系列视频讲的很好，因此就不费笔墨了。

<iframe src="//player.bilibili.com/player.html?isOutside=true&aid=90884303&bvid=BV1E7411c7td&cid=155201542&p=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"></iframe>
<iframe src="//player.bilibili.com/player.html?isOutside=true&aid=91157831&bvid=BV1w7411F7qb&cid=155655804&p=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"></iframe>
<iframe src="//player.bilibili.com/player.html?isOutside=true&aid=91909822&bvid=BV1Y7411K7y6&cid=157166678&p=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"></iframe>
<iframe src="//player.bilibili.com/player.html?isOutside=true&aid=92995456&bvid=BV1iE411E7ou&cid=158773390&p=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"></iframe>

《一个月开发明日方舟》中的角色动画也都是基于骨骼动画制作的（剑气是通过 LineRenderer 实现的）：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA0d4829d78f3c07e5a95991a75f6f8b43.png" width=300 />

## 动画事件
**动画事件**（Animation Events）是 Unity 动画系统中的一个强大的功能，它可以被放在动画剪辑中特定的某一帧，当该动画播放至事件所在的这一帧时，就会触发相应的动画事件。

我们可以在需要的帧右键创建动画事件，也可以通过动画事件按钮图标创建。完成创建后，点击事件的小图标，就可以在检查器中为其绑定事件了。动画事件可以调用**挂载在动画对象上的脚本**里的公有方法，例如我在攻击动画中调用 `Attack` 方法：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA9c72d1de195b278c9e6e22a5683aff94.png" />

完成后。鼠标点击事件的小图标，就能发现事件的名字变成了方法名。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA96c381a527acd6b8c64a2d03d529fae4.png" />

# Animator窗口
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA072a06381ccbea7e8d6477005b33162d.png" alt="一个Animator" />

双击 Animator Controller 文件（或者在菜单栏中选择 Window > Animation > Animator）就可以打开 Animator 窗口中，在其中你可以看到各个动画剪辑的连接可视化界面。当我们创建了多个**动画状态**，就可以在中间形成**切换和过渡关系**（如上图）。除此之外，Animator窗口还包含了**动画层**、**动画参数**、**混合树**、**子状态机**等。由于篇幅有限，状态机的大部分内容会被我放到下一篇文章中介绍，尽请期待！

Animator Controller 是状态机的核心，后续的状态与过渡都将在这个界面中编辑。

## 状态（State）
在 Animator Controller 中，状态表示某个动画剪辑的执行。每个状态通常对应于角色的一个行为，比如“站立”或“行走”。

**状态的创建与设置**：

1. 添加状态：
在 Animator 窗口空白处右键，选择 Create State > Empty 或 From New Clip。
如果选择 From New Clip，Unity 会自动为状态生成一个新的 Animation Clip。

2. 状态的配置：
点击某个状态，在 Inspector 面板中可以设置：
Motion：绑定的 Animation Clip。
Speed：动画播放速度，默认为 1。

3. 默认状态：
一个状态会被标记为“默认”（橙色），表示动画初始播放的状态。
改变默认状态：右键某个状态 -> Set as Layer Default State。

## 过渡（Transition）
过渡定义了状态之间如何切换，比如从“行走”切换到“跑步”。过渡不仅决定切换的时机，还可以通过混合实现平滑过渡。

**创建过渡**：

1. 添加过渡：
右键某个状态 -> Make Transition，然后点击目标状态。
在状态之间会出现一个箭头，这就是过渡的可视化表现。

2. 过渡的配置：
选中过渡箭头，在 Inspector 面板中可以设置：
    - Has Exit Time：是否等待当前动画结束再切换。
    - Transition Duration：过渡的持续时间，影响动画的混合效果。
    - Conditions：指定触发过渡的条件（如参数变化）。

3. 条件的设置：
条件是过渡的重要部分，基于参数驱动：
    - 首先，在 Animator 窗口点击左侧 Parameters 面板，添加一个参数（如 float 类型的 Speed）。
    - 在过渡的条件中，设置“Speed > 1”作为触发条件。

4. 默认过渡设置建议：
通常开启 Has Exit Time，确保动画不会过早切换。
设置合理的 Transition Duration 以实现流畅过渡。

# 动画复用
当我们制作完成一个动画（例如 {% label 开门动画 green %}）之后，肯定希望它能作用于多个不同的“**门**”上实现**复用**，这样就不要为不同的门创建几乎同样的动画了，该怎么做呢？

以我们创建一个相机旋转动画为例，新建动画时实际上是创建了一个 `.anim` 文件，而这个文件本质上是一个 `.yaml` 文件。文件中记录了关于动画的数据，例如当我们修改了门的旋转 **Rotation** 时，就会在文件中找到一个叫 `m_EularCurve` 的项：

```YAML
m_Curve:
    - serializedVersion: 3
        time: 0
        value: {x: -25, y: 0, z: 0}
        inSlope: {x: 0, y: 0, z: 0}
        outSlope: {x: 0, y: 0, z: 0}
        tangentMode: 0
        weightedMode: 0
        inWeight: {x: 0.33333334, y: 0.33333334, z: 0.33333334}
        outWeight: {x: 0.33333334, y: 0.33333334, z: 0.33333334}
    - serializedVersion: 3
        time: 0.16666667
        value: {x: -25, y: 4, z: 0}
        inSlope: {x: 0, y: 24, z: 0}
        outSlope: {x: 0, y: 24, z: 0}
        tangentMode: 0
        weightedMode: 0
        inWeight: {x: 0.33333334, y: 0.33333334, z: 0.33333334}
        outWeight: {x: 0.33333334, y: 0.33333334, z: 0.33333334}
    - serializedVersion: 3
        time: 0.25
        value: {x: -25, y: 6, z: 0}
        inSlope: {x: 0, y: 18, z: 0}
        outSlope: {x: 0, y: 18, z: 0}
        tangentMode: 0
        weightedMode: 0
        inWeight: {x: 0.33333334, y: 0.33333334, z: 0.33333334}
        outWeight: {x: 0.33333334, y: 0.33333334, z: 0.33333334}
    - serializedVersion: 3
        time: 0.33333334
        value: {x: -25, y: 7, z: 0}
        inSlope: {x: 0, y: 0, z: 0}
        outSlope: {x: 0, y: 0, z: 0}
        tangentMode: 0
        weightedMode: 0
        inWeight: {x: 0.33333334, y: 0.33333334, z: 0.33333334}
        outWeight: {x: 0.33333334, y: 0.33333334, z: 0.33333334}
    m_PreInfinity: 2
    m_PostInfinity: 2
    m_RotationOrder: 4
path: Player_Camera
```

`time` 表示变换发生的**时间点**，`value` 代表该点的值，Unity 会自动使用插值算法计算两点中间的帧的值（如果未设置曲线）。注意到最后一项 `path` 的值是 **Player_Camera**，这代表动画是作用在动画根物体的 **Player_Camera** 子物体上的：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAf87a1d5292f7af2b23efbe3d5f952bc6.png" width=280 />

当我们想复用这个动画到其他相机上时，把相机命名为 “Player Camera”，把它设成一个空对象的子物体，再将动画放到相机的父对象上即可。在实现动画复用的过程中，命名、结构（对象层级）、属性能够对应上的话，就能够实现动画的复用了。

## Avatar
对于人形模型来说，由于不同的建模软件（MAYA、3DMax、Blender 等）的人物模型的骨骼名字甚至层级都可能并不相同，如果按照上面的逻辑为骨骼改名来实现复用的话，工作量也太大了。

Unity 为人形模型提出了一套解决方案：**Avatar**（替身系统）。当一个人物模型导入 Unity 之后，我们可以把模型的骨骼和 Avatar 对应起来：

1. 在模型的 `Rig` 面板，选择 `Animation Type` 为 **Humanoid**。
2. 设置 `Avatar Definition` 为 **Create from this model**。如果已经有配置好的 Avatar（例如成女、成男等），也可以选 **Copy**。
3. 点击 Avatar，可以看到 `Mapping` 面板中模型的骨骼已经自动对应上去了。如果没有的话，就需要麻烦动动手喽。

> 从优化角度考虑，因为已经有 Avatar 了，模型的骨骼就可以被删除了，因此可以勾选 `Optimize Game Objects` 选框。

为我们需要复用动画资产的人形模型**们**都创建这一套，然后找到动画资产，在动画资产的 `Rig` 面板将 `Avatar Definition` 设置为 **Copy from this model**，将源设置为最初模型的 Avatar。

最后，在模型的 Animator 组件中，将对应的 Avatar 挂载上去就可以了。

## IK系统
IK 即 Inverse Kinematic（反向动力学），研究的是如何通过手/脚/头等**肢体末端**来逆向控制其他骨骼的变换。在 Unity 等游戏引擎中，IK 往往比较简单，因此一般采用 **CCD IK**（Cyclic Coordinate Descent，循环坐标下降），从末端骨骼依次计算到根骨骼的旋转来实现。

在 Unity 中使用了 Avatar 系统之后，人物模型由于正向动力学的计算是从根节点出发的，因此在末梢会存在一定的偏移，这些偏移会导致动画看起来有点“怪”。这时候我们就可以通过设置末端 IK 的**权重**，类似于 footIK 那样，让模型实际的手脚更靠近 IK goal 所在的位置。

在动画机的任意一个层级中打开 `IK Pass`，Unity 就会在生命周期函数中计算 `OnAnimatroIK` 了：

```C#
// 左手 IK Goal 位置
public Transform leftHandIKPos;
// 右手 IK Goal 位置
public Transform rightHandIKPos;

private void OnAnimatorIK(int layerIndex){
    // 设置 IK goal 的位置和旋转
    animator.SetIKPosition(AvatarIKGoal.LeftHand, leftHandIKPos.position);
    animator.SetIKRotation(AvatarIKGoal.LeftHand, leftHandIKPos.rotation);
    animator.SetIKPosition(AvatarIKGoal.RightHand, rightHandIKPos.position);
    animator.SetIKRotation(AvatarIKGoal.RightHand, rightHandIKPos.rotation);
    // 左手 IK Goal 权重
    animator.SetIKPositionWeight(AvatarIKGoal.LeftHand, 1f);
    animator.SetIKRotationWeight(AvatarIKGoal.LeftHand, 1f);
    // 右手 IK Goal 权重
    animator.SetIKPositionWeight(AvatarIKGoal.RightHand, 1f);
    animator.SetIKRotationWeight(AvatarIKGoal.RightHand, 1f);
}
```

我们可以把 IK goal 的位置放在一个合适的地方，例如持枪下的右手以胸口作为参照、左手放在武器上。接下来在动画中调整 IK goal 的位置，找到合适的角度应用就好啦！