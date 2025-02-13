---
title: Unity动画系统：动画状态机
date: 2021-12-05 11:09:27
tags: ['动画','Unity','动画系统','教程','动画混合','BlendTree','经验分享','引擎功能']
categories: 引擎功能
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAfb5258df9e386e511358baa8e1159733.png
katex: true
---

在 Unity 的动画状态机（Animator）中，主要有以下几个核心组件，它们共同组成了动画系统的运行框架：
- **Animator Controller**：动画控制器
- **State**：状态，通常对应的是动画剪辑
- **State Machine**：状态机，用来组织和管理一组动画状态。
- **Transition**：定义两个状态之间的切换逻辑
- **Parameters**：动画参数，用于传递条件或数据
- **Layers**：层，支持在一个动画状态机中处理多个动画序列
- **Blend Tree**：混合树，根据一个或多个参数动态混合多个动画

# 状态机
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA7636edac5888063208fc591ee8f57c95.png" width=500 />

**状态机**就是Animator右侧大块区域所展示的“状态与转移”所组成的图（如上图所示）。

## 状态
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA8fd9ad65e656c5123d8b8ced1cea109a.png" alt="State的检查器面板" width=300 />

点击状态，我们可以在检查器中看到相关信息，包括：
- **tag**：动画状态也可以打标签，通过标签可以对状态进行管理。
- **Motion**：该状态播放的是哪段动画剪辑（或者混合树所管理的树）。
- **Speed**：动画播放速度。如果想通过脚本更改播放速度，可以勾选 `Parameter` 选框，为速度关联一个**浮点类型的变量**进行控制。
    - 值为 `0`：卡在第一帧不动
    - 值大于 `0`：正向播放
    - 值小于 `0`：倒放
- **Motion Time**：动画播放进度。我们同样可以勾选 `Parameter` 选框，为进度关联一个**浮点类型的变量**进行控制。
- **Mirror**：启用镜像播放。我们同样可以勾选 `Parameter` 选框，为进度关联一个**布尔类型的变量**进行控制。
- **Cycle Offset**：动画播放的偏移量。偏移不会影响动画剪辑，只是改变了开始播放的位置。我们同样可以勾选 `Parameter` 选框，为进度关联一个**浮点类型的变量**进行控制。
    - 值为 `0`：不偏移，直接从第一帧开始播放
    - 值为 `0.5`：偏移一半，从动画的中间一帧开始播放
- **Foot IK**：仅适用于具有 Avatar 的状态，该选项可以将人物模型的脚往 Unity Avatar 脚部 IK goal 的位置拉近，在一定程度上改善脚步动画的不自然。
- **White Defaults**：动画状态是否为**不通过动画设置**的动画**属性**（例如位置等）写入**默认值**。
    - **默认值**：在 `OnEnable` 时保存的各个属性的值
    - **默认值写入**：当动画片段中不存在某个属性字段（例如位置）时，如果开启了White Defaults，就会把默认值写进去。
- **Transitions**：该状态的转移状态列表。

双击状态，可以看到状态采用的动画剪辑本身。当动画播放出现诡异的循环的时候，往往是其中的 Loop（循环播放）忘记关闭了

## 转移
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA714634e3163c1ea23e5395fb756aeb50.png" width=300 />

转移允许状态机从一个动画状态**切换**或**混合**到另一动画状态。

许多萌新在刚接触动画系统的时候会发现两个状态之间的切换似乎有点问题：感觉状态与状态之间好像粘连在一起了一样，没有办法立即切换成功。这往往是转移的设置上出了问题。如果我们希望两个状态直接切换、不出现任何拖沓的话，应该取消勾选 `Has Exit Time` 和 `Fixed Duration`。

- `Has Exit Time`：退出时间。退出时间不是真实的时间，而是**动画播放的百分比**，表示在播放了 ExitTime% 之后，角色会和下一段动画进行“混合”，让动画的效果看起来不是那么突兀。
- `Fixed Duration`：固定时长。代表过渡时长是一个真实的时间。如果取消勾选，过渡时长将会变成百分比形式。
- `Transition Duration`：过渡时长。该状态与下一个状态过渡混合的时长。

我们可以通过转移检查器的可视化界面来调整这些数据：

<img src="transition.gif" width=300 />

## 子状态机
在上一篇文章中我们提到过“如果一个角色的状态数量非常大，难以管理该怎么办？”这就可以用到**子状态机**，将一部分状态挪到子状态机里，降低第一层状态机的复杂度。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAe51e3b8a1f93bcfddd32912d0a806aad.png" alt="子状态机" />

子状态机可以通过在状态机界面右键 `Create Sub-State Machine` 创建，内部和状态机一模一样，由 **Entry**、**Exit** 和 **AnyState** 三个标准状态组成。

# 动画参数
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAf04d2388f6cf53dd3335113f2cb8773e.png" alt="动画参数界面" width=250 />

动画参数位于 Animator 界面的左侧，我们可以点击面板右上角的**加号**来创建一个新的动画参数，支持：
- **Float**：浮点数
    - Greater：大于
    - Less：小于
- **Int**：整型
    - Greater：大于
    - Less：小于
    - Equals：等于
    - NotEqual：不等于
- **Bool**：布尔值
    - True：真
    - False：假
- **Trigger**：触发器，触发则引起状态转移

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAba37c4518e5e7504294fe1bd3ee865b0.png" alt="约束条件" />

点击除了默认转移之外的任意一条转移，我们就可以给转移加上**动画参数约束的条件**了。当满足条件时，动画机就会驱动当前状态转移至下一个状态。Unity 提供了一系列 API，允许开发者通过**代码控制动画参数**（Animator.SetTrigger、SetBool 等）。举个例子：当下落角色的高度接近地面，让角色播放准备落地的动画。这可以通过射线结合 Trigger 来实现。

在代码中，我们可以通过字符串来查询一个参数，也可以通过 `int` 类型的**哈希值**来查询，其效率更高，例如：

```C#
int param = Animator.StringToHash("speed");
this.GetComponent<Animator>.SetFloat(param, 0.5f);
```

# 分层动画
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAe82136244bb268f8a86939322343cc72.png" alt="层面板" />

在项目中，玩家所控制的角色通常具有**站立**、**走路**、**奔跑**等类型的动画。如果在开发的过程中，策划同学增加了一个枪械的玩法，那我们是不是要去制作**持枪站立**、**持枪走路**、**持枪奔跑**的动画呢？显然这样会大大增加美术组的工作量。

“层”（Layer）位于 Animator 面板的左侧，可以通过层面板的右上角加号按钮新建一个层。每个层级有**权重**（Weight）控制对最终动画的影响，分层动画可以用于分离身体的不同部分（如上半身播放攻击动画，下半身播放跑步动画）。层的**混合模式**有两种，一种是**覆盖**（Override），另一种是**叠加**（Addition）。对于动画层来说，**越下方的层拥有越高的播放优先级**，因此最下方的权重为 `1` 的覆盖层会完全盖住上方所有动画层正在播放的动画（如果没有设置 Avatar Mask）。叠加类型则顾名思义，是在当前展示的动画基础上按照权重叠加上该层的动画内容，需要动画存在动画曲线时才能作用。

![](layer.gif)

## 动画遮罩
动图里的例子（混合站立和走路）其实不是特别好，因为两个层的动画都对全身有影响，显得很奇怪，有没有办法让一个层播放身体一部分（例如上半身）动画、另一个层播放其他部分呢？这里就需要引入“**动画遮罩**”的概念了。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAc852648e0363719608ee332d9fced13d.png" alt="创建动画遮罩" width=300 />

对于这个例子来说，我们可以通过遮罩屏蔽**走路**动画的上半身，这样就不会影响到手臂了。应用遮罩之前，需要创建一个 Avatar Mask（是的，它只适合人物模型）文件，如上图所示。遮罩界面展示了一个绿色的人形，点击想要屏蔽的部分就会变成红色，表示动画不会播放红色遮罩区域，只会播放绿色区域。

点击层边上的齿轮符号，就可以为层添加动画遮罩啦。

{% note orange 'fas fa-lightbulb' %}
**Unity 的人形动画**

Avatar 就是人形动画，是 Unity 为人形模型动画复用推出的一套机制，Avatar Mask 也是作用在 Avatar 上的遮罩。下面这期视频介绍了 Avatar 与人形动画复用相关的知识点：

<iframe src="//player.bilibili.com/player.html?isOutside=true&aid=848600767&bvid=BV1GL4y1B7s1&cid=425760212&p=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"></iframe>
{% endnote %}

# BlendTree
混合树（BlendTree）是用来混合的一种**特殊的状态类型**，比如我们可以根据角色的速度来混合行走和奔跑动画，可以用来实现流畅的动画过渡，比如用动画机实现八方向的移动。为了使混合后的运动合理，要混合的运动必须具有**相似的性质和时机**（例如走路和奔跑都是移动腿部的动画，存在关联）。

右键新建 New Blend Tree，双击就可以进入混合树的编辑界面。以行走和奔跑为例：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA6c0c311e7d9903e5945d3cb5bf327a12.png" />

{% note info %}
虽然都提到了“混合”，但**混合树和过渡时的混合并不相同**：

- **过渡**：用于**在给定时间内**从一个动画状态平滑过渡到另一动画状态。
- **混合树**：允许通过不同程度**合并多个动画**来使动画平滑混合。每个运动对最终效果的影响由一个**混合参数**控制。
{% endnote %}

## 1D混合
1D 就是 1 Dimension，一维混合。动作的混合受到**一个参数**的影响，例如在上图中，参数从 `0` 到 `1` 变换的过程中会完成从 `Walk` 到 `Run` 的平滑过渡。

<img src="blend.gif" width=400 alt="1D混合走路与跑步" />

通过取消勾选 `Automate Thresholds` 勾选框，我们可以调整动画播放的切换阈值，以更好的调整过渡效果。在脚本中，为控制混合的参数（和动画参数是一样的）赋值（例如当前速度）就可以起到更改动画混合的效果，不过需要注意的是，混合的参数的范围在**0 ~ 1**，因此我们需要对输入做一些映射处理。

如何修改混合树中片段的播放速度呢？这和混合树面板上的“**Threshold**（阈值）”以及时钟图标（**播放速度**）有关。我们可以通过 **Compute Thresholds** 快速计算出各个动画片段的阈值。以“向后走-静止-向前走”为例，可以通过 z 方向（前后方向）上的动画速度计算阈值。假设我们希望动画以当前的 `1.5` 倍速播放，就可以在时钟图标一栏填入 **1.5/threshold**，并将阈值设置为 `1.5`（向后则是 `-1.5`）。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAf76c2dd67c976ee23efc5ea694f21cc1.png" width=400 />

## 2D Simple Directional
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA1d3c96b84c88cee3e11b5a8078cf9227.png" alt="2D Simple Directional" />

二维混合树又被称作**双变量混合树**，2D Simple Directional 是其中最简单的一种类型，它的缺陷在于同一个方向上不能有两个动作（原理会在下文中说明）。2D 混合树由**两个变量**影响，分别体现在**横轴**和**纵轴**上，它们组成的平面被称为“**参数空间**”。

2D 混合树比较常用的场景是八个方向的行走 / 奔跑等动画的混合。当我们启用了RootMotion之后，角色的移动是由动画控制的，因此从代码方面可能仅通过更改 X-Z 平面调整**横向** / **纵向**速度（这里就涉及到两个控制量，换句话说就是方向）来控制角色的移动方向，而不直接更改其 Transform。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA20938079ea508e49f6c14d0c80da38ef.png" width=200 />

根据动画带有的信息，点击 BlendTree 面板的 `Compute Position` 即可自动计算动画在混合树平面上所处的位置，点击红点即可检查参数不同时各个动画的权重，有小圆圈代表结果收到该剪辑影响，圆圈越大影响越大。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAc7f8e16e055d0d5a4c3ec0006f1878d0.png" width=200 />

需要注意的是，**图的中心（0，0）处需要有一个动画剪辑**（比如站立），否则图上的所有动画剪辑都会对输出结果造成影响，这和 **2D Simple Directional** 的**插值算法**有关，该算法会以原点为其中一点、从原点出发到其他动画剪辑形成的向量，和从原点到当前位置形成的向量中，顺、逆时针角度最小的两个点建立一个**三角形**，以当前点作为重心计算各个顶点的**权重**，从而得到各个顶点对当前的影响。

如果原点处没有动画片段的话，unity 会把原点的权重平均分配给剩下的所有剪辑，由此可见为了保证正确的效果，必须有一个点是原点。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAa5d94b719aa7791d0f5a0b978b96916a.png" alt="计算权重" width=500 />

前面提到的为什么“从原点出发的一个方向上不能有两个动画剪辑”，想必现在大家也能理解了，因为此时可能会出现三点共线的情况，就没有办法找到三角形计算权重了。同样的，**2D Simple Directional** 不允许原点出发的 180 度范围内都没有动画点，因为这样也找不到三角形。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA41e52ae58c28e43be037356651d711e7.png" width=200/>

## 2D Freeform Catasian
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA3495e871467624551889235b809c91aa.png" width=350 alt="结合了静息、行走、奔跑和转向的2D Freeform Catasian混合树" />

**2D Freeform Catasian** 也是一种 2D（双变量）混合树，但是它的权重计算算法和上一个小节介绍的不同，使用的是**梯度带算法**（Gradient Band Interpolation），适合需要根据**笛卡尔坐标系**下的**两个独立参数**（如转向和速度）进行动画混合的场景。动画片段在空间中呈网格状分布（如上图所示），Idle 位于原点，行走系列和奔跑系列动画的速度相同。

在梯度带算法下，参数空间中的动画点 `Pi`（xi, yi）对空间中的任意一点 `P` 都能造成一个影响值 `hi`，`hi` 的计算由以下公式得出：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA03b7e63aa40a2c313815b488ea141cac.png" width=250 />

以 `Pi` 为其中一点，遍历其他所有点 `Pj`，向量部分就是点 `P` 到 `PiPj` 所组成向量的**投影长度**与 `PiPj` **向量长度**的比值。该比值在一定程度上反应了 `P` 点到 `PiPj` 的距离关系。

该值被限制在 `0` 到 `1` 之间：
- 当超出`1` 时只能为 `1`，此时点 `P` 不受 `Pi` 影响；
- 当小于 `0` 时只能为 `0`，此时点 `P` 不受 `Pj` 影响。
- 除此之外，`P` 落在两条蓝线中间时，`Pi` 和 `Pj` 都能对点 `P` 造成影响：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAdfd6880d8675121ce678a889984e715f.png" width=350 />

于是我们发现，投影长度与向量长度的比值越大，`Pi` 对 `P` 的影响就越小，不过这样有点反人类。因此我们用 `1` 减去这个比值，来表示 `Pi` **对** `P` **的相对影响力**。从 `Pj` 到 `Pi`，在几何上组成了一条**影响力梯度带**，这就是该算法被称为梯度带算法的原因。

对空间中的其他 `n - 1` 个点，通过同样的方式就可以算出一系列**相对影响力**，接着我们取其中**最小的值**，这就是最终的**影响值** `hi` 了。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA5a4570c3482789a870c8befbf75cc58c.png" width=250 />

最终，点 `Pi` 对点 `P` 的影响权重 **w** 是 `hi` 除以其他所有点的**权重**的和。

### 梯度带算法的局限性
当我们希望内圈是行走动画、外圈是奔跑动画时，我们会发现不管采样点多接近奔跑动画的范围，行走动画始终会对其产生影响（还不小）：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAea9f4ffcf3bcdefad751bc548490fa8d.png" alt="内圈为行走，外圈为奔跑" />

因此如果希望制作一个融合了行走和跑步的多方向双变量混合树的话，**2D Freeform Catasian** 并不是一个合适的选择。

## 2D Freeform Directional
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAb7abaaa22c86ddfa484dcf871a3da696.png" width=350 alt="结合了静息瞄准、行走瞄准、奔跑瞄准的2D Freeform Directional混合树" />

**2D Freeform Directional** 采用的是**极坐标下的梯度带算法**。该算法采用**极轴**（半径分量）和**极角**（角度分量）来表示二维坐标系下一个的点，**2D Freeform Directional**需要根据**方向**（垂直或水平）和**幅度**混合动画，因此动画片段按照方向分布，如上图所示。该类型混合树适合用于制作角色的 8 向运动。

- **半径分量**：两点**模长的差**除以两点模长的**均值**（抵消长度单位的影响）。对 `PPi` 来说有点不一样。
- **角度分量**：两点的**夹角**乘以 `α`。`α` 是用来调整半径分量和角度分量在计算中的**重要程度**。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA6248f478857acbb58d0c2f8fbe7d39bf.png" alt="半径分量、角度分量的定义" width=280 />

接下来的算法就很类似了，取所有**相对影响力**中的最小值作为 `Pi` **对** `P` **的影响力**，再除以**所有相对影响力的和**，得到点 `Pi` 对于点 `P` 的权重。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA573cc9d151168c7dea49303f1c67a3d7.png" width=250 />
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAafc20dd17063136282f2a90e5d1b45d1.png" width=250 />

极坐标下的梯度带算法和一般的梯度带算法的区别在于：

1. 当两点**半径分量**相同（与原点距离一致，处在一个同心圆上）时，梯度带仅受到**角度**影响，梯度带将沿角度方向平均分布；
2. 当两点**角度分量**相同（与原点的角度相同，处在同一条直线上）时，梯度带仅受到**距离**影响，梯度带将展示为一个圆环；
3. 其他情况下，梯度带会呈现出**阿基米德螺旋**（Archimeddean spiral）的形状：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA002cd1833b2c09a14e0e47ef19ec1e3a.png" />

**2D Freeform Directional** 可以更精确地对不同方向和速度进行插值，原点必须有动画剪辑存在。

## Directional
**Directional**（直接）允许用户**直接通过一系列参数控制**每个动画剪辑的权重，一般用在面部形状混合上（即 **Blend Shape** 的概念）。例如为面无表情、微笑、愤怒，可以通过表情参数进行控制。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA0a36d8b2d9f34ff3aa3125908be07b2b.png" alt="Unity官方举例的直接混合树" />

# RootMotion
有的时候，动画和程序的逻辑是分开的，例如程序控制玩家以 1 m/s 的速度向前移动，但动画师傅是按照 0.5 m/s 制作的动画，此时就会出现明显的“滑步”问题：

<img src="https://p2.itc.cn/q_70/images03/20231031/90e07766f3c54109b7daafa937051692.gif" alt="恶搞一下，总之给人的感觉就很不真实" />

在这种模式下，即使动画中的角色“好像在移动”，但实际上点击角色就会发现角色的 Transform 根本没有发生任何改变，因此开发者只能靠代码驱动角色移动（有点像动画刚体）。而 RootMotion 可以让动画驱动游戏对象移动（而不是代码），这使得运动更符合物理逻辑。RootMotion 通过**相对位移**和**相对转角**（四元数）移动游戏对象，相对移动的情况还和缩放有关（因此有缩放曲线会影响开启了 RootMotion 动画的性能），它是通过将位移、旋转、缩放矩阵相乘来实现的。

<img src="https://pic3.zhimg.com/v2-97e48d42832f2b659838b0f61f4dd954_b.webp" width=500 />

要想启用神奇的 RootMotion，需要按照以下步骤：
1. 首先，需要在 Animator 组件上勾选 `Apply Root Motion` 选框启用 RootMotion。
2. 接着，在模型的检查器的 **Rig** 面板，将动画类型设置为 `Humanoid`（人形）。这一步会为模型创建一个 Avatar，并将模型的骨骼映射到 Avatar 上。
3. 将动画的 **Rig** 同样设置为 `Humanoid`，但将 **Avatar Definition** 设置为 `Copy From Other Avatar`，将源指定为模型的 Avatar。
4. 为 Animator 组件的 Avatar 指定第二步中创建的 Avatar。
5. 现在动画就可以作用到角色的 Transform 上了。

如果动画不是 `Humanoid` 形式，但也想启用 RootMotion 该怎么办呢？Unity 也为 `Generic` 形式的动画设置了 RootMotion，在遮罩情况下我们只需要为其创建 Avatar（`Generic` 动画也有 Avatar，但和 `Humanoid` 的不同），为 Avatar 指定一根**根骨骼**（该骨骼唯一的作用就是用来记录模型的位移和旋转信息，是其他所有骨骼的根骨骼），其后的操作就和上述步骤相同了。

既然通过动画控制移动，那我们的脚本还有什么用呢？其实 RootMotion 有时并不完全可靠，脚本除了用来设置动画参数、控制状态之间的转换、修改动画播放速度之外，还可以通过调用 `OnAnimationMove()` 方法、将动画机的速度同步到物理引擎（**刚体**）的速度上，起到类似 RootMotion 的效果控制角色移动。RootMotion 要解决的问题本质是动画与实际表现的同步（即“防止脚底打滑”），而不是控制玩家位移，因此真正移动的部分交还给物理。

## 动画烘培
有的时候，一些动作可能会改变角色的高度（在 Unity 中代表 Y 轴）信息，例如跳跃。如果在平地上播放 “**跳跃 -> 静息**” 的动画，就会发现玩家停留在了一个较高的位置没有落下，这是因为角色的 Transform 完全受到了动画驱动。多数情况下，我们希望动画的效果符合物理规则，好在 Unity 中存在 **Bake Into Pose** 功能可以帮助我们解决这个问题。

在动画的 **Animation** 面板中，我们可以找到一系列的“烘培（Bake）”：
- **Root Transform Rotation**：将旋转变换烘培到动画上
- **Root Transform Position（Y）**：将 Y 轴（高度）上的变化烘培到动画上
- **Root transform Position（XZ）**：将 XZ 面（水平面）上的变化烘培到动画上

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA21db28cff564a90a30da7cd4ae7704d2.png" alt="image.png" title="image.png" />

勾选烘培将会停止动画影响到模型在旋转 / Y轴 / XZ 平面上的变换，将其变为骨骼动画（Pose）的而不是 RootMotion 的。上图的面板上有一些奇怪的“红绿灯”，它们表示的信息是**该动作是否适合该类型的烘培**。要将启用了 RootMotion 的动画变为完全不影响模型 Transform，只需要将上述选项全部烘培即可：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA5cab3f6e9a3bfddcdad491645870ea62.png" width=300 />

烘焙界面还有两个值：**Based Upon** 和 **Offset**。前者决定了参数在动画第一帧的值取决于什么（是**根骨骼 / Avatar 计算出的数值**还是**动画的原始数值**）。我们可以通过设置 **Offset** 修改基准值的偏移量。

{% note info %}
**Avatar 计算出的数值是什么？**

如果我们采用的模型与动画被设置成了 `Humanoid` 并启用 **Avatar**，就会在模型身上和地面上看到一个蓝色的小球（如果没有显示的话，需要在 Gizmos 里打开 Animator 的相关选项），这就是 Avatar 计算出来的“**根**”，类似于 `Generic` 动画中指定的**根节点**。通常，这个计算出的重心接近于人物模型 **Hips 骨骼**的位置。
{% endnote %}

{% note warning %}
**特别注意：RootMotion 结合混合树的潜在问题**

当我们开开心心地把一个使用了 RootMotion 动画片段的混合树添加到两个角色模型上时，我们可能发现一件事情：**两个角色的行走速度并不一样**。这可怎么办呢？对于一个游戏来说，如果可爱的萝莉角色跑图跑的比成女角色要慢的话，可能玩家就会有声音了。
在 RootMotion 里，这种问题产生的原因是 RootMotion 的相对位移/旋转计算是参照**标准 Avatar** 的缩放大小来的，如果一个模型的 Avatar 大小与标准 Avatar 不符，那么它的相对位移/旋转也会被放缩，一个只有标准 Avatar 80% 大的模型的相对速度也是标准的 80%。为了解决不一致的问题，我们可以先通过脚本获取模型的大小：
```C#
animator.humanScale;    // 这是 Humanoid 才有的
```
然后将混合树的速度parameter（需要先打开勾选），将值设为 `1` 除以模型大小，就能让不同模型在混合树中播放RootMotion动画的效果相同了：
```C#
animator.SetFloat("xxx", 1 / animator.humanScale);  // blendtree 的 speed
```
{% endnote %}

# 动画机覆写
想象一个画面：我们现在有一系列的友方角色，角色的动画控制器都有**行走**、**奔跑**、**普通攻击**和**特殊攻击**四种状态。对于每个角色来说，行走和奔跑的动画都差不多，但普通攻击和特殊攻击则是定制化的动画。在这种情况下，我们看上去似乎不能用一个 Animation Controller 来控制所有角色，但为每个角色创建一个专属的 Animation Controller 并一步一步调整状态和转移听上去是一件很费时费力的事情（特别是动画数量很多的时候，这还很容易导致出错），该怎么办呢？这时我们就可以考虑使用**动画机覆写**（Animator Override）。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA9b62d3d5f2c21c5f2ba316d56a45cc10.png" alt="创建一个动画机覆写" width=250 />

在合适的文件目录位置右键，就能找到 `Animator Override Controller` 选项，点击它可以创建一个动画机覆写。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA5fee83d028da6384ec70c2954a98f9fc.png" />

可以为动画机覆写指定一个原始版本的动画机（例如前面例子中的角色动画机），然后就可以开始覆写操作了。为想要覆写的动画剪辑（例如特殊攻击）指定 Override 的动画剪辑，就完成了动画机覆写。