---
tags: ['设计模式','游戏开发','MVC','框架','Unity','UI']
categories: 设计模式
title: MVC设计模式
date: 2021-05-19
update: 2021-05-25
cover: https://assets.toptal.io/images?url=https%3A%2F%2Fbs-uploads.toptal.io%2Fblackfish-uploads%2Fcomponents%2Fblog_post_page%2F4084336%2Fcover_image%2Fretina_500x200%2Fcover-unity-with-mvc-how-to-level-up-your-game-development-46aaec316442778385f4c4149a0f5273.png
---

MVC 模式的全称是 Model-View-Controller，即模型-视图-控制器。严格来说，MVC并不是GoF（四人帮）归纳出的一种独立设计模式，而是一种架构模式。这种模式常用于用户图形界面的图形 / 数据 / 逻辑分离，因此在游戏开发领域中常被用于 UI 相关的功能上。那么，什么是 M / V / C 呢？

- **M**（模型）
模型层（M 层）负责**存放数据**和**业务逻辑**。
- **V**（视图）
视图层（V 层）负责**展示用户图形界面**，是用户与之直接交互的元素。
- **C**（控制器）
控制层（C 层）负责**响应用户输入或事件**，更新 M 的数据并让 V 展示的界面刷新。

使用 MVC 模式的目的是将 M 和 V 的实现代码分离，从而使同一个程序可以使用不同的表现形式。比如一批统计数据可以分别用柱状图、饼图来表示。C 存在的目的则是确保 M 和 V 的同步，一旦 M 改变，V 应该同步更新。

# Unity中的MVC
![](https://img-blog.csdnimg.cn/20181106152450894.png)
![](http://xpsuper.com/upload/2021/10/MVC01-da44a1398db94f8987ec9aa716cb1725.png)

以上述框架为例，假设我们要做一个点击按钮更新金币数量（即点击一次金币数 +1 ）和清空当前金币的功能：

![](https://velog.velcdn.com/images/suhan0304/post/d6333381-8221-4cae-9f3c-f7d21ed3e978/image.png)

## Model（M）
M 层是**纯数据类**，不继承 Monobehavior，主要作用是存放所有数据，包括静态数据、变量、资源（如贴图、游戏对象等）等。对于 Model 的数据管理，可以通过**序列化**或 **ScriptableObject** 管理数据。

```C#
[System.Serializable]
public class CoinModel {
    public int Coins { get; private set; }
    public event Action<int> OnCoinsChanged;

    public void AddCoin() {
        Coins++;
        OnCoinsChanged?.Invoke(Coins);
    }

    public void ResetCoins() {
        Coins = 0;
        OnCoinsChanged?.Invoke(Coins);
    }
}
```

既然我们强调 M 是**纯数据类**，为什么这里出现了增加金币和重置金币的逻辑 `AddCoin` 和 `ResetCoins` 呢？

> 其实在刚接触 MVC 时，我对 MVC 和 ECS 两个设计模式产生了混淆，认为 M 层类似 ECS 中的 Entity 只能存放数据，而业务逻辑应该放在 C 层（或者说 ECS 的 System），这其实是不对的。

M 层通常既包含数据，也包含与这些数据相关的业务逻辑，主要原因有下：

1. 避免**贫血模型**
如果 Model 只存储数据，而所有业务逻辑都放到 Controller 中，那么 M 层就沦为了一种数据结构，这在很多架构思想中被称为“贫血模型”，容易导致业务逻辑分散在各个 Controller 中，后期维护和测试都不便。

2. Model 应该表达**某个业务**（例如金币相关管理）
一般来说，Model 不仅仅是数据的容器，它往往是对现实业务的抽象，包括该业务对象的属性和行为。在例子中的金币场景里，“增加金币”和“重置金币”就是金币的业务行为，与金币这个实体强相关，放在 CoinModel 中更符合面向对象的思路（而非 ECS 的面向数据思维），也让逻辑更清晰可测试。

3. Controller 应该负责**调度**
Controller 的职责是响应用户输入、调用 Model 的方法或更新 Model 的数据，然后让 View 去刷新到用户界面上。C 层通常不承担具体的业务运算，而是调用 Model 中的方法。如果把所有业务逻辑都丢给 Controller，会造成 Controller 过度膨胀，跟 View 绑定得更死，后期扩展也会麻烦。

## View（V）
V 层负责管理所有 UI 组件对象，并为它们声明相关的事件：

```C#
// View（管理 UI组件，监听 Model 变化）
public class CoinView : MonoBehaviour {
    [SerializeField] private Text coinText;
    [SerializeField] private Button addButton;
    [SerializeField] private Button resetButton;

    public void Initialize(CoinModel model, Action onAdd, Action onReset) {
        model.OnCoinsChanged += coins => coinText.text = coins.ToString();
        addButton.onClick.AddListener(() => onAdd?.Invoke());
        resetButton.onClick.AddListener(() => onReset?.Invoke());
    }
}
```

在 V 层中，我们添加了所有需要更新的和可以和玩家发生交互的 UI 组件（非静态 UI），例如上面的按钮和金币文本。除了手动一个个赋值之外，也可以在 `Start` 函数中通过 `Find` 方法查找并赋值。

`Initialize` 方法则为 UI 元素绑定了相应的事件，该方法在 Controller 中被初始化。

## Controller（C）
C 层负责持有对 Model 和 View 的引用，响应用户输入或事件，更新 Model 并让 View 刷新。从下面的代码可以看出，Controller 确实只起到了调度和协调的作用，是 M 层和 V 层之间的桥梁。

```C#
// Controller（协调 Model 和 View）
public class CoinController : MonoBehaviour {
    [SerializeField] private CoinModel model;
    [SerializeField] private CoinView view;

    private void Start() {
        view.Initialize(model, model.AddCoin, model.ResetCoins);
    }
}
```

---

在用户点击增加金币的按钮时，MVC 架构下的 UI 发生了这样一系列的事情：

1. 按钮被点击，触发 `OnClick` 事件，通知所有订阅者
2. View 中绑定的 `onAdd` 回调被执行
3. Model 中的 `AddCoin` 函数被执行，更新金币数量同时触发 `OnCoinsChanged` 事件
4. View 中绑定的回调被执行，更新金币文本

实际上，MVC 模式不仅可以用于 UI 系统（只是在大部分情况下，它天然地适合于此），也可以拓展引用到其他模块上，例如<a href="https://github.com/jskyzero/MVC.DotNet/tree/master">这个仓库</a>所展示的玩家系统。

# 参考资料
https://bbs.huaweicloud.com/blogs/306798
