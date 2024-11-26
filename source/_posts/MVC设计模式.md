---
tags: [游戏开发/通用]
title: MVC设计模式
date: 2021-05-19
update: 2021-05-25
cover: https://assets.toptal.io/images?url=https%3A%2F%2Fbs-uploads.toptal.io%2Fblackfish-uploads%2Fcomponents%2Fblog_post_page%2F4084336%2Fcover_image%2Fretina_500x200%2Fcover-unity-with-mvc-how-to-level-up-your-game-development-46aaec316442778385f4c4149a0f5273.png
---

MVC 模式的全称是 Model-View-Controller，即模型-视图-控制器。严格来说，MVC并不是GoF（四人帮）归纳出的一种独立设计模式，而是组合模式（UI界面的构建）、策略模式（VC之间的一对多关系）和观察者模式（MV间的事件触发）的组合。
这种设计模型通常用于用户图形界面的图形/数据/逻辑分离（是不是感觉在某些方面很像 ECS？），因此在游戏领域中常被用于 UI 相关的功能上。那么，什么是 M / V / C 呢？
- **M**（模型）
模型层（M 层）负责**存放数据**，类似于 ECS 中的 C（组件）。
- **V**（视图）
视图层（V 层）负责**展示用户图形界面**，类似于 ECS 中的 E（实体），是用户与之直接交互的元素。
- **C**（控制器）
控制层（C 层）负责**处理数据逻辑**，类似于 ECS 中的 S（系统），包含了所有数据计算的部分，负责更新M层的数据。

使用 MVC 模式的目的是将 M 和 V 的实现代码分离，从而使同一个程序可以使用不同的表现形式。比如一批统计数据可以分别用柱状图、饼图来表示。C 存在的目的则是确保 M 和 V 的同步，一旦 M 改变，V 应该同步更新。

# Unity中的MVC
![](https://img-blog.csdnimg.cn/20181106152450894.png)
![](http://xpsuper.com/upload/2021/10/MVC01-da44a1398db94f8987ec9aa716cb1725.png)

以上述框架为例，假设我们要做一个点击按钮更新金币数量（即点击一次金币数 +1 ）和清空当前金币的功能：

![](https://velog.velcdn.com/images/suhan0304/post/d6333381-8221-4cae-9f3c-f7d21ed3e978/image.png)

## M
M 层存放所有数据，包括静态数据、变量、资源（如贴图、游戏对象等）等。在这个例子中，M 层中只有金币数量的数据。
```C#
class CoinModel : Monobehavior{
  // 金币数据
  public int _coins{get; set;}
}
```

## V
V 层负责为拿取所有 UI 组件对象，并为它们声明相关的事件：
```C#
class CoinView : Monobehavior{
  CoinController coinCtrl = GameObject.Find("#VisualElement").GetComponent("CoinController");
  // 拿取所有组件obj
  public Text coin_num;
  public Button b_add;
  public Button b_reset;
  public Awake(){
    coin_num = GameObject.Find("#Label--Coin");
    b_add = GameObject.Find("#Button--Coin");
    b_reset = GameObject.Find("#Button--Reset");
  }
  // 声明点击事件
  public void Update(){
    if(Intput.GetMouseButtonDown(0)){
      // 触发硬币+1
      if(b_add.OnePointColliderObject() != null){
        coinCtrl.AddCoin();
      }
      // 触发硬币重置
      if(b_reset.OnePointColliderObject() != null){
        coinCtrl.ResetCoin();
      }
    }
  }
}
```

## C
C 层负责业务逻辑相关功能（在本例中为“金币 +1 ”和“清空金币”两大功能）的实现，获取Model的数据，通知View层更新数据。
```C#
public class CoinController{
  // 获取M层和V层 
  CoinModel coinMod = GameObject.Find("#VisualElement").GetComponent("CoinModel");
  CoinView coinVie = GameObject.Find("#VisualElement").GetComponent("CoinView");
  // 硬币++
  public void AddCoin(){
    coinMod._coins++;
    coinVie.coin_num.text = coinMod._coins;
  }
  // 重置硬币
  public void ResetCoin(){
    coinMod._coins = 0;
    coinVie.coin_num.text = coinMod._coins;
  }
}
```

实际上，MVC不仅可以用于UI系统（只是在大部分情况下，它天然地适合于此），也可以拓展引用到其他模块上，例如<a href="https://github.com/jskyzero/MVC.DotNet/tree/master">这个仓库</a>所展示的玩家系统。

# 参考资料
https://bbs.huaweicloud.com/blogs/306798
