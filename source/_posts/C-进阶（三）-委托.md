---
title: C#进阶（三）-委托
date: 2023-02-23 23:12:36
tags: ['C#', '委托', '特性', '语言基础']
categories: 语言
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAd7b8b231ed9781bba1c0554bcf9d08a4.png
---

# 什么是委托？

委托（delegate）类似于C++中的**函数指针**，是类型化了的函数指针（一种可以存放特定参数/返回值类型方法的容器），可以作为**参数**一样被传递。

```C#
// 定义了一个返回值和参数为空的委托类型
private delegate void MyDelegate();
// 声明MyDelegate类型的一个委托
private MyDelegate _myDelegate;

public void Main(){
    // 为委托赋值方法
    _myDelegate = Test;
    // 调用委托
    _myDelegate();
}

// 挂载到委托上的函数，其参数和返回值需与委托一致，否则不会被感知
public void Test(){
    Console.WriteLine("Test");
}
```

这时有人就会有疑问了：这和直接在 `Main` 中调用 `Test` 方法有什么区别呢？

```C#
public void Main(){
    Test(); // 执行结果和上面的代码块是完全一样的
}

public void Test(){
    Console.WriteLine("Test");
}
```

委托最主要的目的是作为参数传入函数中，充当**回调函数**的功能。例如在一个技能被触发之后除了通用的“播放技能音效”和“播放技能动画”之外，技能还需要执行对应的具体逻辑（例如传输、耗蓝、某些特殊效果）。如果为每个技能编写函数，就会变成：

```C#
private void Skill1(){
    PlayAudio();
    PlayAnim();
    Skill_1_Logic();
}
private void Skill2(){
    PlayAudio();
    PlayAnim();
    Skill_2_Logic();
}
private void Skill3(){
    PlayAudio();
    PlayAnim();
    Skill_3_Logic();
}
```

很显然这不是一个优秀的代码设计典范，存在太多重复的地方。而如果我们将技能逻辑视为一个参数，那么就可以通过委托将其抽象成：

```C#
private dedlegate void SkillLogicDelegate();
// 无参数无返回值
private void Skill(SkillLogicDelegate _skillLogic){
    PlayAudio();
    PlayAnim();
    _skillLogic();
}
```

## 多播委托
**多播委托**（multicast delegate）则代表一个委托存放了多个函数，我们可以通过 `+=` 操作符添加函数。

```C#
// 定义了一个返回值和参数为空的委托类型
private delegate void MyDelegate();
// 声明MyDelegate类型的一个委托
private MyDelegate _myDelegate;
public void Main(){
    _myDelegate += Test;
    _myDelegate += Test;
    _myDelegate += Test;
    _myDelegate.Invoke();
}

public void Test(){
    Console.WriteLine("Test");
}
```

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAc5931afdb24b6ccd90876b814c98e1f9.png" alt="输出结果" />

同样，我们也可以通过 `-=` 操作符从多播委托中移除函数。如果函数不存在则程序无反应。如果调用了一个空的委托，会产生 `NullReference` 报错，因此在使用 `Invoke` 调用委托时可以用 `?.` 语法糖进行判空。

```C#
// 定义了一个返回值和参数为空的委托类型
private delegate void MyDelegate();
// 声明MyDelegate类型的一个委托
private MyDelegate _myDelegate;
public void Main(){
    _myDelegate += Test;
    _myDelegate += Test;
    _myDelegate += Test;
    _myDelegate.Invoke();
    _myDelegate -= Test;
    _myDelegate?.Invoke();
}

public void Test(){
    Console.WriteLine("Test");
}
```

多播委托的底层是一个**数组**，我们称之为“**委托链**”，因此多播委托也可以被称为**链式委托**。通过下面的反编译结果可以看出，多播委托使用了 `Delegate.Combine` 函数。当两个及以上的委托被链接到一个委托链时，调用头部的委托将导致该链上的所有委托方法都被执行。通过 `System.MulticastDelegate` 中的 `GetInvocationList()` 方法，可以以数组的形式获得整个链式委托中的所有委托。

```C#
// 反编译结果
public static void Main(string[] args) {
    Program.a = (Action) Delegate.Combine((Delegate) Program.a, (Delegate) new Action((object) null, __methodptr(Test)));
    Program.a = (Action) Delegate.Combine((Delegate) Program.a, (Delegate) new Action((object) null, __methodptr(Test)));
    Program.a = (Action) Delegate.Combine((Delegate) Program.a, (Delegate) new Action((object) null, __methodptr(Test)));
    Program.a();
}
```

如果在某个委托方法中修改了委托链（例如 `+=` 了新的方法），新委托链是不会被执行的，因为本次执行的委托引用指向的是旧的链。委托是一种**引用类型**（类似于字符串一般具有不变的特性），在增加或移除方法时实际上创建了一个新的委托实例并把它赋给当前的委托变量。在多播委托执行时，C# 会先将当前委托链**拷贝**出来，然后按这个拷贝的顺序依次调用。这保证了即使在回调中修改了委托链，拷贝的链条也不会被改变。

```C#
public static Action a;
public static void Main(string[] args) {
    a += Test;
    a += Test;
    a += Test;
    a.Invoke();
    Console.WriteLine("_______");
    a.Invoke();
}
public static void Test() {
    Console.WriteLine("Test");
    a += Test;
}
```

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA0b570b2efde5ba89fd652973e0d5267a.png" alt="输出结果" />

需要注意的是，对于有返回值的多播委托，如果没有手动调用委托链上的每个方法，只能得到**委托链上最后被调用的方法的返回值**。

## 两种特殊的委托
每次使用委托，都需要先声明委托的类型（指定返回值和参数），再为该类型的委托声明一个实例去存储函数，步骤稍微有些繁琐。针对常用的两种类型的委托，C# 为我们封装了两种委托类型：`Action` 和 `Func`。

### Action
`Action` 是没有返回值的委托。

```C#
// 这样就声明了一个无参无返回值的委托实例
private Action _action;
// 泛型参数无返回值的委托实例
private Action<T> _action1;
```

### Func
`Func` 是有返回值的委托，其中尖括号的最后一个代表返回值（由 `ref` 修饰，所以是返回值）。

```C#
// 一个返回值为int类型、无参的委托实例
private Func<int> _func;
// 参数为两个int，返回值为int类型的委托实例
private Func<int, int, int> _func1;
```

## 匿名函数与Lambda表达式
有些时候，一个函数可能就在回调时使用了一次，不存在复用。这时如果再为这段代码去定义一个函数，可能就显得有些浪费空间。C#提供了**匿名函数**和**Lambda表达式**的功能，我们可以通过这两种写法为委托加上一段匿名的逻辑，例如：

```C#
private Action _myDelegate1;
private Func<int, int, int> _myDelegate2;

public void Main(){
    // Lambda表达式
    _myDelegate1 = () => {
        Console.WriteLine("Test1");
    };
    // 匿名函数
    _myDelegate2 = delegate(int a, int b){
        return a + b;
    };
    // 等同于
    // _myDelegate2 = (int a, int b) => { a + b;};

    _myDelegate1?.Invoke();
    int res = _myDelegate2.Invoke(1, 1);
    Console.WriteLine(res);
}
```

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA8d7f96b6ffd52061cb42e8fbd5f7aefd.png" alt="运行结果" />

Lambda表达式写起来特别精简，想必有函数式编程经验的小伙伴对这种写法比较熟悉，因为Lambda表达式是仿照函数式的编写方式。

处于委托的特性，委托适合用于回调函数和事件机制（通过观察者模式实现）中。

# 注意
`delegate` 实际上是一种特殊的**类**，是一个引用类型的变量，存放在托管堆上，因此会受到GC的影响，使用时需要注意避免内存泄漏等问题。如果没有及时通过 `-=` 移除函数，委托会一直持有它，这会导致一些资源无法被GC导致一些问题产生。

```C#
Program._myDelegate1 = Program.<>c.<>9__2_0 ?? (Program.<>c.<>9__2_0 = new Action((object) Program.<>c.<>9, __methodptr(<Main>b__2_0)));
Program._myDelegate2 = Program.<>c.<>9__2_1 ?? (Program.<>c.<>9__2_1 = new Func<int, int, int>((object) Program.<>c.<>9, __methodptr(<Main>b__2_1)));
Action delegate1 = Program._myDelegate1;
```

# 参考资料
https://learn.microsoft.com/zh-cn/dotnet/api/system.delegate.combine?view=net-8.0
https://www.bilibili.com/video/BV1vgpreoE2p