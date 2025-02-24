---
tags: ['AI', '状态机','行为树','游戏开发']
categories: 项目复盘
title: 游戏AI-状态机与行为树
date: 2023-05-10
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA9bda383188cb8e39bd77a5e7736b7948.png
---

# 什么是游戏AI？
游戏中，能**模拟玩家行为**、**和玩家进行交互**的游戏对象就可以被称作为游戏 AI。AI 可以观察环境、做出反应，并达到相应的目的。需要实现一个 AI，我们通常需要借助游戏引擎中的**动画**、**物理**、**渲染**，以及 AI 系统中的**环境感知**、**寻路系统**和 **gameplay** 来实现。常见的 AI 有 NPC、敌人、队友等，游戏 AI 存在的目的在于提升玩家的游戏代入感、增强玩家的心流体验。

- **动画**：动画（如骨骼动画、帧动画、程序动画、morph动画、curve动画、顶点动画等）可以让 AI 看起来更真实，不那么生硬死板。
- **物理**：提供真实世界的感知和反馈，为游戏 AI 增强效果，让 AI 有更丰富的感知和交互表现。物理引擎可以用于碰撞检测/事件触发、仿真（头发模拟、流体模拟等）、AI感知、场景查询等。
- **渲染**：经过渲染引擎的渲染，游戏 AI 的形象才能最终显示在屏幕上。次世代渲染通过**应用阶段**、**几何阶段**（模型变换、视图变换、投影变换）、**光栅化阶段**实现最终效果。
- **环境感知**：AI 能感受周围环境并根据环境做出相应的决策。环境感知主要是通过**数据查询**（例如 UE 中的 EQS，即环境场景查询）来实现的。我们可以将复杂环境的数据转化为 AI 系统理解的**中间数据**，再将数据烘培到特定的物体上（例如掩体），AI 就可以通过查询知道这里有个掩体，去执行更进一步的决策。
- **寻路系统**：常用的寻路算法有 bfs、迪杰斯特拉和 A*（最广泛使用的寻路算法，也是《寻星之路》所使用的）。寻路算法的主要功能是在两点（即起点和终点）之间找到一条路径，我会在下一篇文章中介绍 A* 算法的实现和优化。
- **gameplay**：gameplay 所涉及的范围非常广泛，包括且不限于战斗、3C、AI、玩家能力等多个模块，在游戏 AI 的语境下指的主要是 AI 相关逻辑的实现。一个 gameplay 模块通常和很多系统相耦合，因此设计十分重要。

这篇文章重点介绍的是游戏 AI 的 gameplay（行为模型）部分，也就是游戏 AI 如何通过感知和记忆来执行决策的。以我的毕设《寻星之路》为例，游戏中存在敌人类型的游戏 AI ，它存在“**按照路线巡逻**”、“**警觉**”、“**追击玩家**”三种状态，它们之间的关系如下：

- AI 默认处于“按照路线巡逻”状态
- 在“按照路线巡逻”中，如果发现了玩家，就进入“警觉”状态
- 在警觉状态中，如果玩家持续存在，经过两秒后进入“追击玩家”状态
- 在“警觉”和“追击玩家”状态中，如果玩家逃脱巡查范围或死亡，则回到“按照路线巡逻”状态

对初学者来说，我们很容易会想到为每个状态设置一个bool值，用来判断对象是否处于某个状态中，例如：

```C#
bool isPatrol = true; // 初始处于巡逻状态
bool isAlert = false;
bool isChase = false;
void Update(){
  if(isPatrol){
    if(playerTarget){
      isAlert = true;
      isPartol = false;
    }else{
      // 巡逻
    }
  }
  if(isAlert){
    if(playerTarget){
      // 启用某种计时器，结束之后有：
      // isChase = true;
      // isAlert = false;
    }else{
      isProtal = true;
      isAlert = false;
    }
  }
  if(isChase){
    if(playerTarget){
      // 追击玩家
    }else{
      isProtal = true;
      isChase = false;
    }
  }
}
```

实际在开发的过程中，我们可能因为策划的需要而为敌人 AI 加入新的状态，例如追击一段时间后开始奔跑等等。为了实现这个新的状态，我们就需要加入一个新的 bool 值 isRunning 来判断敌人是否处于奔跑状态。当状态越来越多，需要维护的 bool 值也越来越多，复杂的分支和状态导致代码的可读性降低，非常容易出 bug。

那么，有什么能拯救敌人 AI 的方法吗？

# 状态机(State Machine)与状态模式
## 有限状态机(FSM)
如果把刚才的三种状态抽象成一张图：
![](Clipboard_2024-10-17-12-05-25.png)
恭喜，你成功创建了一个**有限状态机**（FSM）。FSM 是一种基于状态转换的行为模型。它将对象的行为建模为一系列状态，通过定义状态之间的转换条件和动作来控制对象的行为。状态机由多个状态组成，每个状态表示对象在特定情况下的行为。当特定条件满足时，状态之间会进行切换，从而驱动对象执行相应的动作。

> 有限状态机借鉴了计算机科学里的自动机理论（automata theory）中的一种数据结构（图灵机）思想。有限状态机可以看作是最简单的图灵机。

在状态机中，整个状态机由状态、事件和转换组成。展开来说：
- 状态机拥有一组状态，可以在这些状态之间进行切换，但同一时间只能处于一种状态中；
- 状态机会接受到一组事件（可能由输入等造成）作为转换条件；
- 每个状态之间有一组转换，每个转换都关联着一个事件并指向另一个状态。状态的转换取决于转换条件和当前所处状态。

为了避免写出复杂的 if-else 语句，我们可以考虑用枚举来表示每个状态，并用 switch-case 分支语句构建每个状态的行为和转换逻辑。在这种写法中，所有处理单个状态的代码都集中在一起了，这是实现状态机最简单的方法。在 if-else 写法中，通过 bool 值标识可能会存在一些没有意义的值，但在枚举写法中，每一个枚举值都是有意义的。

```C#
// AI 的三种状态
enum State{
  Protal,
  Alert,
  Chase
}
void Update{
  switch(state){
    case Protal:
      if(playerTarget){
        state = Alert;
      }else{
        // 巡逻
      }
    break;
    case Alert:
      if(playerTarget){
        // 启用某种计时器，结束之后有：
        state = Chase;
      }else{
        state = Protal;
      }
    break;
    case Chase:
      if(playerTarget){
        // 追击玩家
      }else{
        state = Protal;
      }
    break;
  }
}
```

## 状态模式
状态模式将这可能存在于玩家控制器中的一大串 switch-case 解耦成了更符合面向对象思维的写法。在状态模式中，我们可以定义状态接口 `IState` 作为所有状态的基类，：

```C#
interface IState{
    // 状态进入，初始化新状态
    void Enter(StateContext ctx);
    // 状态退出，清理前状态
    void Exit(StateContext ctx);
    // 状态行为
    void Action(StateContext ctx);
    // 事件处理（状态切换）
    void HandleInput(StateContext ctx);
}
```

接着，我们可以为每个状态定义一个类（以警觉为例，我们正好可以实现一下计时器方法），并实现状态接口：

```C#
class AlertState : IState {
    private float _timer;   // 计时器
    
    // 进入状态
    public void Enter(StateContext ctx) {
        _timer = 0;
        ctx.Animator.Play("Alert");
    }
    
    // 状态内对事件的处理
    public IState HandleInput(StateContext ctx) {
        if(ctx.PlayerTarget == null) 
            return AIState.Protal;  // 切换状态为巡逻
        if(_timer > 2f) 
            return AIState.Chase;   // 切换状态为追逐
        return null;
    }
    
    // 状态的业务逻辑
    public void Action(StateContext ctx) {
        _timer += Time.deltaTime;
        // 一些别的逻辑
    }
    
    // 退出状态
    public void Exit(StateContext ctx) {
        ctx.Animator.ResetTrigger("Alert");
    }
}

```

在上面的代码中，有两个概念：**上下文环境**（StateContex）和**状态实例**（AIState）。

- **上下文环境**
  通过上下文环境，状态得以感知各类事件的发生（例如捕获到玩家），从而实现状态转移。
  ```C#
class StateContext {
    public GameObject AIEntity;    // AI实体
    public Transform PlayerTarget; // 玩家目标
    public Animator Animator;      // 动画控制器
    public float DetectionRange;   // 侦测范围
}
  ```

- **状态转移**
  转移则是通过返回一个状态实例来实现的，如果不发生切换则返回空。如果状态类中不包含数据成员（比如Protal和Chase），那么这些不论怎么实例化都始终一样的对象，可以考虑作为静态只读对象存放在状态的类 AIState 中，例如：
  ```C#
// 静态实例
class AIState {
    public static readonly IState Protal = new ProtalState();
    public static readonly IState Chase = new ChaseState();
}
  ```
  对于有数据成员的状态（比如警戒状态来说，它包含了与计时器有关的变量_timer），则可以考虑通过**对象池**的方式管理状态实例，毕竟如果频繁的创建和销毁状态实例可能会增加 GC 压力：
  ```C#
// 有状态对象使用对象池
class StatePool {
    private Dictionary<Type, Queue<IState>> _pool = new();
    
    public T GetState<T>() where T : IState, new() {
        if(_pool[typeof(T)].Count > 0) 
            return (T)_pool[typeof(T)].Dequeue();
        return new T();
    }
    
    public void Release(T state) {
        _pool[typeof(T)].Enqueue(state);
    }
}
  ```

ok，现在我们已经有了各个状态，也有了状态之间如何转移的逻辑，如何应用到敌人身上呢？或者说，我们怎么知道某个敌人现在处于什么状态呢？这就需要一个状态管理器（状态机）`StateMachine` 来进行管理。

```C#
class StateMachine {
    private IState _currentState;   // 当前所处状态
    private StateContext _context;  // 上下文环境
    
    // 更新状态
    public void Update() {
        var newState = _currentState.HandleInput(_context);
        if(newState != null) {
            ChangeState(newState);
        }
        _currentState.Action(_context); // 执行状态的业务逻辑
    }
    
    // 状态切换，负责上一个状态的退出和下一个状态的进入逻辑
    private void ChangeState(IState newState) {
        _currentState?.Exit(_context);
        _currentState = newState;
        _currentState.Enter(_context);
    }
    
}
```

在类似于 AI 控制器的管理脚本中，维护 `StateMachine` 实例，就能实现简单的 AI 状态机了。

## 并发状态机
在前文的有限状态机中同一时间 AI 只能处于一个状态中，那么如果我希望 AI 在追击角色的**同时**执行持枪/开火的循环状态，应该怎么办呢？对FSM，我们可能要把状态加上“巡逻时持枪”、“巡逻时开火”、“警觉时持枪”、“警觉时开火”、“追击时持枪”和“追击时开火”了，提升了代码的复杂性。

但其实仔细想想，持枪/开火这个循环其实和其他状态是独立的，如下图所示，那么我们就可以在 AI 控制器中声明两个**状态管理器**，分别指代持枪/开火和其他的 AI 行为就可以了。这就是并发状态机的思想，适合多个毫无关联的状态机并发执行（如果有关联，并发状态机依然可以完成任务，但需要结合一些 if 语句进行判断）。
![](Clipboard_2024-10-17-15-58-43.png)

## 分层状态机
![](https://picx.zhimg.com/v2-f9122ffbc3cfd2350199893bd2b64f1f_r.jpg)
在我们实际编写状态机代码时，可能会发现状态越来越复杂（可能达到了几百上千个状态），管理越来越困难，那么除了并发状态机之外（比如我们实在没有办法拆成并发的）有没有别的优化的方式呢？这就是这一小节要介绍的**分层状态机**，又名**层次状态机**。在层次状态机中，我们可以把一系列有关联的状态组合成一个抽象的“**父状态**”，例如一个怪物的攻击行为有好几种不同的攻击方式，每种攻击方式都对应了一个状态（因为 action 执行的内容各不相同），那么就可以让这一系列状态都继承一个父状态“攻击”。

上图就展示的很直观了，状态机由数个父状态组成，每个状态中有子状态，状态机的切换是在父状态之间切换的。对于层次状态机，我们可以在 `IState` 上为其加上**子状态**指针：

```C#
interface IState {
    // 基础方法们
    void Enter(StateContext ctx);
    void Exit(StateContext ctx);
    IState HandleInput(StateContext ctx);
    void Action(StateContext ctx);
    
    // 分层
    IState CurrentSubState { get; } // 当前子状态
}
```

对一个可以嵌套子状态的父状态（例如上文中提到的攻击状态），它需要管理（持有）当前的子状态，并处理相应的事件分发逻辑。所有事件会优先交由子状态进行处理，如果子状态无法处理，再返回给父状态处理。由于状态机可能嵌套了很多层，所以可能会**冒泡地**从最内的层次一路返回到最外的层次。

```C#
abstract class AttackCompositeState : IState {
    protected IState _currentSubState;
    
    public IState CurrentSubState => _currentSubState;
    
    public override IState HandleInput(StateContext ctx) {
        // 优先由子状态处理
        var childResult = _currentSubState?.HandleInput(ctx);
        if (childResult != null) return childResult;
        
        // 子状态未处理时检查父状态迁移
        if (...) {
            return AIState.nextState;
        }

        // 不转移
        return null;
    }
}
```

进入某个父状态之后状态机就会进入父状态内部的状态机循环，直到达到切换条件。UE5 的动画蓝图就是基于层次状态机实现的。

即使状态模式已经在一定程度上解耦了状态机的复杂代码，但在实际项目中，我们随时可能增加新状态、减少状态或者改变状态之间的迁移关系，如果状态越来越多，任何一点小修改都会产生很大的工作量（例如我们需要维护新的状态类），甚至可能破坏原有的逻辑。不过状态机相比于我们接下来要介绍的行为树，优点在于**执行效率更高**、**不需要判断冗余的条件**，因此适合**状态明确且有限的系统**。

# 行为树(Behavior Tree)
![可视化行为树编辑器](https://user-images.githubusercontent.com/8220843/70063424-b20b3700-1622-11ea-8c58-102322c2a88b.png)

**行为树**（Behavior Tree）是一种基于**树结构**的行为模型。它以树状的方式描述了对象的行为决策过程。行为树由多个节点组成，每个节点代表一个具体的行为或决策。节点之间通过连接关系形成树状结构，决定了对象在特定情况下应该执行哪些行为（不同层从上往下，同一层从左往右）。通过遍历行为树，对象可以根据节点的逻辑规则来选择合适的行为进行执行。行为树则适合用于描述对象的**复杂行为决策过程**，例如角色在特定情况下应该采取何种行为（如追逐、逃避、攻击等）。

作为一种树，行为树由三种节点构成，每个节点都有<font color=green>**成功**</font>、<font color=red>**失败**</font>和<font color=orange>**运行中**</font>三种返回值。

- **行为节点**（Task）
行为树的叶子节点，代表了 AI 的具体行为（例如移动、攻击、什么也不做），包含初始化函数 Init 和行为逻辑函数 OnTick（每帧执行）。

- **控制节点**（Composite）
控制节点一般为行为树的中间节点，控制行为树的行为变换。具体来说有以下三种类型：
  - **顺序节点**（Sequence）
  相当于**与**（and），依次执行所有子节点，若当前子节点返回成功，则继续执行下一个子节点；若子当前节点返回失败，则中断后续子节点的执行，并把结果返回给父节点。
  <img src="https://lifan.tech/images/behavior/behavior-sequences.png" width=200>
  - **选择节点**（Selector）
  相当于**或**（or），依次执行所有子节点，若当前子节点返回成功，则中断后续节点运行，并把结果返回给父节点。如果返回失败，则执行下一个节点。
  <img src="https://lifan.tech/images/behavior/behavior-selector.png" width=200>
  - **平行节点**（Parallel）
  依次执行所有子节点，无论失败与否，都会把所有子节点执行一遍。至于Parallel节点该返回什么值给父节点，这要看需求。比如：成功数 > 失败数返回成功，否则返回失败。
  <img src="https://lifan.tech/images/behavior/behavior-parellel.png" width=200>
  - **随机节点**（Random）
  随机选择一个子节点来运行。

- **装饰节点**
  - **逆变节点**（Inverter）
  对子节点的返回值取反，相当于**非**（not），它只会有一个子节点。
  - **成功节点**（Succeeder）
  不管其子节点返回何值，都会返回Success给父节点
  - **重复节点**（Repeater）
  重复执行n次子节点。
  - **重复直至失败节点**（Repeat Until Fail）
  重复执行子节点，直到失败为上；同样也有类似的重复直至成功节点这里就不列出了。
  - **执行一段时间**（MaxTime）
  重复执行子节点一段时间。

行为树的出现从根本上解决了前文提到的复杂代码维护的问题，它把每个行为作为一个原子项，任何人都可以决定 AI 的执行流程，作为程序的我们只需要集中精力根据需求增加新的行为，而不用关心具体流程、转换方式是什么样的。

不过在性能上，行为树的运算通过帧循环的 update 来驱动，每帧会遍历所有非行为节点，在树比较复杂时会造成资源浪费。在我用的 TheKiwiCoder 行为树插件中，行为树可以通过一个名叫 **Blackboard**（黑板）的类来存放**共享数据**（例如储存寻路算法的结果），这样可以避免每走一步都执行一遍寻路，减少开销。

针对行为树的性能优化，还可以考虑**行为树+状态机**的实现方式：
- **顶层状态机 + 细节行为树**
如果用状态机实现简单状态的切换，进入某个状态（例如战斗）后再用行为树表示复杂行为逻辑，这会是一个有效的优化。尤其是怪物很多时，大部分时间段，大部分怪都处于巡逻状态，完全没有必要遍历行为树。
- **顶层行为树 + 细节状态机**
也可以用行为树作为上层来管理宏观策略，状态机作为底层处理具体的原子状态（例如移动、攻击等等）。

现代游戏 AI 一般都是采用行为树来实现的了，在 UE 中更是包含了行为树的功能模块以供开发者使用。

## 寻星之路中的行为树
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAf5c5776b4f62ae3a2a05b40a2f0c44eb.png" width=400 />

还记得我们在文章最开始提到的敌人 AI 逻辑吗？那其实就是《寻星之路》中的敌人 AI 逻辑。如果把这个逻辑转变成行为树，就会是上图所示的这样。该行为树由三类节点构成：

- **行为节点**：巡逻（Move）、警觉（Alumb）、回到巡逻路径（MoveBack）、追击玩家（Chase）
- **控制节点**：顺序节点、选择节点
- **装饰节点**：重复

现在我们沿着根节点开始捋一遍。
1. 从根开始，进入重复节点，意味着接下来将重复执行重复节点下的所有树结构
2. 进入选择器。若选择器中当前子节点返回成功，则中断后续节点运行，并把结果返回给父节点
3. 进入选择器的第一个节点顺序节点。在顺序节点中，若子当前节点返回失败，则中断后续子节点的执行，并把结果返回给父节点
4. 进入警觉（Alumb）节点，
  - 如果玩家未使用魔法，则该节点为Fail状态，顺序节点返回失败，顺序节点返回失败，选择器进入下一个节点
  - 如果玩家使用魔法，则节点为Running，执行逻辑
    - 如果到达警觉节点后视野内无玩家，则节点变为Success，顺序执行下一个节点MoveBack回到巡逻路径
    - 如果到达警觉节点后视野内有玩家，行为树就会立刻将当前节点的状态设为Fail，随后行为树根据选择器的逻辑进入下一个选择器的第一个节点追击（Chase）
5. 在Move节点敌人将沿路径巡逻，当发现玩家时节点变为Fail，随后行为树根据选择器的逻辑进入下一个选择器的第一个节点追击（Chase）
6. 当玩家脱离追击视野，追击节点变为Success，执行顺序节点的下一个节点MoveBack回到巡逻路径，完事后返回Success
7. 选择器返回Success，开始重复

# 更多可能的游戏AI构建方式
时隔将近两年，今天（2025/1/4）听完了腾讯游戏客户端的公开课之后，我来更新一下这篇文章。随着现在的 **LLM** 和**智能体**（AI agent）的兴起，越来越多项目在尝试将**强化学习**与**深度学习**相结合，并引入到游戏 AI 中。也有一些项目直接将 NPC 的对话接入了训练好的 ChatGPT 中，以求回复的真实感。

目前来说，不管是将训练好的模型放在客户端本地，还是通过 API 调用服务器上的大语言模型资源，其开销和实时性都是不如直接将更多文本配置在本地的。游戏 AI 必定会随着 LLM 的进一步发展得到进化，这是一条未尽之路，值得我们所有人关注。

# 参考资料
游戏编程模式（游戏设计与开发）
https://zhuanlan.zhihu.com/p/540191047
https://lifan.tech/2020/02/15/game/behavior-tree/
游戏动画：https://zhuanlan.zhihu.com/p/462945444
游戏物理：https://zhuanlan.zhihu.com/p/660201778