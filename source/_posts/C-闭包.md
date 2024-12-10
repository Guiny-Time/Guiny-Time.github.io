---
title: C#进阶（一）-闭包
date: 2023-02-13 20:07:22
tags: ['C#', '闭包', '特性', '语言基础']
categories: 语言
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA0431fbacc38d0fdf8123bab71d5622f0.png
---

# 什么是闭包？
C#中的**闭包**（Closure）指函数可以访问其**定义作用域之外的变量**（外部变量）。换句话说，一个闭包就是一个“捕获”或“携带”了其生成的环境中、所引用的**自由变量**的函数。这个被引用的自由变量将和这个函数一同存在，即使已经离开了创造它的环境也不例外。不止C#中存在闭包的特性，JS中也存在闭包。

C#闭包最常见的例子是在**匿名函数**（或**Lambda表达式**）中捕获自由变量，例如：

```C#
public static Action a;
    public static void Main(string[] args) {
        for (int i = 0; i < 5; i++) {
            a += () => {
                // 闭包捕获i
                Console.WriteLine(i);
            }; 
        }
    a();
}
```

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA63f3ccd4b622113dd2a3aa81bd432f72.png" alt="输出结果" />

为什么输出全是 `5` 而不是 `1, 2, 3, 4, 5` 呢？那是因为for循环中的变量 `i` 并不在匿名函数的作用域内，为了能够使用 `i`，在匿名函数中形成了一个闭包存放 `i`。通过 Rider 的 IL Viewer 工具，我们可以检查代码反编译后的结果，可见由编译器为匿名函数生成的密封类\<\>c__DisplayClass1_0中出现了一个内部变量 `i`，这就是存放外部变量的地方了：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAbe7446a2ec3507b3f79c33f8c782520c.png" alt="右边是反编译后的C#代码" />

有别于一般匿名函数或委托的静态特性（即编译后生成的包装类是静态的，这样可以减轻GC压力），闭包的匿名函数在每次使用时（在上面的例子中一共使用了五次），都需要 `new` 一个新的匿名函数实例出来。下面是反编译后的 Main 函数：

```C#
public static void Main(string[] args) {
    Program.<>c__DisplayClass1_0 cDisplayClass10 = new Program.<>c__DisplayClass1_0();
    for (cDisplayClass10.i = 0; cDisplayClass10.i < 5; cDisplayClass10.i++)
        Program.a = (Action) Delegate.Combine((Delegate) Program.a, (Delegate) new Action((object) cDisplayClass10, __methodptr(<Main>b__0)));
    Program.a();
}
```

在上面的代码结束之后，cDisplayClass10.i 变为了 `5`，这正是actions们捕获到内部的 `i` 变量指向的。因此在后续执行时，输出的结果全部都是5。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA966501e6fa443d70cbb43e6a906612d2.png" width=400 />

想要避免出现这样的问题，只需要在 for 循环里新建一个内部变量去暂存i的值，再在匿名函数中使用新建的内部变量即可。每次循环中新建的内部变量的地址都不相同，所以输出将会是 `1, 2, 3, 4, 5`。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA375ee4a73bb0b6042be0ebcf8dfb4abb.png" width=400 />

# 闭包变量的生命周期
由前文可知，闭包中存放了外部变量，这一行为可以**延长外部变量的生命周期**。例如：

```C#
public Action action;
public void TestFunc2(){
    int num = 1234;
    action = () =>{
        Console.WriteLine(number);
    }
}
```

`number` 作为函数 `TestFunc2` 的成员变量，理应在函数执行结束后被释放，但委托 `action` 捕获了它形成闭包，因此每次调用 `action` 时都会输出 `number` 的值（1234）， `number` 的生命周期和 `action` 同步了。

除此之外，还有一个避免出现该问题的方法：在闭包中使用foreach而不是for。至少在 C# 7.0+ 版本中这个问题已经被注意到了，使用 foreach 的时候编译器会自动生成代码绕过这个闭包陷阱。另外在C# 7.0+ 中，用本地函数代替委托可以在编译时生成结构体而非类，作为值类型的结构体可以在方法跑完后就立即释放，不需要等待垃圾回收。所以在官方的推荐中，如果委托的使用不是必要的，更推荐使用本地函数而非匿名函数。

```C#
// 匿名方法
static void GetClosureFunction1()
{
    int val = 10;
    Func<int, int> internalAdd = x => x + val;

    Console.WriteLine(internalAdd(10));

    val = 30;
    Console.WriteLine(internalAdd(10));
}

// 本地函数
static void GetClosureFunction2() {
    int val = 10;
    int InternalAdd(int x) => x + val;

    Console.WriteLine(InternalAdd(10));

    val = 30;
    Console.WriteLine(InternalAdd(10));
}
```

# 闭包可能带来的问题
首先就是内存泄漏的风险，如果忘记回收委托，委托将一直持有外部变量导致内存泄漏。开发者应该注意及时移除事件绑定。

其次是GC压力。存在闭包的匿名函数每次调用时都会new一个新的出来：

```C#
Program.<>c__DisplayClass1_0 cDisplayClass10 = new Program.<>c__DisplayClass1_0();
```

最后是不小心用了外部变量的引用时又遇到了多线程，可能会因为共享外部变量而造成预期之外的结果。采用临时变量或锁可以避免这个问题。

因此，我们应该谨慎小心地使用闭包。

# 使用场景
感觉 C# 闭包似乎里里外外都是问题，那它有什么实际的应用场景吗？闭包的核心能力是可以捕获外部变量，这特别适用于延迟执行的场景，例如<a href="https://www.bilibili.com/video/BV1Q84SeSEeA">视频</a>中为一系列按钮赋id。在赋值的函数结束之后，仍可在外部通过委托打印id。基于这个特性，闭包还可以起到缓存的作用。

闭包的另一个常见应用场景是在异步编程中，尤其是在捕获异步执行的上下文时。

```C#
async Task DownloadFiles(List<string> fileUrls) {
    foreach (var url in fileUrls) {
        string currentUrl = url; // 避免捕获 url 的引用
        await Task.Run(() => {
            Console.WriteLine($"Downloading {currentUrl}");
            // 下载逻辑
        });
    }
}
```

# 参考资料
https://juejin.cn/post/7167641621515730981
https://www.bilibili.com/video/BV1Q84SeSEeA
https://cloud.tencent.com/developer/article/2094857
https://www.cnblogs.com/eventhorizon/p/9535289.html