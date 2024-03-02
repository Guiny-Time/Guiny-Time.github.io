---
title: Corgi Engine快速指南(1)——如何搭建第一个关卡？
tags: ['游戏编程','经验分享','CorgiEngine','Unity']
categories: CorgiEngine
date: '2024-02-28'
copyright_author: 时光
cover: https://img13.360buyimg.com/ddimg/jfs/t1/164196/2/42865/31836/65e15815Feea0380b/d3400ea9ca2bce3e.jpg
---

# 首先得有一个场景

<img src="https://img10.360buyimg.com/ddimg/jfs/t1/106900/40/48267/18613/65df33f1F17526d0a/d62fc47aa4fbaaa2.jpg" alt="一个最基础的关卡" title="一个最基础的关卡" />

{% btn 'https://corgi-engine-docs.moremountains.com/',引擎文档,far fa-hand-point-right,blue center larger %}
{% btn 'https://corgi-engine-docs.moremountains.com/API/index.html',引擎API文档,far fa-hand-point-right,red center larger %}
{% btn 'https://www.youtube.com/watch?v=3BRDH7H3bcA&list=PLopcAUHZHov3s5Eo9GOWRwny9UBjuBIDg&index=3',视频教程,far fa-hand-point-right,purple center larger %}

在Corgi Engine中如何搭建最简单的场景？引擎已经为我们准备了配置好基础数据的预制件，我们可以直接拖入新建场景中，一个场景最少所需的预制件分别是：

{% tabs 一个场景最少所需的内容 %}
<!-- tab GameManagers -->
- 路径：CorgiEngine/Common/Prefabs/LevelManagers/GameManagers

- 包含了Game Manager、Sound Manager、Achievement Rules 和 Time Manager，以组件的形式添加在物体上。

- 负责处理得分、时间系数（游戏时间快慢）、暂停、设置解锁成就/增加进度的条件等功能。如果不需要其中的某些管理器，可以删除。
<!-- endtab -->

<!-- tab UI Camera -->
- 路径：CorgiEngine/Common/Prefabs/GUI/UICamera
    
- 不论游戏是否有UI，UI相机都需要加入场景中，因为预制件上附加了InputManager，这是玩家控制角色的关键组件（我也不知道为什么引擎不把这个mgr放到GameManagers上）。
    
- 你可以把所有GUI相关的元素放到这个相机底下的Canvas里。
<!-- endtab -->

<!-- tab LevelManager -->
- 这是个例外，我们无法找到LevelManager预制件，所以需要在场景中自行创建。
    
- 新建空物体，然后添加“LevelManager”脚本即可。在脚本中可以设置游戏关卡的边界
<!-- endtab -->

<!-- tab Camera -->
- 路径：CorgiEngine/Common/Prefabs/Camera/RegularCamera
    
- 我们还需要往游戏中加入相机，这样才能看到关卡样貌。Corgi Enging提供了名为CameraController的脚本，可以追踪我们放在游戏中的角色，让视觉效果更好。
    
- 引擎提供了两个相机，其中MainCamera支持后处理效果，RegularCamera实现了常规相机的功能。
<!-- endtab -->

<!-- tab Level -->
- 一个简单的平台即可，这样玩家就可以有地方站立了。
    
- to do so，创建一个square（或者任何2d形状），**将Layer设置成Platforms（否则玩家会无视这些图形）**，再添加collider 2d即可。
<!-- endtab -->
{% endtabs %}

# 然后得有一个角色
{% btn 'https://corgi-engine-docs.moremountains.com/API/index.html',引擎文档,far fa-hand-point-right,blue block center larger %}

我们最好按照官方文档推荐的结构（也就是以下方式，将sprite和控制相关组件分开）创建新角色，这有助于统一角色层级并方便后续管理（当然也可以按自己想的来，问题不大）：
1. 创建空对象 **MyCharacter**，将 Tag 和 Layer 设置成“Player”
2. 在 **MyCharacter** 节点下创建 **MyModel** 空节点并加上 *SpriteRenderer* 组件。该组件用于渲染角色的sprite
3. 在 **MyCharacter** 上添加 *Character* 脚本，可以找到一个叫 <u>AutoBuildPlayerCharacter</u> 的按钮，它可以帮助你自动创建一系列角色功能。先不要点
<img src="https://img12.360buyimg.com/ddimg/jfs/t1/120178/36/43284/12461/65e07a19F21b7af86/ec9d3b07c17092f1.jpg" alt="image.png" title="AutoBuildPlayerCharacter按钮" width=300 />
4. 将 **MyModel** 赋值到 **MyCharacter** 的 *Character* 脚本的 <u>CharacterModel</u> 字段上（如果有动画，记得在Character Animator上也赋值）
<img src="https://img12.360buyimg.com/ddimg/jfs/t1/160775/27/42499/9365/65e07b91Fc329b5a2/19ca93c7aa64851d.jpg" width=300 />
5. 如果你刚才没有点 <u>AutoBuildPlayerCharacter</u> 按钮的话，现在在 **MyCharacter** 上添加 *BoxCollider2D*（<u>x 偏移量</u> 设置为零） 、 *Rigidbody2D*、*CharacterLevelBounds*（控制玩家触边后的反应） 和 *CorgiController* 组件。确保 *BoxCollider2D* 和 *Rigidbody2D* 是这样的：
<img src="https://img12.360buyimg.com/ddimg/jfs/t1/231698/30/13893/31735/65e07b20F159bdd2a/2af26801c5b19700.jpg" alt="image.png" title="BoxCollider2D和Rigidbody2D的正确配置" width=200 />

现在，你已经创建好了一个角色的雏形了，你可以选择将其进一步设置为**可玩角色（玩家）**或**AI（敌人等）**。
<img src="https://img12.360buyimg.com/ddimg/jfs/t1/240289/13/4842/12222/65e07be1F6752cad1/c041309ad030c84d.jpg" alt="image.png" title="角色" />

## 在场景中自动生成的可玩对象
我们刚才创建了一个基础角色 **MyCharacter** 。现在，我们将尝试将其设置为关卡自动生成和管理的可玩角色。在经过这个设置之后，关卡管理器就可以在指定的开始位置生成玩家:P
1. 确保CharacterType为Player
<img src="https://img14.360buyimg.com/ddimg/jfs/t1/97635/2/43747/5973/65e07ca1Ff07c975a/51ebb973aeb8597e.jpg" alt="image.png" title="CharacterType" width=300 />
2. 将 **MyCharacter** 制作成预制件（直接拖到文件系统里就好
3. 找到基础场景（上一节所创建的）里的 *LevelManager* 脚本，将预制件赋值到 <u>PlayerPrefabs[0]</u> 和 <u>SceneCharacter</u> 中
<img src="https://img14.360buyimg.com/ddimg/jfs/t1/234637/1/13133/4694/65e07c20F2fb6c0a9/a1bc04873ba147ca.jpg" alt="image.png" title="Character组件中的PlayerPrefabs" width=300 />

这样，我们在开始游戏之后就会发现场景中生成了玩家（的预制件）！并且引擎自动为生成的玩家附加了target camera，当我们控制玩家移动的时候，相机就会跟随玩家一起运动。
### 代码引用
获取场景中的玩家（只有一个时）的方式：：
```CSHARP
LevelManager.Instance.Players[0]
```

## 在场景中创建AI
{% btn 'https://corgi-engine-docs.moremountains.com/ai.html',引擎文档,far fa-hand-point-right,blue center larger %}
{% btn 'https://www.youtube.com/watch?v=RKhmF_frdRs&list=PLopcAUHZHov3s5Eo9GOWRwny9UBjuBIDg&index=5',视频教程,far fa-hand-point-right,purple center larger %}

我们可以利用Corgi Engine在场景中创建并管理AI（例如敌人），为它们设定一定的行为逻辑。要创建一个AI，我们先按照刚才的步骤来一遍直到创建完 *Character* 组件。这时我们需要设置 Character 组件的 <u>CharacterType</u> 为 <u>AI</u> 而不是 Player ：
<img src="https://img10.360buyimg.com/ddimg/jfs/t1/170713/36/38262/4620/65e19734F74488f1d/a686fa303226d8ff.jpg" alt="image.png" title="CharacterType" />

别忘记给 <u>CharacterModel</u> 等 *Character* 组件的字段赋值。
接着，点击 *Character* 组件最下方的 <u>AutoBuildAICharacter</u> 按钮。
<img src="https://img14.360buyimg.com/ddimg/jfs/t1/96902/11/47763/12853/65e1c56eFce3e6f2c/7a04b6e7619656c0.jpg" width=300 title="AutoBuildAICharacter" />

相比于自动创建的可玩角色，AI多出了两个功能，*DamageOnTouch* 和 *AIWalk*。顾名思义，第一个脚本代表接触伤害，第二个则控制了 AI 的移动。
<img src="https://img14.360buyimg.com/ddimg/jfs/t1/163569/10/22953/3526/65e1c808Fe5a278cd/02586f25e24f2495.jpg" width=300 title="AI功能" />

我们可以设置 *AIWalk* 组件的 <u>Avoid Falling</u> 选项为true，这样敌人就可以在一个平台上来回巡逻：
![敌人自动巡逻](aiwalk.gif)

### 不会掉下平台的小怪
玩空洞骑士的时候对这个怪很有印象：
<img src="https://img12.360buyimg.com/ddimg/jfs/t1/95299/33/46605/15901/65e1cdedFfb94819c/790ab6977cff6989.jpg" width=300 title="tiktik" />
有没有办法在Corgi引擎里实现这种行走模式呢？隆重介绍：*SurfaceCrawler*组件！不过在用他的时候需要关掉很多其他组件（比如CorgiController，否则有冲突）
![敌人贴平台走](aiside.gif)

# 让角色动起来
{% btn 'https://corgi-engine-docs.moremountains.com/character-abilities.html',能力文档,far fa-hand-point-right,red center larger %}
{% btn 'https://corgi-engine-docs.moremountains.com/adding-character-level.html',引擎文档,far fa-hand-point-right,blue center larger %}
{% btn 'https://www.youtube.com/watch?v=cyXPMaHr6iU&list=PLopcAUHZHov3s5Eo9GOWRwny9UBjuBIDg&index=11',视频教程,far fa-hand-point-right,purple center larger %}

有以下三种方法能让AI/可玩角色在场景中动起来：
- 手动创建角色属性
- 自动创建（AutoBuildPlayerCharacter）
- copy一个（有什么不行的？

{% tabs 让角色动起来的方法 %}
<!-- tab 手动创建 -->
该方法要求我们通过附加组件的方式为角色添加我们所需要的功能，我们可以有所选择，不用一股脑全加上去。虽然有点麻烦，但是是最贴合实际需求的定制化方法。
1. 为你的角色 **MyCharacter** 添加 *CorgiController* 组件，在之后的章节中我们将详细介绍如何配置该组件
2. 确保你的角色 **MyCharacter** 有 *Character* 组件，在之后的章节中将详细介绍其中的配置
3. 添加 *RigidBody2D* 组件.需要注意的是该组件需设置为 <u>Kinematic</u>
4. 如果玩家有血量需求，添加 *Health* 与 *HealthBar* 组件
5. 添加你所需要的角色能力，你可以在能力文档中了解各个能力的详情
<img src="https://img11.360buyimg.com/ddimg/jfs/t1/224530/4/12004/13155/65e19136F616da7c4/ebba72b491c441fb.jpg" alt="image.png" title="一个只会移动和跳跃的角色" />
<!-- endtab -->

<!-- tab 自动创建 -->
当我们点击 *Character* 脚本上的 <u>AutoBuildPlayerCharacter</u> 按钮时，系统就自动创建了一系列玩家能力。我们可以根据实际需求删掉不需要的组件。
<img src="https://img13.360buyimg.com/ddimg/jfs/t1/159518/1/42713/33888/65e191bdFfac30e28/5d3191bbd55c674c.jpg" alt="image.png" title="自动添加的能力" />

<!-- endtab -->

<!-- tab copy一个 -->
比较调整合适的能力很复杂，直接复制配置好的预制件，然后改名、换贴图就好了
<!-- endtab -->
{% endtabs %}
## 自定义控制
在默认的配置中，玩家通过wasd进行移动，按空格进行跳跃。如果我们想用别的键位呢？又或者如果我们想用手柄控制呢？再或者我们想让用户自定义控制按钮呢？
这里，我们引入ProjectSettings里的InputManager：

<img src="https://img12.360buyimg.com/ddimg/jfs/t1/88517/14/48172/57337/65e1c2e2Fa2a0cecc/f88aeaedb712dadf.jpg" alt="image.png" title="Input Manager" />

Corgi Engine为所有的玩家功能（如移动、跳跃、奔跑等）都设置了对应的按键，直接在Manager里修改即可。并且对应的，如果我们希望添加某个需要靠按键交互的功能，也可以在InputManager里创建，之后在代码中使用。

## 真正的“动”起来
{% btn 'https://corgi-engine-docs.moremountains.com/animations.html',引擎文档,far fa-hand-point-right,blue center larger %}
{% btn 'https://www.youtube.com/watch?v=OUB73CWQSmE&list=PLopcAUHZHov3s5Eo9GOWRwny9UBjuBIDg&index=6',视频教程,far fa-hand-point-right,purple center larger %}

是的，现在我们可以控制玩家的精灵四处移动了，但是它没有动画，显得有点丑。动画需要我们自行制作，但corgi engine为我们提供了详细的动画参数，我们可以通过动画参数来控制动画机的行为。

![有动画效果的触底死亡](kill.gif)

<img src="https://img10.360buyimg.com/ddimg/jfs/t1/135656/31/41561/34213/65e1e8a1F9ca793f6/9816675268137320.jpg" alt="image.png" title="一个简单的动画机及引擎自带的动画参数（部分）" />

我们可以根据实际需求，对照着官方文档自行添加需要的参数（脚本中通过名称来查找对应参数，因此只需要添加正确命名的参数即可）即可。但我们也可以通过一键引入的方式导入参数，方法如下：
1. 为角色具有 *animator* 组件的层级添加 *CharacterAnimationParametersInitializer* 脚本
<img src="https://img11.360buyimg.com/ddimg/jfs/t1/7095/29/23662/9407/65e1e907F8f3477a8/1c40f2130dd5e2ff.jpg" alt="image.png" title="CharacterAnimationParametersInitializer" />

2. 点击 <u>AddAnimationParameters</u> 按钮，引擎将自动添加所有动画参数并删除组件
3. 现在我们可以在动画机中使用这些参数了