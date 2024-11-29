---
tags: ['设计模式','游戏开发','ECS','框架']
title: ECS设计模式
categories: 设计模式
date: 2024-08-09
update: 2024-08-15
cover: https://docs.unity3d.com/Packages/com.unity.entities@0.3/manual/images/WhatIsECSinfographic0000.png
---

![](https://miro.medium.com/v2/resize:fit:795/1*u4_kgaIkyGWY5r0ijLazeQ.png)

# 什么是ECS？为什么我们需要它？

在传统游戏开发中，开发者习惯于将每个功能写成一个class，例如玩家控制器（PlayerController）。在这个脚本中不但包含了玩家自身的**数据**（data，例如速度、血量等），还包含了针对这些数据的处理**逻辑**（behavior）。

有时作为开发者，我们需要在极短时间完成原型发开来验证玩法的可行性（或者我们只是打了个game jam），一些能够被重复利用的“轮子”是相当重要的。我们前面所提到的每个功能一个 class 的做法其实很不适合拓展，因为数据和行为是耦合的，不便于修改。

有没有更好的设计模式呢？接下来让我们隆重推出本文的主角——**ECS设计模式**！本文将参照轻量级 ECS 解决方案 **Entitas** 对该设计模式从源码层面进行介绍。首先，什么是*ECS* ？ECS 即 **Entity-Component-System**，指“**实体-组件-系统**”设计模式，是一种有别于传统 **OOP**（面向对象）类型的设计模式，它是 **DOP**（面向数据）的。

- **E**ntity/实体
实体类似于 GameObject，可以简单理解为“游戏对象”。
- **C**omponent/组件
组件包含了实体的数据（Data）。组件有且仅有数据，不包含任何逻辑，如：位置组件（记录位置信息）、运动组件（记录速度/加速度等）等等。
- **S**ystem/系统
系统包含了游戏逻辑，比如运动系统负责计算速度变化等等。

同时，为了便于管理，Entitas中有以下概念：
- Context/上下文环境
上下文环境包含了所有实体，开发者可以往其中添加/移除实体。同时，Context 还存储了一些全局状态（例如关卡进度、唯一玩家等等）。
- Group/组
组可以视为 context 的子集，如果说 context 是一个数据库，group 就是通过 SELECT 从 context 中筛选出的特定类型的实体。我们可以把一类对象归为一个组（例如玩家组、子弹组），这有助于提高查询速度。

ECS 把数据和逻辑解耦，各个实体上的组件中只保留了数据。逻辑运算由一个系统负责，在特定的时间点对每个它旗下的所有组件执行，那么理所当然的，每个实体上的组件里的数据不同，逻辑运算后的结果也就不同了。
```
+-----------------+                                            +------------+
|     Context     |                                            | ComponentA |
|-----------------|                                            |------------|
|    e       e    |      +------------+                        |    Data    |
|       e      e--|----> |   Entity   |                        +------------+          
|  e        e     |      |------------|                               ^
|     e  e     e  |      | ComponentA |                               |
| e          e    |      |            |      +------------+      +---------+       +------------+
|    e     e      |      | ComponentB-|----> | ComponentA |<---- | ASystem | ----> | ComponentA |
|  e    e    e    |      |            |      |------------|      +---------+       |------------|
|    e    e     e |      | ComponentC |      |    Data    |   calculate behavior   |    Data    |
+-----------------+      +------------+      +------------+                        +------------+

```

区分后，代码的可读性和可维护性上得到了提升，这样一来我们可以为每个系统编写各类行为（函数）的轮子，提升开发效率了。当然，ECS的优点并不只有这么些，它真正的优点是：当面对**大量游戏对象**（例如1000个）时，采用ECS架构比传统的架构开销更小。为什么？

## 针对大量游戏对象的优化

![](https://cdn.mos.cms.futurecdn.net/mz2NU5EYiqXGEKzm9ejTF.jpg)

在RTS等类型的一些游戏中，场景内有时会同时存在大量的游戏对象。如果我们为每个游戏对象的prefab都添加了脚本Enemy，假设我们需要在脚本的 Update 函数中更新游戏对象的信息，会发生什么事呢？答案是：游戏的**帧率可能出现严重下滑**，特别是对于处理器性能较差的低端机型来说。

Unity 生命周期函数是基于类反射实现的，并且这一过程会在运行时不停对所有MonoBehaviour遍历进行。反射在这种情况下允许 Unity 在运行时动态调用对象的特定方法（如 Start、Update 等），而不需要编译时确定。这种方法灵活，但性能上可能稍有影响，因为反射操作通常比直接调用**慢**。这导致当场景中存在 1000 个游戏对象时，执行 1000 次 Update 将严重拖垮游戏帧率。

而在 ECS 中，我们只需要通过执行一次旋转**系统**中的 Update 函数对所有存在旋转**组件**的**实体**进行更新就可以了，不需要执行 1000 次 Update。ECS 主要的开销集中在查询和提取包含特定组件的实体上，我们可以用 pool（Entitas 中的 group） 的思想来优化这一过程。

```
+-----------------+
|     Context     |
|-----------------|
|    e       e    |
|       e      e  |
|  e        e     |
|     e  e     e  |
| e          e    |
|    e     e      |
|  e    e    e    |
|    e    e     e |
+-----------------+
  |
  |     +-------------+  Groups:
  |     |      e      |  Subsets of entities in the context
  |     |   e     e   |  for blazing fast querying
  +---> |        +------------+
        |     e  |    |       |
        |  e     | e  |  e    |
        +--------|----+    e  |
                 |     e      |
                 |  e     e   |
                 +------------+
```

> **生命周期函数居然是基于反射实现的？**
Uniy 引擎整体是架设在 mono IDE 基础上的，支持通过 string 来查找方法，且和真正的反射一样是能够查找调用**私有方法**，当找到这些方法后会存下函数指针待之后使用。在 Unity 内部存在一张表，这张表的形成是通过在场景中查找全部 MonoBehaviour 类型脚本然后遍历里面的方法，将需要的调用的方法全部记录下来然后再进行调用，防止那些不需要调用的 MonoBehaviour 中的方法占用空间，节省资源开耗。

> **为什么不采用多态？**
反射实际上是开销非常大的调用方式，比虚方法来说要高得多。Unity 使用这种事件机制的根本原因是出于对灵活性的考虑，因为并非所有的 MonoBehaviour 都需要 Update（或 Start，Awake 等等）。Unity 会维护一个需要 Update 的 Behaviour 列表，藉此避免进行空的虚函数调用，提高性能。Unity 采用组件式设计，触发一个事件，需要通知到相应 gameobject 的所有组件。如果使用多态来实现，则必须假设所有组件都派生自包含此事件的基类，或者筛选出派生自此基类的组件逐一通知。这样一来是麻烦，二来则容易带来复杂的继承关系，与组件式这种倡导用聚合代替继承的设计从理念上就是相悖的。

## 在ECS中实现事件系统
为了严格贯彻 ECS 思想，我们需要分离数据和行为，因此我们无法采取传统的组件思想。

### 如果我们直接遍历所有实体
如何在 ECS 中触发事件呢？例如，当一个游戏对象的 y 值超出游戏可视边界（屏幕）的 y 值时，我们希望该对象被销毁，此时应该怎么办？或许我们可以直接在 system 中遍历某类 group 下的所有实体并进行检查。
```
void Update(){
  Execute();  // 执行逻辑
}

void Execute(){
  var edgePos = pool.edgeEntity.position;   // 获取边缘位置
  foreach(var entity in positionGroup.GetEntities()){ // 对所有具有位置组件的实体
    if(entity.position.y >= edgePos.y){  //超出边界
      entity.isDestory = true;  // 销毁
    }
  }
}
```
假设存在一个针对子弹的系统 BullteSystem，在执行过程中会不断遍历所有存活的子弹 group，那么我们可以在 update 中对每个子弹的位置和每个敌人的位置进行判断，如果子弹的位置与敌人重合（举个例子）则认为子弹击中敌人。这样想想，如果不采取四叉树/八叉树缩小检索范围，开销还是很大的。

### 如果我们专门为事件拟定一个系统
假设我们不愿意逐帧判断，而是仅在某些变化发生的时候对事件进行响应呢？这便是 ReactiveSystem（响应系统），它会对 context 的变化（例如全局状态上的改变）做出反应，而不是类似于一般系统那样在 Update 中逐帧执行。

举个例子，当玩家到达出口的位置时，当前关卡 + 1的实现逻辑如下：
```C#
// 触发器，用于标记玩家和玩家的位置组件
TriggerOnEvent IReactiveSystem.trigger{
  get{
    return Matcher.AllOf(Matcher.Player, Matcher.Position).OnEntityAdded();
  }
}
// 当玩家位置改变时，遍历所有 exitGroup 中出口的位置进行检查
void IReactiveExecuteSystem.Excute(List<Entity> entities){
  var playerPos = context.playerEntity.position;
  foreach(var exit in exitGroup.GetEntities()){
    if(playerPos.Equals(exit.position)){
      int curLevel = context.level.level;
      context.ReplaceLevel(curLevel + 1);
      return;
    }
  }
}
```

# Entitas源码学习
## 核心模块
在Entitas源码的核心组成中，我们可以找到以下八个模块：
1. **Collector**
  Collector可以根据特定的组事件（groupEvent）观察并收集同一个上下文环境中一个或多个组的变化实体，用于实现ECS的事件系统。
  上文中提到的TriggerOnEvent就归属于Collector模块。
2. **Context**
  上下文环境，在一些关于Entitas的早期文档中被称为池子（pool），管理实体和组的生命周期。
  可以创建和销毁实体，并获取实体组。
  创建上下文的首选方法是使用代码生成器（code generator，我会在下文介绍）生成的方法，例如 var context = new GameContext()；
3. **Entity**
  实体，即游戏对象。程序员可以通过 context.CreateEntity / entity.Destory 的方式创建 / 销毁实体。用户可以增加/替换/移除实体上的组件。
4. **EntityIndex**
  为实体创建一个基于键值对的索引，目的是加速实体查询。它将实体与某个键（如组件中的某个值）关联起来，允许通过键快速找到相关实体集。EntityIndex 支持单个键或多个键的映射，并提供增删实体的操作。这对于大规模实体管理场景下的高效查询非常有用，特别是在需要频繁根据某个属性查找实体时。
    - 为实体建立索引。
    - 提供高效的查找机制。
    - 实体的增删操作自动更新索引。
5. **Extentions**
  工具类，包含移除字符串后缀（用于异常）和单一实体收集器。
6. **Group**
   使用 context.GetGroup(matcher) 获取与指定匹配器匹配的实体组。使用相同的匹配器调用 context.GetGroup(matcher) 将始终返回相同的组实例。
   创建的组由上下文管理，并且始终是最新的。
   它会自动添加与匹配器匹配的实体，或者在实体不再与匹配器匹配时立即将其删除。
7. **Matcher**
  匹配器，用于匹配实体组件的工具，它的作用是通过**组件索引**匹配实体，来判断某个实体是否具备特定条件。它的主要功能包括：
    - AllOf: 匹配具有指定所有组件的实体。
    - AnyOf: 匹配具有指定任意组件的实体。
    - NoneOf: 匹配不具备指定组件的实体。
8. **Systems**
  系统模块为系统分组提供了一种便捷的方式。
  开发者可以添加 IInitializeSystem、IExecuteSystem、ICleanupSystem、ITearDownSystem、ReactiveSystem 和其他嵌套系统实例，所有系统都将按照开发者添加的顺序进行初始化和执行。

### Context
> A context manages the lifecycle of entities and groups.
You can create and destroy entities and get groups of entities.
The preferred way to create a context is to use the generated methods from the code generator, e.g. var context = new GameContext();
上下文管理实体和组的生命周期。
您可以创建和销毁实体，并获取实体组。
创建上下文的首选方法是使用代码生成器生成的方法，例如 var context = new GameContext()；

#### **Context.cs**
  - 包含了两类（实体与组）的 4 个事件
    - 实体创建
    - 实体将被销毁
    - 实体销毁
    - 组创建
  - 规定了实体的相关设置
    - 实体所能搭载的最大组件数量
    - 管理组件池（这是针对每一种类型的组件而言的 stack 结构的池，用于组件的复用）
  - 管理实体与组
    - 维护 context 中的实体、可复用实体（位于对象池中）、被其他对象（如 group、collector、ReactiveSystem）保留的实体数量
    - 创建实体
      1. 检查是否存在可复用的实体，如果不存在再创建新实体并根据相关设置进行初始化
      2. 加入 context 维护的实体列表，保留实体。保留/释放是 AERC（自动实体参照计数）的一部分，内部用于防止将保留的实体集中起来。如果手动使用保留，也必须在某些时候手动释放。
      3. 清空缓存（entitiesCache）。这是一个泛型实体数组 TEntity[]，用于居埃苏返回当前 context 中的所有实体。由于数组的特性，每次创建新的实体时需要清空缓存，并在需要的时候重新创建一次。
      4. 增加实体相关事件的委托（组件添加/组件移除/组件替换/释放实体/实体销毁）
      5. 触发实体创建事件，返回实体对象
    - 销毁所有实体
      1. 获取 context 中的所有实体（TEntity[]）
      2. 逐个 Destroy，最后清空维护的实体表
      3. 如果存在被保留的实体，则返回报错
    - 检查上下文是否包含某个特定实体
    - 获取所有实体（如果不存在缓存，则将实体表转成 TEntity[] 返回）
    - 获取所有符合匹配器条件的实体（其实就是返回一个组里的实体）
    - 获取所有符合匹配器条件的组（采用相同的匹配器返回的组是同一个）
    - 添加 EntityIndex 名称
    - 获取 EntityIndex 名称
    - 重置实体 id 编号（creationIndex）
    - 清空特定id的组件的组件池
    - 清空所有组件池
  - 上下文手动创建方法
    Entitas的作者并不建议我们通过手动方式进行创建，而是建议我们使用代码生成器。如果采用手动方式，我们需要人为
  - 重置上下文
    - 摧毁所有实体
    - 重置实体 id 编号（creationIndex）
  - 移除所有事件监听

#### **ContextInfo.cs**
  - 包含了上下文的信息，主要用途是提供更好的报错信息。
    1. 上下文名称 Name
    2. 组件名称数组 ComponentNames
    3. 组件类型数组 ComponentTypes

### Entity
> Use context.CreateEntity() to create a new entity and entity.Destroy() to destroy it.
You can add, replace and remove IComponent to an entity.
使用 context.CreateEntity() 创建新实体，使用 entity.Destroy() 销毁实体。
您可以在实体中添加、替换和删除 IComponent。

#### Entity.cs
- 包含了 5 个事件（组建添加/组件移除/组件替换/实体释放/实体摧毁）
- 实体属性
  - 实体所能搭载的最大组件数量
  - id（是唯一的，在context中创建实体时被设置）
  - isEnable（表示实体的生存状态）
  - RetainCount / 引用计数（保留此实体的对象数量）
- 组件方法
  - 根据组件 id 添加组件
  - 根据组件 id 移除组件
  - 根据组件 id 替换组件
  - 根据组件 id 获取组件
  - 获取实体挂载的所有组件
  - 获取实体挂载的所有组件的 id 组成的数组
  - 判断是否挂载了对应组件 id 的组件
  - 判断是否挂载了所有存在于对应一系列（int[]）组件 id 的组件
  - 判断是否挂载了任意一个存在于对应一系列（int[]）组件 id 的组件
  - 判断是否挂载了组件（挂在组件是否为空）
  - 移除所有组件
  - 根据组件 id 获取对应的组件池中的组件（如果不存在就创建一个）
- AERC管理方法
  - 保留该实体。一个所有者（owner）只能保留一次同一实体
  - 释放该实体
  - 摧毁实体

#### EntityEqualityComparer.cs
实现了 IEqualityComparer\<TEntity\> 接口，用于
  - 比较两个 Entity 对象的相等性。它重写了 Equals 方法来判断两个实体对象是否相等（直接比较引用）
  - 重写了 GetHashCode 方法返回实体的 ID 作为哈希码

#### AERC（Automatic Entity Reference Counting）
实体模块里最有趣的一个功能是AERC（自动实体引用计数），是一个在内部执行的有助于调试和优化性能的功能。
在Entitas框架中，自动实体引用计数（AERC）的作用是跟踪实体的**引用数量**，以防止实体在被使用时被错误回收、在计数为0时将实体返回到对象池中以释放资源。具体而言，引用计数通过 Retain 和 Release 方法来管理：
1. **Retain**：增加实体的引用计数，表示该实体被某个系统或对象“保留”。
2. **Release**：减少引用计数，表示系统或对象不再使用该实体。 当引用计数为零时，实体才会被回收到对象池中，从而避免使用中的实体被错误回收。

AERC 在实现思路上参考了 CG 的引用计数机制，分为 SafeAERC 和 UnsafeAERC 两种实现方式。SafeAERC 额外记录了谁保留了实体，提供更安全的管理，而 UnsafeAERC 只维护计数，不追踪具体的所有者。

- **SafeAERC**
SafeAERC 会检查实体是否已被保留或释放。它比 UnsafeAERC 慢，但可以保留所有者的信息。
每个实体在被保留（Retain）或释放（Release）时，SafeAERC 会记录和跟踪拥有它的对象。SafeAERC 通过一个哈希集合_owners 存储了所有保留实体的对象；Retain 方法会检查实体是否已经被当前对象保留，防止重复保留，而 Release 则确保只能由已保留该实体的对象进行释放。
- **UnsafeAERC**
UnsafeAERC 不会检查实体是否已被保留或释放。它比 SafeAERC 快，但会丢失有关所有者的信息。
UnsafeAERC 只管理引用计数，Retain 和 Release 操作会简单增加或减少计数，不检查是否已被保留或释放，也不记录哪个对象在保留实体。因此，它更快，但无法追踪所有者。

### System
> Systems provide a convenient way to group systems.
You can add IInitializeSystem, IExecuteSystem, ICleanupSystem, ITearDownSystem, ReactiveSystem and other nested Systems instances.
All systems will be initialized and executed based on the order you added them.
系统为系统分组提供了一种便捷的方式。
您可以添加 IInitializeSystem、IExecuteSystem、ICleanupSystem、ITearDownSystem、ReactiveSystem 和其他嵌套系统实例。
所有系统都将按照您添加的顺序进行初始化和执行。

#### System.cs
System.cs 方便管理和执行四种类型的系统，分别是：初始化/执行/清理和终止。
- 初始化系统
顾名思义，包含一个Initialize方法，可以实现各类“初始化”功能，例如“在上下文中创建一个玩家实体，并为其初始化生命值和位置”
- 执行系统
- 清理系统
- 终止系统

代码中的每个函数在执行时，会遍历（因此是有序的）特定类型的系统列表，调用相应的初始化、执行、清理或终止操作。
- 添加/移除系统到对应列表
- 按顺序执行所有特定类型系统的对应方法
```
public virtual void Initialize()
{
    for (var i = 0; i < _initializeSystems.Count; i++)
    _initializeSystems[i].Initialize();
}
```
- 激活所有响应系统（ReactiveSystem）
- 停用所有响应系统
- 仅清除所有响应系统

#### ReactiveSystem.cs
如果根据指定的收集器发生了变化，响应系统就会调用 Execute(entities)，并且只会传入发生变化的实体。
常见的用例是对变化做出反应，例如，实体位置发生变化时，会更新相关游戏对象的 gameObject.transform.position。

#### ParallelSystem.cs
并行系统使用实体子集调用 Execute(entities)，并将工作量分配给多个线程。
在 Entitas 中编写多线程代码时，不要使用 AddXyz() 和 ReplaceXyz() 等生成的方法。

### Group
> Use context.GetGroup(matcher) to get a group of entities which match the specified matcher. Calling context.GetGroup(matcher) with the same matcher will always return the same instance of the group.
The created group is managed by the context and will always be up to date.
It will automatically add entities that match the matcher or remove entities as soon as they don't match the matcher anymore.
使用 context.GetGroup(matcher) 可以获取与指定匹配器匹配的实体组。使用相同的匹配器调用 context.GetGroup(matcher) 将始终返回相同的组实例。
创建的组由上下文管理，并且始终是最新的。
它将自动添加与匹配器匹配的实体，或在实体不再与匹配器匹配时立即将其删除。

Group 用于管理和跟踪符合特定 Matcher 的实体集合，实现动态管理实体的添加、移除和更新，并通过事件系统触发相关的回调。
主要功能和函数：
1. 事件系统
OnEntityAdded、OnEntityRemoved 和 OnEntityUpdated 事件分别在实体被添加、移除或更新时触发，通知订阅者发生的变化。
2. 构造函数
Group(IMatcher<TEntity> matcher)：初始化时使用 matcher 匹配实体。
HandleEntity/UpdateEntity
3. 处理实体的添加、移除或更新逻辑，动态管理实体是否应该存在于组中。
AddEntity/RemoveEntity
4. 添加或移除符合条件的实体，并触发相应事件。
GetEntities() / GetSingleEntity()：

返回当前组内所有实体，或返回唯一的实体（如果组内只有一个实体）。

### Matcher
#### Matcher.cs
Matcher<TEntity> 类主要用于匹配实体的**组件**。它允许定义三个类型的组件条件：AllOf、AnyOf 和 NoneOf。

属性：
  - Indexes: 返回合并后的索引数组，使用 ??= 操作符进行初始化。
  - AllOfIndexes, AnyOfIndexes, NoneOfIndexes: 返回各自的组件索引。
- 构造函数
- 匹配方法
  - AnyOf(params int[] indexes): 设置满足条件的任何组件的索引。
  - AnyOf(params IMatcher<TEntity>[] matchers): 允许使用其他匹配器的索引。
  - NoneOf(params int[] indexes): 设置不满足条件的组件的索引。
  - NoneOf(params IMatcher<TEntity>[] matchers): 允许使用其他匹配器的索引。
  - Matches(TEntity entity): 检查一个实体是否满足 AllOf、AnyOf 和 NoneOf 的条件。
总结
这个类通过提供灵活的组件匹配功能，支持复杂的实体查询和操作。

### Collector
> A Collector can observe one or more groups from the same context and collects changed entities based on the specified groupEvent.
收集器可观察同一上下文中的一个或多个组，并根据指定的 groupEvent 收集已更改的实体。

作用是收集符合特定条件的**组**，并通过监听组的**变化事件**来进行实体的收集。每个 Collector 会关联一个或多个组和事件类型（如实体添加、移除或添加/移除），从而**在这些事件发生时将实体收集到集合中**。

主要函数解释：
- 构造函数：创建一个 Collector，并根据传入的 Group 和 GroupEvent 来决定如何收集实体。它初始化了内部的集合和事件处理委托 _onEntityDelegate。
- Activate()：激活 Collector，开始监听相关的 GroupEvent，并收集符合条件的实体。
- Deactivate()：停止监听事件，并清空已收集的实体集合。
- ClearCollectedEntities()：释放所有已收集的实体，并清空集合。

# 总结

总的来说，在游戏开发中的 ECS 设计模式有以下优点：
1. 性能优化：ECS将数据和逻辑分离，通过缓存友好的内存布局提高CPU缓存利用率，从而优化性能。
2. 模块化和可扩展性：逻辑通过系统处理，数据通过组件管理，方便不同功能模块独立开发和调试。
3. 数据驱动：实体仅是ID，组件存储数据，系统执行逻辑，可以轻松增加或移除组件来改变实体行为，灵活处理游戏状态。
4. 并行处理：系统可独立处理各类组件，利于多线程优化（这指的是通过 Unity ECS 的 JobSystem 来进行多线程并发处理）。

# 参考资料
https://github.com/sschmid/Entitas/wiki/Unity-Tutorial-Hello-World
https://docs.unity3d.com/Packages/com.unity.entities@0.3/manual/index.html
https://github.com/sschmid/Entitas
https://www.youtube.com/watch?v=jQEXETwgPDs
https://www.cnblogs.com/morning-lee/p/7509018.html
https://medium.com/%E9%81%8A%E6%88%B2%E9%96%8B%E7%99%BC%E9%9A%A8%E7%AD%86/unity-ecs-%E8%9C%BB%E8%9C%93%E9%BB%9E%E6%B0%B4-e259ccf02d09
https://paakmau.github.io/202002110101/
https://yimicgh.top/%E7%BF%BB%E8%AF%91/ECS-01-Core/
