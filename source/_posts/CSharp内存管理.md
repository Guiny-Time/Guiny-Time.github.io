---
tags: [游戏开发/通用]
title: CSHARP内存管理
date: 2024-09-16
updated: 2024-09-16
cover: https://imgix.datadoghq.com/img/blog/engineering/dotnet-continuous-profiler-part-4/il-1313-net-profiler-blog-pt4-240520-final.png
---

# 在C#应用的背后...
> C#是微软推出的一种基于.NET框架和后来的.NET的、面向对象的高级编程语言。C#衍伸自C和C++，继承了C和C++的强大功能，同时去掉了一些复杂特性，使其成为C语言家族中高效强大的编程语言。C#以.NET框架类库作为基础，拥有类似Visual Basic的快速开发能力。微软在2000年发布了这种语言，希望借助这种语言来取代Java。

![](https://celestialsys.com/wp-content/uploads/2020/02/NET-LANGUAGES.png)

.NET 框架中有一些但看缩写非常类似的概念，例如：CLI/CIL/CLR/CTS/CLS/JIT/GC等等。

在**通用语言基础架构** (Common Language Infrastructure) 中，C# 在构建时会被编译为 **CIL** (Common Intermediate Language)，即**通用中间语言**（在一些地方也被称为**字节码**/ByteCode）。在.Net开发平台下，所有语言（C#、VB.NET、J#、C++/CLI）都会被编译为通用中间语言，再由 **CLR** (Common Language Runtime)，即**通用语言运行时**负责运行。

在 CLR 中，CLR 通过 **JIT** (Just In - Time Compiler)，即**即时编译器**在运行时将中间语言转换成存储在内存中的**机器码**（native code），因为转换是在程序执行的过程中逐渐进行而不是一次性完成的因此被称为“即时”。由于机器码被存储，因此不需要多次转换。

CLR 还包含了**公共语言规范** (CLS，负责将各种 .NET 编程语言语法规则和规定转换为 CLR 可理解的格式)、**通用类型系统**（CTS，负责理解 .NET 编程语言的数据类型并将其转换为 CLR 可理解的格式，即值类型和引用类型）和**垃圾收集**（GC，提供自动内存管理功能）。

# C#的内存是什么样的
## 内存分区
在C#中，内存分成5个区，他们分别是：
- **堆**
用于动态分配内存，**存放引用类型**对象本身。在 C# 中由 .NET 平台的垃圾回收机制（GC）管理，容量大但分配和释放速度相对较慢。栈、堆都属于动态存储区，可以实现动态分配。
- **栈**
由编译器自动分配释放，通常容量较小，速度快。存放：
  - **值类型**的**对象本身**
  - 引用类型的**引用地址**（指针）
  - 静态区对象的**引用地址**（指针）
  - 常量区对象的**引用地址**（指针）等。
- **全局/静态存储区**
用于存储静态成员（静态变量和静态方法）、静态类、全局变量等，它们在整个程序生命周期中一直存在，直到程序运行结束时才会被释放和回收。
- **常量存储区**
存储程序中的常量值的对象本身，通常是**只读**的，编译时已确定，存储在程序的数据段中。
- **代码段**
用于存储存放编译后的程序指令，通常是只读的，在程序运行时被执行。

> **为什么要在内存中对堆(Heap)和栈(Stack)进行区分？**
将内存划分为堆和栈是为了**更高效地管理内存**，并针对不同类型的数据进行优化处理。
>- 栈用于存储值类型和函数调用的局部变量，分配**快**但**内存空间有限**。栈的内存分配是**自动且快速**的，基于LIFO（后进先出）结构，分配和释放内存效率高。栈上分配的数据**生命周期短**，作用范围仅限于当前作用域。
>- 堆用于存储引用类型，适合更大的对象，内存分配更灵活、管理更复杂，访问速度**慢**。能够支持**长生命周期**的数据。在堆上分配的对象需要手动或通过垃圾回收器来释放，避免内存泄漏。

## 值类型和引用类型
C#中有且只存在两种变量类型：**值类型**(value type)和**引用类型**(reference type)，它们都隐式继承自万物之源 System.Object 类（不过几乎所有的引用类型都是直接从 Systerm.Object 继承，而值类型则是继承 Systerm.Object 的子类 Systerm.ValueType 类），区别在于拷贝方式的不同。

<img src="https://eamonkeane.dev/content/images/2021/02/Frame-1-1.png" width=400/>

- **值类型**
  - 包含了常见的简单类型（如int、double、float、char、bool）、结构体 struct 和枚举 enum。
  - 拷贝的是值本身（深拷贝）
  - 不可以被派生，不支持多态。
  - 大小最好不要超过64字节（可能与 CPU 缓存行通常是 64 字节，如果 CPU 缓存行不能装下一个完整的值类型数据可能会有性能问题有关），适合作为存储数据的载体。
  - <u>**通常来说**</u>存储在栈上，可以被自动释放，内存管理上高效。为什么说“通常来说”呢？因为如果一个值类型的变量被一个引用类型所封装，那么这个值类型变量也会被存放在**堆**上。

<img src="https://eamonkeane.dev/content/images/2021/02/Frame-2.png" width=400/>

- **引用类型**
  - 包含了类、接口、string和各种类型的数组。
  - 拷贝的是值的引用，指向的是同一个地址里的值（浅拷贝）
  - 可以派生，支持多态。
  - 大小可以很大，适合定义应用程序的行为。
  - 存储在堆上，需要GC或手动释放，相对低效。

### 装箱与拆箱
从值类型转换到引用类型的过程就是“**装箱**”，反过来就是“**拆箱**”。相对于简单的赋值而言，装箱和取消装箱过程需要进行大量的计算，因此需要避免频繁发生装箱/拆箱的操作。

#### 装箱
![在这里，o在栈中存放的是i在堆中的地址1](https://learn.microsoft.com/zh-cn/dotnet/csharp/programming-guide/types/media/boxing-and-unboxing/boxing-operation-i-o-variables.gif)

装箱是将值类型转换为 object 类型或由此值类型实现的任何**接口类型**的过程。常见语言运行时 (CLR) 对值类型进行装箱时，会将值包装在 System.Object 实例中并将其存储在**托管堆**中，装箱是**隐式**的。

```C#
int i = 123;
// The following line boxes i.
object o = i;
```

#### 拆箱
![](https://learn.microsoft.com/zh-cn/dotnet/csharp/programming-guide/types/media/boxing-and-unboxing/unboxing-conversion-operation.gif)

拆箱将从对象中提取值类型，是从 object 类型到值类型或从接口类型到实现该接口的值类型的**显式转换**。拆箱操作包括：
- 检查对象实例，以确保它是给定值类型的装箱值。
- 将该值从实例复制到值类型变量中。

```C#
int i = 123;      // a value type
object o = i;     // boxing
int j = (int)o;   // unboxing
```

要在运行时成功拆箱值类型，被取消装箱的项（例如上面代码中的 o ）必须是对一个对象的**引用**，该对象是先前通过装箱该值类型的实例创建的。尝试取消装箱 null 会导致 NullReferenceException。尝试拆箱对不兼容值类型的引用会导致 InvalidCastException。

# 托管与非托管
在上文中提到了“**托管堆**”的概念，那么这里的“托管”是什么意思呢？是否还存在“非托管堆”呢？
托管堆也被称之为**托管内存**，是被 CLR（公共语言运行时）所托管的内存。存在**非托管内存** (unmanaged memory)的说法，它也被称之为**原生内存** (native memory)。

在这里我们还要引入**托管资源**和**非托管资源**的概念。
- 托管资源
这是指托管堆上分配的内存资源，由 CLR 在适当的时候调用 GC 进行回收。比如我们对类或者委托等引用类型使用 new 关键字，那么创建的对象就被分配在托管内存中。
- 非托管资源
指的是 CLR 不知道如何回收的内存资源。最常见的一类非托管资源如文件、窗口、网络连接、数据库连接、笔刷、图标等。对非托管资源需要编写代码手动回收（如实现 IDisposable 接口），因此不会产生 GC。

# 垃圾回收/GC(Garbage Collection)
在前文中我们反复提到了 GC 的概念，这究竟是什么呢？
> 在计算机科学中，垃圾回收（英语：Garbage Collection，缩写为GC）是指一种自动的存储器管理机制。当某个程序占用的一部分内存空间不再被这个程序访问时，这个程序会借助垃圾回收算法向操作系统归还这部分内存空间。垃圾回收器可以减轻程序员的负担，也减少程序中的错误。

C# 中堆区内存由 GC 负责清理，当对象超出作用域范围或者对象失去指向的引用地址，就会在一定时间内进行统一的处理，无需程序员手动处理。为什么要使用GC呢？也可以说是为什么要使用内存自动管理？有下面的几个原因：

1. 提高了软件开发的抽象度；
2. 程序员可以将精力集中在实际的问题上而不用分心来管理内存的问题；
3. 可以使模块的接口更加的清晰，减小模块间的偶合；
4. 大大减少了内存人为管理不当所带来的Bug；
5. 使内存管理更加高效。

关于GC，知乎的这篇文章写的很好：https://zhuanlan.zhihu.com/p/38799766

# 参考资料
https://www.youtube.com/watch?v=RM5N_p28d8c&list=PLs0ItJ2M3S9oebJ4hx6Vk32grry0XSHf8
https://celestialsys.com/blogs/an-introduction-to-net-framework/
https://zhuanlan.zhihu.com/p/113513147
https://blog.yusong.me/dotnet/memory#%E6%95%B0%E6%8D%AE%E4%B8%8E%E5%9C%B0%E5%9D%80
https://learn.microsoft.com/zh-cn/dotnet/csharp/programming-guide/types/boxing-and-unboxing
https://www.bilibili.com/video/BV1oq4y1H7wz/
有关于结构体实现接口：https://blog.csdn.net/snakorse/article/details/28889611

