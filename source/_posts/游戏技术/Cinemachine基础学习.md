---
title: Cinemachine基础学习
date: 2023-03-12 12:22:51
tags: ['Cinemachine','Unity','相机','控制','角色','3C','经验分享','引擎功能']
categories: 引擎功能
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA8ba9748cd63f7a972eae71287b91c644.png
---

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAe5bca3b0f2b9fc488f4bd2379e92e636.png" />

相机的控制与管理是游戏体验中相当重要的一部分，好的镜头语言能够让玩家更容易地进入心流，更好地体验游戏。相机属于游戏 **3C** 中的 **Camera**（另外两 C 分别是**角色**和**控制**），其地位不言而喻。在本文中，我将探索如何通过 Unity 的 Cinemachine 实现相机跟随的一系列功能。

Cinemachine 是 Unity 提供的**高级相机系统**，用于创建平滑、智能的摄像机跟随和过渡效果。Cinemachine 能够自动适应游戏场景，无需手动编写复杂的相机控制代码，让开发者专注于游戏玩法。要想在项目中使用 Cinemachine，需要从 Package Manager 中安装 Cinemachine 包。

导入 Cinemachine 包之后，在场景中创建一个**虚拟相机**，相机的视野立即产生了变化。观察视窗我们会发现游戏原本的主相机多出了一个图标，点开一看是一个名为“**Cinemachine Brain**”的组件造成的。自动添加了 **Cinemachine Virtual Camera** 脚本的虚拟相机和添加了 **Cinemachine Brain** 的主相机一起，组成了 Cinemachine 的基本运行逻辑。

好那么问题来了，为什么我们创建了一个“虚拟相机”？虚拟相机是什么，它有什么作用呢？

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAc29e6bb5a9885f9cf9b3f87332b440ff.png" width=200 />

在场景面板右键新建对象时，可以在 Cinemachine 选项下找到好几种不同类型的 Camera。其实这几种相机都是**相机模板**，开发者可以定义相机的各种属性和配置。而创建完这些相机之后，原本场景中的相机就无法调整了，因为它的**属性和行为被虚拟相机接管**了。

新建面板下的几种不同的 Camera 对应了几种不同的相机模板，例如下表：

| 相机类型                          | 功能                                                                                  | 适用游戏类型                              |
| ----------------------------- | ------------------------------------------------------------------------------------- | ----------------------------------------- |
| Virtual                       | 基本虚拟相机，通过跟随和注视目标来控制主摄像机视角，可实现平滑运动和镜头过渡           | 通用：适用于大部分3D/2D常规游戏，如TPS、横版游戏  |
| FreeLook                      | 自由视角相机，允许玩家通过多轨道混合自主旋转视角，提供三重高度镜头                    | 3D动作、冒险及角色扮演游戏                   |
| Blend List                    | 预设混合列表相机，根据条件或顺序在多个虚拟相机之间平滑切换，实现多镜头混合              | 适用于需要过场动画或镜头切换的游戏              |
| State-Driven                  | 状态驱动相机，根据动画状态机状态自动切换相机视角，跟随角色动作变化                      | 动作驱动类游戏，如动作RPG                    |
| ClearShot                     | 智能选择最佳视角相机，在多个虚拟相机间自动挑选最清晰画面，并进行无缝切换                 | 体育、竞速及实时竞技类游戏                   |
| Dolly Camera with Track       | 沿预定义轨道平滑移动的相机，可实现电影级动态跟踪和镜头过渡                              | 过场动画、电影场景                          |
| Dolly Track with Cart         | 模拟推车在轨道上平稳运动，提供精确的位置和速度控制                                     | 过场动画、导览镜头及动态场景                  |
| Target Group                  | 自动跟踪并调整多个目标的视角和缩放，确保所有目标都在画面内                              | 多人、团队竞技、战术及策略游戏                |
| Mixing                        | 整合多个虚拟相机的输出，实现复合镜头效果，通过混合不同画面创造独特视觉体验                | 电影特效、复杂视角切换和混合镜头场景            |
| 2D                            | 专为2D游戏设计的相机，提供平滑跟随、适当缩放和场景边界处理                              | 2D平台、横版卷轴和横版格斗等2D游戏             |

作为 Cinemachine 的基础教学，本文自然从最基本的虚拟相机开始讲起。在实际的开发中，开发者可以根据自己的需要选择合适的相机，还可以**创建多个虚拟相机**并在它们中间切换。以达到最佳镜头效果。

接下来，先看看 Cinemachine 的一些基本组件吧！

# Cinemachine Brain
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA12726556aaeb38907e4f8ac31d5e7267.png" alt="CinemachineBrain" width=350 />

在创建完虚拟相机（或其他相机模板）后，场景中的主相机被自动挂载上了一个 Cinemachine Brain 组件（如上图所示），上面的几个字段分别是：

1. **Live Camera**：正在使用哪个虚拟相机的配置。
2. **Live Blend**：当前虚拟相机的切换过程。如果要从一个虚拟相机转换到另一个虚拟相机，该字段就会显示转换的进度。
3. **Show Debug Text**：勾选后会在 Game 视图左上角显示当前所使用的虚拟相机以及相机切换过程（如有）。
4. **Show Camera Frustum**：在 Scene 视图中显示当前相机视锥体。
5. **Ignore Time Scale**：表示相机行为是否受 TimeScale 影响。
6. **World Up Override**：在相机发生旋转的时候，如果相机本地坐标下的 y 轴与世界坐标下的 y 轴之间的夹角大于 90° 时，相机会自动旋转并调整该夹角小于 90°。如果不希望这种自动旋转发生，可以用场景中对象的 y 轴覆盖世界坐标的 y 轴。
7. **Update Mathod**：虚拟相机行为的同步方式
    - **Fixed Update**：与物理引起同步
    - **Late Update**：与画面绘制同步
    - **Smart Update**：Unity 根据实际情况选择如何同步
    - **Manual Update**：开发者自行编写代码控制同步
8. **Blend Update Method**：两个虚拟相机之间的切换行为
    - **Fixed Update**：与物理引起同步
    - **Late Update**：与画面绘制同步
9. **Default Blend**：默认的相机切换模式
    - **Cut**：瞬间切换
    - **Ease In Out**：淡入淡出
    - **Ease In**：淡入匀出
    - **Ease Out**：淡出匀入
    - **Hard In**：加入切入
    - **Hard Out**：加速切出
    - **Linear**：匀速出入
    - **Custom**：自定义切入、切出曲线
10. **Custom Blends**：通过建立资源文件的方式为不同的相机制定不同的切换模式。
11. **Events**：相机的事件触发
    - **Camera Cut Event**：相机瞬间切换（Cut）模式时触发。
    - **Camera Activated Event**：相机切换（除了瞬间切换）的第一帧时触发。

# Cinemachine Virtual Camera
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAbd9fa500772df9cfe7e3253b83a98267.png" width=350 />

前面说到，我们创建了一个**虚拟相机**，这是 Cinemachine 所提供的几种相机模板（或者说相机的“**配置文件**”）中最基础的一种。每种相机模板在创建时都附带了一个 Cinemachine 有关的脚本，对于 Virtual Camera 来说就是 **CinemachineVirtualCamera.cs**。该脚本的几个字段分别是：

1. **Status**：相机状态，分为以下三种状态。当点击边上的 `Solo` 按钮之后会立即激活该相机，方便开发者进行调试。
    - **Live**：正在被主相机使用
    - **Standby**：待机，未被主相机使用但仍在计算，刷新频率取决于Standby Update字段的设置
    - **Disable**：禁用
2. **Game Window Guides**：游戏窗口引导，在游戏界面显示虚拟相机的提示信息，包括安全区等。
3. **Save During Play**：勾选后，在 Play 模式下对虚拟相机的修改将被保存。
4. **Priority**：优先级。Cinemachine 会使用高优先级的相机，但在 Timeline 中无效。
5. **Follow**：可以赋值一个游戏对象，相机会跟随该对象。
6. **Look At**：可以赋值一个游戏对象，相机会看向该对象。
7. **Standby Update**：相机处于待机状态时，位置旋转等信息以什么样的频率刷新。有三种模式可选：
    - **Never**：永不刷新
    - **Always**：刷新频率与激活的相机一致
    - **Round Robin**：轮替刷新，Unity 会在每一帧选择一个处于 standby 状态的相机进行刷新
8. **Lens**：镜头设置，包含以下五个参数：
    - **Vertical FOV**：视场大小（Field of Viewport）
    - **Near Clip Plane**：近裁平面距离
    - **Far Clip Plane**：远裁平面距离
    - **Dutch**：斜角镜头角度
    - **Advance**：无（不建议设置成无）or 透视相机 or 正交相机 or 物理相机。上面四个参数是当相机设置为透视时有的，如果设置成另外两种相机会略有不同，这里就不展开了。
9. **Transitions**：虚拟相机切换时的属性设置，例如：
    - **Blend Hint**：相机切换的行为具有怎样的**物理特征**
        - **None**：直接对两个相机的角度和位置做插值，实现平滑过渡（线性）
        - **Spherical Position**：球形切换，以 LookAt 的对象为球心进行切换，变换在一个球面上
        - **Cylindrical Position**：圆柱切换，，以 LookAt 的对象为中心进行切换，变换在一个圆柱体的平面上（水平是圆，垂直是线）
        - **Screen Space Aim When Target Differ**：当 LookAt 的对象不一样（Target Differ）时，在**屏幕空间**做切换，切换时在位置上使用相机的世界坐标系、在旋转上根据两个相机所构成的屏幕空间的夹角做切换
    - **Inherit Position**：继承位置，即切换发生时，该虚拟相机移动到相机的位置
10. Body：涉及相机的跟随行为
11. Aim：涉及相机的瞄准行为
12. Noise：相机晃动，可以为其添加柏林噪声
13. Extensions：相机能力拓展

由于Body、Aim涉及的内容比较多，接下来将分小节介绍。

## Body
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA9ffbb8288e9ee52d02cb4ce07ef92354.png" width=350 />

Body 控制了虚拟相机的跟随行为（对 Follow 字段对象的跟随），默认选择的模式是 **Transposer**。除了 **Transposer** 之外，还有六种跟随模式：

| 类型                     | 功能                                                                              | 适用场景                                 |
|-------------------------|-----------------------------------------------------------------------------------|----------------------------------------------|
| None                    | 不执行自动跟随，不进行位置计算，需完全手动控制相机位置。                              | 固定镜头、特殊自定义相机控制场景               |
| 3rd Person Follow       | 采用第三人称跟随方式，自动保持预设的目标相对偏移位置，并内置平滑缓动效果。             | 第三人称动作、冒险和角色扮演游戏               |
| Framing Transposer      | 自动调整相机位置以确保目标在画面中保持合适构图，支持动态调整和缓动以提升视觉表现。         | 需要强调画面构图的动作、射击或策略游戏           |
| Hard Lock To Target     | 直接锁定目标位置，不加入平滑过渡，确保相机始终严格跟随目标的指定偏移。                  | 竞技、格斗或需要精准锁定目标的实时对战场景       |
| Orbital Transposer      | 允许相机围绕目标进行环绕旋转，通过调整角度和距离实现自由视角变换。                      | 探索、模拟、动作和角色扮演游戏中需要多角度观察目标 |
| Tracked Dolly           | 将相机沿预定义的轨道平滑移动，通常结合轨道设计实现电影级的镜头运动效果。                | 过场动画、电影式镜头和定点轨道移动场景            |
| Transposer              | 提供灵活的跟随控制，通过自定义偏移和缓动参数平滑跟随目标，适合各种跟随需求。             | 各类3D游戏，如第三人称、动作或策略游戏           |

以 **Transposer** 为例，我们来看看它的几个属性：

1. **Binding Mode**：相机与目标的绑定模式，会影响相机的位置（但不**会改变相机自身的旋转**），有以下几种：
    - Lock To Target On Assign：在添加（assign）目标对象时变化到目标**本地坐标系下**的相对位置（follow asset）上，之后锁定位置关系（lock），只跟随移动，不受目标旋转影响。
    - Lock To Target With World Up：在添加目标对象时变化到目标**本地坐标系下**的相对位置上，之后锁定位置关系，不仅跟随移动还会绕目标的**偏航轴**（Yaw）旋转。
    - Lock To Target No Roll：在添加目标对象时变化到目标**本地坐标系下**的相对位置上，之后锁定位置关系，不仅跟随移动还会绕目标的**偏航轴**和**俯仰轴**（Pitch）旋转。
    - Lock To Target：在添加目标对象时变化到目标**本地坐标系下**的相对位置上，之后锁定位置关系，不仅跟随移动还会绕目标的**偏航轴**、**俯仰轴**和**横滚轴**（Roll）旋转。
    - World Space：在添加目标对象时变化到目标**世界坐标系下**（目标在世界坐标系下移动 offset 距离）的相对位置上，之后锁定位置关系，只跟随移动，不受目标旋转影响。
    - Simple Follow With World Up：相机不会沿着任何一个轴旋转，同时尽量保持x轴方向不移动。
2. **Follow Offset**：相机与目标的固定偏移量，默认值是位于目标身后 10m 的位置。相机会尽量维持这个距离
3. **Dampings**：阻尼系数，可以理解为相机移动延迟，阻尼越大相机移动延迟越大

## Aim
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA4c60a4e8642989a5c81fa0c0515437fa.png" width=350 />

Aim 即“瞄准”，控制了虚拟相机的旋转行为（对 LookAt 字段对象的视角调整），默认选择的模式是 **Composer**（保证相机“看向”目标对象）。除了 **Composer** 之外，还有五种瞄准模式：

| 类型                    | 功能             | 适用场景                  |
|-------------------------|--------------------------------------------------------------------------------------------------|---------------------------------------------------|
| None                    | 不进行自动瞄准调整，相机不会主动定位目标，需要由其他系统或脚本控制视角。                             | 适用于自定义相机控制或特殊场景下需要手动调控的游戏。           |
| Composer                | 自动调整相机构图，根据预设的构图指南平滑地将目标保持在画面中理想的位置，同时支持平滑过渡。              | 适用于大多数动作、冒险、角色扮演等追求稳定构图和视觉体验的游戏。   |
| Group Composer          | 针对多个目标进行构图优化，自动平衡画面中目标群体的位置和比例，确保整个群组都能良好地呈现在视野中。          | 适用于多人、团队竞技、策略以及需要同时关注多个对象的游戏场景。       |
| Hard Look At            | 强制相机始终直视指定目标，不进行平滑调整，确保目标始终位于画面中心，提供即时、精准的视角锁定。             | 适用于竞技、格斗等需要精准锁定目标的实时对战或动作游戏。            |
| POV                     | 允许玩家通过输入实时控制相机的视角，实现第一人称视角下的自由瞄准和观察，视角完全由玩家决定。               | 适用于第一人称射击、探索以及模拟类游戏，强调玩家视角控制的体验。        |
| Same As Follow Target   | 相机的瞄准方向完全与跟随目标保持一致，不做额外的构图调整，简单直接地复制目标的朝向。                     | 适用于不需要额外构图调整，只需简单跟随目标方向的第三人称游戏。         |

以 **Composer** 为例，我们来结合游戏窗口看看它的几个属性：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA31569a321169b3fda885e51bd474593e.png" width=350 />
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA67be4d4662e3830ae6b6685d7f807630.png" alt="Composer展示的debug界面" />

1. **Track Object Offset**：图上的黄色小方块，代表与 LookAt 目标（以下简称目标）之间的**偏移**。
2. **Lookahead Time**：根据当前目标的移动速度推测出一个偏移量。如果运动很不规律，该选项可能放大随机效果。
3. **Lookahead Smoothing**：让 LookAhead 算法的预测更加平滑，适合运动变化较大时
4. **Lookahead Ignore Y**：勾选后忽略目标在垂直方向上的偏移预测
5. **Dampings**：阻尼
6. **Screen XY**：代表 **DeadZone** 的中心位置，（0.5，0.5）表示屏幕中心。DeadZone 是上图中间的透明区域，用于**减小相机的敏感程度**，只有当黄色小方块（目标）离开 DeadZone 时相机才会转动
7. **DeadZone Width/Height**：DeadZone 的宽度和高度
8. **SoftZone Width/Height**：**SoftZone**（上图蓝色区域）的宽度和高度。当黄色小方块（目标）位于 SoftZone 内时，相机会转动使其回到 DeadZone（即瞄准）
9. **Bias XY**：SoftZone 位置的偏移量
10. **Center On Activate**：勾选时，虚拟相机激活时目标会出现在屏幕中央，反之会出现在 DeadZone 的边缘

上图中，在 SoftZone 之外还有一部分红色（上图中有点偏灰）的区域，Cinemachine 会保证黄色小方块（目标）一定不会落入该区域内。

# 参考资料
https://www.bilibili.com/video/BV1Ur4y1b7ff
