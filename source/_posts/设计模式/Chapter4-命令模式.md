---
tags: ['游戏编程','经验分享','命令模式','设计模式','Unity']
categories: 设计模式
title: 游戏中的命令模式
date: '2022-01-04T16:24:22.691Z'
updated: '2022-01-04T17:43:53.841Z'
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA1f1ad2ea155c7c9fedb576cd331b66ae.png
---

# 什么是命令模式？

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA1f1ad2ea155c7c9fedb576cd331b66ae.png" />

命令模式是《游戏编程模式》中介绍的第一种设计模式，也是作者“最喜欢的设计模式”。它到底有什么魅力呢？

命令模式是一种**行为型设计模式**(Behavioral Pattern)，和之前介绍的几种创建型设计模式有所不同。行为型模式是对在不同的对象之间划分责任和算法的抽象化，不仅仅关注类和对象的结构，而且重点关注它们之间的**相互作用**。

GoF对命令模式的定义如下：

> 命令模式将一个**请求**（ request） 封装成一个**对象**，从而允许开发者使用不同的请求、队列或日志将**客户端参数化**，同时支持请求操作的**撤销与恢复**。”命令就是面向对象化的**回调**。

《游戏编程模式》的作者认为这个定义不够明确，因此在书中给出了一个更简单明朗的定义：

> 命令就是一个对象化（实例化）的方法调用（这通常涉及到委托/函数指针/闭包...）。

在游戏开发中，命令模式由三个部分组成：**调用器(invoker)**，**命令(command)**和**接收者(receiver)**。

- **调用器**
  也成为“发送者”，负责对请求进行初始化，其中必须包含一个成员变量来存储对于命令对象的引用。
  发送者触发命令，而不向接收者直接发送请求。发送者并不负责创建命令对象，它通常会通过构造函数从客户端处获得预先生成的命令。

- **命令**
  抽象的命令接口，包含了execute函数。

- **具体命令**
  具体命令，实现了execute函数，代表命令的具体行为逻辑。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA0c1bce4e7593b0271a2d375ae356d85f.png" width=550 />

## 优点
- 命令模式将**请求与执行过程解耦**，客户程序不需要知道一个命令具体会有什么行为。
- 可以这样**创建宏**(一系列命令)，像管道一样执行命令流
- 可以在**不修改现有代码的情况下增加新的命令**
- 利用命令缓冲区（通常由队列维护）可以实现**撤销、回放**等操作

## 缺点
- **每增加一个新的命令，会增加一个类**，这样会增加维护成本
- 因为我们将命令与客户程序解耦了。当**发生错误或者异常的时候，如何正确处理返回返回值会变得棘手**。在多线程环境下这个问题会更麻烦。

# 应用场景：输入绑定

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA48232e4cc3814a32a0db7d8767bd362a.png" />

通常来说，一个游戏会允许玩家自定义操作按键，例如有的玩家习惯用方向键控制角色移动，有的则习惯用wasd。上图展示了空洞骑士中的默认键位配置，玩家可以自行更改其中的任意一项。对现代版本的Unity来说，我们可以用新输入系统为每个操作（也就是命令，例如向上、跳跃等）绑定对应的按键，还可以通过代码更改绑定按键的值，实现自定义配置的功能。但在新输入系统发布之前，如何允许用户自行配置输入与操作的绑定或许是一个很头疼的功能。对初学者来说，可能会倾向于直接采用硬编码的方式ban掉这个功能：

```C#
public class InputManager{
  public GameObject player;
  public void HandleInput(){
    if(Input.GetKeyDown(KeyCode.W)){
      Up(player);
    }else if(Input.GetKeyDown(KeyCode.A)){
      Left(player);
    }else if(Input.GetKeyDown(KeyCode.S)){
      Down(player);
    }else if(Input.GetKeyDown(KeyCode.D)){
      Right(player);
    }
  }
  // 具体逻辑
  private void Up(GameObject player){
    player.transform.position = player.transform.position + new Vector3(0, 1, 0);
  }
  // ...
}
```

但如果我们将上下左右的命令封装成对象呢？

```C#
// 命令接口
public interface ICommand{
  void execute(GameObject o);
}
// 具体命令
public class UpCommand : ICommand{
  void execute(GameObject o){
    // 具体逻辑
    o.transform.position = o.transform.position + new Vector3(0, 1, 0);
  }
}
// 发送者
public class InputManager{
  // 存储对于命令对象的引用
  ICommand w;
  ICommand a;
  ICommand s;
  ICommand d;
  public ICommand HandleInput(){
    if(Input.GetKeyDown(KeyCode.W)) return w;
    if(Input.GetKeyDown(KeyCode.A)) return a;
    if(Input.GetKeyDown(KeyCode.S)) return s;
    if(Input.GetKeyDown(KeyCode.D)) return d;
    // 如果什么输入也没有，返回空
    return null;
  }
}
// 接收者
public class GameCycle(){
  public GameObject player;
  ICommand command;
  void Update(){
    command = InputManager.HandleInput();
    if(command){
      command.execute(player);
    }
  }
}
```

这就是命令模式的一个实例。在用户自定义绑定的时候，更改按键（和更改前按键）对应的命令即可。发送者返回了命令给接收者（在这里是游戏循环），这是“命令即具体化”的体现。

## 如何实现撤销？
前文中提到了很多关于“撤销”和“回放”的内容，这个需求在游戏中很常见，例如不小心点错了等等，用户习惯用ctrl+z来撤销做错的这一步（或多步）。上文的代码似乎并没有实现重做功能，该怎么做呢？其实原理也很简单，在每个具体命令下实现Undo方法定义如何重做即可：

```C#
// 命令接口
public interface ICommand{
  void execute(GameObject o);
  void undo(GameObject o);
}
// 具体命令
public class UpCommand : ICommand{
  void execute(GameObject o){
    // 具体逻辑
    o.transform.position = o.transform.position + new Vector3(0, 1, 0);
  }
  void undo(GameObject o){
    o.transform.position = o.transform.position - new Vector3(0, 1, 0);
  }
}
```

然而需要注意的是，有些命令并不“基于上一步”，而是直接修改了状态，例如**传送**。这类型命令的重做就需要变量来“记住”原先的位置。总之，undo逆转了命令。

```C#
// 传送命令
public class TpCommand : ICommand{
  Vector3 prePos;
  void execute(GameObject o, int x, int y){
    // 具体逻辑
    prePos = o.transform.position;
    o.transform.position = new Vector3(x, y, 0);
  }
  void undo(GameObject o){
    o.transform.position = prePos;
  }
}
```

### 多次撤销
undo方法虽然实现了撤销的操作，但只执行了最近命令一次，如果想撤销刚才玩家执行的一系列命令该怎么办呢？我们可以维护一个**队列**和指向当前命令的指针来记录所有命令。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA88b10f5af49b19b67ada603d5cb76836.png" />

当在当前命令指针处执行了新的命令，当前指针的右侧将变成新的命令，同时回收掉原本后续的命令（相当于覆盖掉了）。有了这个队列，我们也可以轻易的实现“回放”功能，只需要从旧到新依次执行命令即可。在网络同步中，帧同步也是利用了类似的思想，逐帧地存储各个客户端发送的指令（也就是命令），因此适合战场回放一类的功能。

# 参考资料
[美] Robert Nystrom 尼斯卓姆. 游戏编程模式 (游戏设计与开发). 人民邮电出版社.
https://unity.com/cn/how-to/use-command-pattern-flexible-and-extensible-game-systems
https://design-patterns.readthedocs.io/zh-cn/latest/behavioral_patterns/command.html
https://refactoringguru.cn/design-patterns/command

才疏学浅难免有所疏漏，欢迎大家在评论区和我交流！

<img src="https://i.pinimg.com/originals/91/90/9f/91909fe563ff1329dfecd73ffeef7622.gif" />