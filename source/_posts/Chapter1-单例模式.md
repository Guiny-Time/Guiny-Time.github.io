---
tags: ['游戏编程','经验分享','单例模式','设计模式','Unity']
title: 游戏中的单例模式
date: '2022-01-02T05:54:50.681Z'
updated: '2022-01-02T07:58:54.814Z'
cover: https://res.cloudinary.com/ddxwdqwkr/image/upload/f_auto/v1614961724/patterns.dev/singleton-pattern.jpg
---

# 什么是单例模式？
这学期开始教设计模式这门课程，其中第一节课所介绍的就是所有设计模式中最简单也最好理解的**单例模式**。单例模式是几大设计模式中最常被使用的模式之一，在游戏编程的实战中也有很多用途，比如说维护一个**全局唯一**的UI管理器、**全局唯一**的特效/音效控制器等等。该模式适合针对“有唯一需求，不希望存在多个”的系统。也就是说，**单例模式的实例同时只能存在 0 个或者 1 个**，实例化的过程分为懒实例（懒汉式）、饥渴实例（饥饿式）等。

> 《游戏编程模式》对于单例模式的定义为：“确保一个类只有一个实例， 并为其提供一个全局访问入口。”

## 类图
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/homework1_1.png"/>

## 代码实现

### final + 懒实例 + 线程锁实现法
**final** 关键字（等同于 C# 中的 **sealed**）的主要目的是防止单例类被**继承**。如果去掉 final，内部类可能会继承单例类，调用内部类的时候会造成**多实例化**的问题。然而在一些特定的情况下，你**可以继承单例**，这是一个强大但是经常被忽视的特性。

> 假设我们需要让文件封装类跨平台，为了实现这一点，我们可以将单例实现为一个抽象接口，并由它的子类提供各个平台上的实现。

什么是**懒实例/懒汉式**？简单地说，懒实例指的是**延迟初始化**，是一种延迟对象创建过程的技术。懒实例允许只在需要时才创建对象（而不是在程序开始时就创建完成），当开发者需要处理开销很大的流程来创建对象时，这种方法很有帮助。

至于为什么需要加**线程锁**（thread lock）呢？原因显而易见，因为我们的程序可能工作在多线程（multi threaded）环境下，**线程安全**至关重要。而如果两个线程同时实例化单例类的实例，可能造成线程冲突（thread conflict），从而得到两个单例类的实例，这显然是不符合单例期望的。所以在多线程环境下，当有两个或以上个线程访问程序时，给方法加上线程锁是必要的。这样的结果是：同一时间只能有一个线程访问代码，其余线程需要等待，避免的冲突的形成。

虽然 Unity 只有一个游戏主线程，但开发者可以通过 Thread 类自定义新开线程以利用 CPU 在多个核心上同时处理多个线程的能力。这些新线程并行运行，通常在完成后将其结果与主线程同步，因此使用单例时依旧需要注意线程安全的风险。

> **题外话：线程池**
为什么使用线程池？线程池主要目的在于**线程资源复用**和**自动管理**。当面对一个任务时开发者可以采用开一个线程的方式来加速运算，然而当任务数量不断增加，自己创建线程的方式不再合理，主要原因有下：
>- 创建和关闭新线程是需要消耗CPU时间的，因为操作系统需要分配给新开的线程地址空间、栈空间、寄存器等，在线程结束的时候，操作系统又将这些东西回收。线程池相比于线程来说，不需要频繁的创建和销毁线程，线程一旦创建之后，默认情况下就会一直保持在线程池中，等到有任务来了，再用这些已有的线程来执行任务
>- 用户可能创建了大量的线程，而这些线程不是每时每刻都在工作（可能处于休眠状态等待事件发生），需要定期唤醒才能轮询更改或更新状态信息。线程池是复用已有线程来执行任务的，而线程是在有任务时才新建的，所以相比于线程来说，线程池能够更快的响应任务和执行任务。此外线程池可以创建固定的线程数，从而避免了无限创建线程的问题。

**优点**
- 通过双重检查和锁保证线程安全
- 加载时快速
- 只有在实例尚未初始化时才会进入锁定代码块，降低锁的开销。

**缺点**
- 代码冗长
- 程序执行过程中需要进行实例化，存在开销问题

在 C# 中，可以采用双重锁来实现线程安全：

```C#
using System;

namespace SingletonPattern {
    public sealed class UIController {
        // 唯一的静态实例
        private static UIController _uiController;
        // 锁
        private static readonly object _lock = new object();

        private UIController() { }

        // 懒实例，在需要（调用获取方法）的时候再判断是否需要实例化，对处理实例化开销比较大的环境下很有利
        public static UIController GetUIController() {
            get {
                // 第一次检查，避免不必要的锁开销
                if (_uiController == null) {
                    lock(_lock) {
                        // 第二次检查，确保线程安全
                        if(_uiController == null) {
                            _uiController = new UIController();
                            Console.WriteLine("A new UI Controller initialized.");
                        } else {
                            Console.WriteLine("Already has a UI controller");
                        }
                    }
                }
                return _uiController;
            }
        }
    }
}
```

测试类：

```C#
namespace SingletonPattern {
    internal class Program {
        public static void Main(string[] args) {
            Console.WriteLine("===单例模式===");
            UIController ui1 = UIController.GetUIController();
            UIController ui2 = UIController.GetUIController();
            // 二者实际上是等价的，全局只存在一个单例的实例
            if (ui1 == ui2)
                Console.WriteLine("These 2 UI Controller re the same.");
        }
    }
}
```

> **输出:**
===单例模式===
A new UI Controller initialized.
Already has a UI controller
These 2 UI Controller re the same.

在现代 C# 开发中，还可以通过 Lazy\<T\> 的方式实现懒汉式单例。Lazy\<T\> 专门用于实现延迟初始化，并且内置了线程安全机制，相比于双重锁的写法更为简洁优雅，以下是例子：

```C#
namespace SingletonPattern {
    using System;

    public class UIController {

        // 使用 Lazy<T> 实现线程安全的延迟加载
        private static readonly Lazy<UIController> _uiController = new Lazy<UIController>(() => new UIController());

        // 私有构造函数，防止外部实例化
        private UIController() { }

        public static UIController GetUIController {
            get {
                Console.WriteLine("Get a UI Controller.");
                return _uiController.Value;
            }
        }
    }
}
```

### 饥渴实现法（Eager Initialization）
和懒实例化相对，饥渴实例（饿汉式）在程序加载之初就完成了实例化，之后在需要使用相关方法时直接返回实例。

**优点**
- 简单明了的代码
- 线程安全，虽然不用线程锁，但是在开始的时候就实例化完了
- 静态字段只初始化一次，程序执行过程中开销小

**缺点**
- 加载时间漫长，因为需要实例化所有单例

```C#
using System;

namespace SingletonPattern
{
    public class MusicMgr
    {
        // 静态只读字段，在首次访问时自动初始化
        private static readonly MusicMgr _musicMgr = new MusicMgr();

        // 私有构造函数，防止外部实例化
        private MusicMgr()
        {
            Console.WriteLine("A music manager has been initialized.");
        }

        public static MusicMgr GetMusicMgr()
        {
            Console.WriteLine("Game has a music controller now.");
            return _musicMgr;
        }
    }
}
```

### Bill Pugh法
Bill Pugh 法是一种优雅的实现单例模式的方式，特别适用于 Java。这种方法通过使用一个**静态内部类** Helper 来延迟加载单例实例，静态内部类在调用它们的getInstance() 方法之前不会加载到内存中，只有当有人调用 getCaptain() 方法时，才需要考虑Helper类。同时 Bill Pugh 法避免锁的性能开销，并保证线程安全。
C# 没有静态内部类的概念，可以使用嵌套类模拟相同的行为：

```C#
public class Singleton
{
    // 私有构造函数，防止外部实例化
    private Singleton() { }

    // 静态嵌套类，用于延迟初始化单例
    private class SingletonHelper {
        // 单例实例，使用只读字段保证线程安全
        internal static readonly Singleton Instance = new Singleton();
    }

    // 公共访问方法
    public static Singleton Instance {
        get {
            return SingletonHelper.Instance;
        }
    }
}
```
SingletonHelper 类不会在 Singleton 类加载时初始化，而是在 Instance 属性被调用时才加载。这一行为依赖于 .NET 的类型加载机制，即只有在使用静态成员时，静态类或嵌套类才会初始化。

C# 的静态构造函数由 CLR（公共语言运行时）负责初始化，并且保证线程安全。internal static readonly 字段的初始化是线程安全的，CLR 确保了不会出现竞争条件。

**优点**
- 线程安全，无须显式加锁，减少了代码复杂度。
- 仅在需要时才实例化单例对象。
- 由于没有锁和检查的开销，性能优于双重锁。
- 简单明了的代码

# 注意！
虽然单例模式看上去简单易用也很好用，但它实在太容易被滥用了。单例作为一个全局变量，在代码中可能导致一下问题：

1. 它们令代码晦涩难懂。不小心在错误的地方错误地修改全局变量面临着高额的调试工作量，因为需要检查每一个引用。

2. 全局变量加剧了耦合。

3. 不利于并发，因为线程不安全的单例可能造成死锁。

大部分单例都以“Manager”、“System”、“Engine”一类的名字命名，它们有时也被称为**保姆类**。为了尽可能避免单例调用分布在代码库的各个角落，我们可以试着只在基类中使用单例。我们也可以考虑将所有单例对象包装到一个现有的类里，只通过这个类来访问各个系统（虽然依然会散布在各处，但查找起来只需要查这个类）。

# 参考资料
[美] Robert Nystrom 尼斯卓姆. 游戏编程模式 (游戏设计与开发). 人民邮电出版社. 
https://blog.csdn.net/weixin_43405845/article/details/105022859