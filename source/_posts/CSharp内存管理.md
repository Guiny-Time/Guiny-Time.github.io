---
tags: ['C#', '.NET', 'Unity', 'GC', '内存管理', '语言基础']
categories: 语言
title: CSHARP内存管理
date: 2024-09-16
updated: 2024-09-16
cover: https://imgix.datadoghq.com/img/blog/engineering/dotnet-continuous-profiler-part-4/il-1313-net-profiler-blog-pt4-240520-final.png
---

# 在C#应用的背后...
> C#是微软推出的一种基于.NET框架和后来的.NET的、面向对象的高级编程语言。C#衍伸自C和C++，继承了C和C++的强大功能，同时去掉了一些复杂特性，使其成为C语言家族中高效强大的编程语言。C#以.NET框架类库作为基础，拥有类似Visual Basic的快速开发能力。微软在2000年发布了这种语言，希望借助这种语言来取代Java。

![](https://celestialsys.com/wp-content/uploads/2020/02/NET-LANGUAGES.png)

.NET 框架中有一些但看缩写非常类似的概念，例如：CLI/CIL/CLR/CTS/CLS/JIT/GC 等等。

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

## .NET中的GC
C# 中堆区内存由 GC 负责清理，当对象超出作用域范围或者对象失去指向的引用地址，就会在一定时间内进行统一的处理，无需程序员手动处理。为什么要使用GC呢？也可以说是为什么要使用内存自动管理？有下面的几个原因：

1. 提高了软件开发的抽象度；

2. 程序员可以将精力集中在实际的问题上而不用分心来管理内存的问题；

3. 可以使模块的接口更加的清晰，减小模块间的偶合；

4. 大大减少了内存人为管理不当所带来的Bug；

5. 使内存管理更加高效。

.NET 的 GC 分为两个阶段：标记和压缩。在GC的过程中，还会用到代数算法的概念，尽可能的减少GC的开销。

### 代数算法

<img src="https://pic4.zhimg.com/v2-8b089380d3908280fd621838a8d5e8d5_r.jpg" />

GC 的代数（Generational）算法指将托管堆分成 0 代、1 代和 2 代（这里的“**代**”可以理解成“**辈分**”，比如 0 / 1 / 2 代分别指儿子、父亲和爷爷）。该算法基于以下几个考虑因素：

1. 压缩托管堆的一部分内存要比压缩整个托管堆速度快。

2. 较新的对象生存期较短，而较旧的对象生存期则较长。

3. 较新的对象趋向于相互关联，并且大致同时由应用程序访问。

垃圾回收主要在回收**短生存期对象**时发生。分代使 .NET 可以单独处理长生存期和短生存期对象。当 0 代托管堆内存满了或者没有足够的空间分配内存时就会触发 0 代GC，此时 0 代中幸存的对象将会进入 1 代（大多数对象通过第 0 代 GC 被回收，不会保留到下一代）。

如果第 0 代托管堆的回收没有回收足够的内存供应用程序创建新对象就会触发 1 代 GC，对 0 代和 1 代内存进行标记-压缩，幸存的对象将会进入 2 代。2 代 GC 也称为完整垃圾回收（full GC），因为它回收所有代中的对象（即，托管堆中的所有对象），开销较大。

对于标记-压缩算法来说，0 代 GC 和 1 代 GC 的压缩速度是很快的，因为这两个代龄加起来总是保持在 16M 左右，而 2 代就很大了（可能达到几个G）。0 / 1 代 GC 的耗时可能在几毫秒到几十毫秒，而 2 代 GC 可以达到几秒。

通常来说新分配的对象都属于 0 代，但如果新对象是大型对象，它们将延续到大型对象堆 (LOH)，这有时称为第 3 代。 第 3 代是在第 2 代中逻辑收集的物理生成。

0 代 / 1 代 / 2 代触发 GC 的频率比为 100：10：1，由此可见代数越低生存周期越短，代数越高生存周期越长。

> 将代数算法的基本思想与社会学中的某些想法相关联也不是不行，我甚至觉得这非常地狱笑话。

### 标记-压缩算法

<img src="https://upload.wikimedia.org/wikipedia/commons/thumb/0/03/Table-based_heap_compaction_flattened.svg/280px-Table-based_heap_compaction_flattened.svg.png" />

C# 的 GC 算法通常采用标记-压缩（Mark-Compact）算法。该算法分为三个阶段：

1. **标记清除**（Mark-Sweep）阶段，找到并创建所有活动对象的列表。
  C# GC 采用可达性判断而不是引用计数的方式进行标记（避免了循环引用的问题）。假设托管堆中所有对象都可以回收，然后从根（静态、全局、局部、函数调用参数、当前CPU寄存器中的对象指针、finalization queue等）出发找出能到达的对象并标记。最后托管堆中没有打标记的对象都是不可达、没有引用的对象，可以被回收。

2. **重定位**阶段，用于更新对将要压缩的对象的引用。
这一步是因为接下来的压缩过程将会移动托管堆中的对象，对象地址发生变化，需要修复所有引用指针，包括 stack、CPU register 中的指针以及托管堆中其他对象的引用指针。

3. **压缩**（Compact）阶段，用于回收由死对象占用的空间。
  由于对象回收之后托管堆内存空间变得不连续，需要压缩幸存的对象（类似于磁盘空间的碎片整理）。
  压缩阶段将垃圾回收中幸存下来的对象移至段中时间较早的一端。由于复制大型对象会造成性能下降，通常不会压缩大型对象堆 (LOH)。 

### Finalization Queue和Freachable Queue
这两个队列和 .NET 对象所提供的 Finalize 方法有关，这两个队列并不用于存储真正的对象，而是存储一组指向对象的指针。如果一个对象含有 Finalize 方法，则在 Finalization Queue 中添加一个指向该对象的指针。

在GC被启动以后，经过标记阶段分辨出哪些是垃圾。再在垃圾中搜索，如果发现垃圾中有被 Finalization Queue 中的指针所指向的对象，则将这个对象从垃圾中分离出来，并将指向它的指针移动到 Freachable Queue 中。这个过程被称为是对象的复生（Resurrection），因为这个对象的 Finalize 方法还没有被执行，所以不能让它直接被销毁，而是等待 Freachable Queue 去触发所指对象的 Finalize 方法执行，之后将这个指针从队列中剔除，这时对象才真正被销毁。

> **题外话**
.NET Framework 的 System.GC 类提供了控制 Finalize 的两个方法，ReRegisterForFinalize 和 SuppressFinalize。前者是请求系统完成对象的 Finalize 方法，后者是请求系统不要完成对象的 Finalize 方法。ReRegisterForFinalize 方法其实就是将指向对象的指针重新添加到 Finalization Queue 中。这就出现了一个很有趣的现象：
>- 在 Finalization Queue 中的对象可以复生到 Freachable Queue 中
>- 对象的 Finalize 方法中调用 ReRegisterForFinalize 方法
>- 对象回到 Finalization Queue 中
这样就形成了一个在堆上永远不会死去的对象，像凤凰涅槃一样每次死的时候都可以复生。

## Unity中的GC
虽然 Unity 的脚本采用 C# 编写，但 Unity 的 GC 不等同于 C# 的 GC，二者不能混为一谈。Unity 的内存管理是由脚本运行时（Mono 或 IL2CPP）处理的，并不是 CLR。

Unity 使用保守的 Boehm-Demers-Weiser (BDW) 垃圾收集器，在触发 GC 时它会停止程序的执行，并仅在完成其工作后恢复正常执行。BDW的工作算法可以描述如下：

1. 垃圾收集器**暂停程序执行**以进行 GC。

2. 根扫描：扫描根，确定所有可达的活动对象，创建可访问对象的图表。

3. 引用计数：计算每个对象的引用数量。

4. 内存回收：垃圾收集器释放没有引用的对象（死对象）占用的内存。

5. 垃圾收集完成后，程序继续执行。

可以看出BDW存在两大弊端，一是执行时会暂停程序导致游戏卡住（而且因为没有代数算法，每次GC都需要扫描所有对象），二是标记清除后没有进一步的压缩和指针修复，产生大量内存碎片。

为了解决第一个问题，从 Unity 2019.1 开始，BDW 默认以**增量模式**使用。这意味着垃圾收集器将其工作负载分布在多个帧上，而不是停止主 CPU 线程来处理托管堆中的所有对象。增量模式整体上不会加速垃圾收集，但由于它将工作负载分布在多个帧上，因此减少了与 GC 相关的性能峰值。

因此行业内有时会有这样的说法：与其触发 GC 产生大量碎片，不如直接滥用对象池。除此之外，一些优化方式还包括：

- 将值类型装箱，避免传递值类型变量而不是引用类型变量（一些方法的输入参数类型可能是object）
- 采用 StringBuilder 处理字符串操作（字符串不可变性）
- 在协程中警惕创建新的 WaitForSeconds 变量，而是复用
- 避免闭包和匿名函数
- 避免 LINQ 和正则表达式
- 使用不会产生垃圾的函数。例如使用 GameObject.CompareTag 而不是将字符串与 GameObject.tag 进行比较，因为返回新字符串会产生垃圾
- 在一些场合使用结构体而不是类

在实际开发中，可以使用 Profiler 等性能分析工具具体分析。

![](https://unity-connect-prd.storage.googleapis.com/20201231/learn/images/858bbe75-86a4-482b-af7e-82b46f2683e9_image4.png)

# 参考资料
https://zhuanlan.zhihu.com/p/38799766
https://medium.com/my-games-company/memory-mastery-comparing-unity-and-net-garbage-collection-4c23e693d3a5
https://www.youtube.com/watch?v=RM5N_p28d8c&list=PLs0ItJ2M3S9oebJ4hx6Vk32grry0XSHf8
https://celestialsys.com/blogs/an-introduction-to-net-framework/
https://zhuanlan.zhihu.com/p/113513147
https://blog.yusong.me/dotnet/memory#%E6%95%B0%E6%8D%AE%E4%B8%8E%E5%9C%B0%E5%9D%80
https://learn.microsoft.com/zh-cn/dotnet/csharp/programming-guide/types/boxing-and-unboxing
https://learn.microsoft.com/zh-cn/dotnet/standard/garbage-collection/fundamentals
https://www.bilibili.com/video/BV1oq4y1H7wz/
有关于结构体实现接口：https://blog.csdn.net/snakorse/article/details/28889611

