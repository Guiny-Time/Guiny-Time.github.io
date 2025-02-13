---
title: Mono & IL2CPP
date: 2023-10-02 15:43:23
updated: '2024-02-04 23:05:17'
categories: 探索发现
tags: ['Unity','跨平台','Mono','IL2CPP','编译','C#','C++']
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA63f27977094652a0092e0ac3fe928047.png
---

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAbfaee249f1d3c5d59577cd3af31683fd.png" alt="一次构建，到处运行" />

1995 年，Java 编程语言横空出世，依托**虚拟机**（**JVM**）的 Java 打着“**一次编写，到处运行**”的**跨平台**旗号，迅速赢得了开发者的青睐，成为当今应用最广泛的十大编程语言之一。

Unity 同样做到了跨平台的特性，它允许开发者在完成游戏开发工作后将其打包发布到多达 27 个目标平台上（如上图所示）~~Unreal做得到吗~~。那么 Unity 是如何实现跨平台的操作呢？它也使用到了类似 JVM 的概念吗？这就要提到 Unity 的**脚本后端处理**（Scripting Backend）了。Unity 具有两个脚本后端：**Mono** 和 **IL2CPP** (Intermediate Language To C++)，它们各自使用不同的编译技术，也是今天本文探讨的主题。在正式开始之前先学习一下 .NET 是怎么被编译成目标平台的代码的：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAde5206458a07165b33de269678ebf4f6.png" alt="C#的编译过程" />

1. 编写 C# 代码
2. C# 编译器编译代码（在 Unity 中就是每次更改完代码之后漫长的进度条等待环节）后得到了中间语言（Intermediate Language Code，IL）。
3. IL 经过公共语言运行时（CLR）和即时编译器（JIT）的编译，得到目标平台的可执行程序集。

可见跨平台的关键就在中间语言到程序集的中间，也就是 CLR 所负责的部分。Mono 和 IL2CPP 是不是对这里下刀的呢？

# Mono
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAe966e7a325b7e39f033df6c4725e4623.png" alt="mono logo" width=100 />

Mono 是基于 .NET 框架的开源开发平台，是 CLR 的开源实现。Mono 允许开发人员构建具有提高开发人员生产率的跨平台应用程序，由以下几个部分组成：

- **C# 编译器**：用于编译 C# 代码。
- **Mono 运行时**：在一些地方也被称作 **Mono VM**（Mono 虚拟机），实现了 CLI，提供 JIT（即时编译器）、AOT（提前编译器）、GC、库加载程序、线程系统等。
- **.NET Framework 类库**：一套与 Microsoft .NET Framework 兼容的类库，提供了构建应用程序所需的基础类，包括集合、输入/输出、网络、安全等。
- **Mono 类库**：Mono 项目中超出 .NET Framework 类库的部分，提供了许多额外的功能。

{% note info %}
**什么是类库（Class Library）？**

**类库**是面向对象编程中一组**预先编写好的类的集合**，这些类可以被开发者在编写程序时直接使用，是十分有用的工具。在 Unity 开发的过程中，我们经常能用到的 `System.Collections`、`System.IO` 等命名空间就源自 .NET Framework 类库。
{% endnote %}

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA9056fe7f474e85c6f3a09582e5bb71cc.png" width=500 />

由上图可见，在 Unity 中 Mono 的工作流程如下：

1. C#，Unity Spcript 和 Boo 通过各自的编译器转成中间语言（IL）
2. IL 在 Mono 虚拟机上被编译成目标平台代码
3. 运行程序

Mono 虚拟机提供了两种编译方式：**即时编译**（JIT，在运行时按需编译代码）和**提前编译**（AOT）。在打包时，选择 JIT 或 AOT 会在第二步上产生一些差异：

- **JIT 模式**下 IL 与 Mono 虚拟机会一起打入包中，IL 在应用程序中通过 Mono 虚拟机解释代码，将指令映射到虚拟内存中执行（ios 禁止了这一行为，因此必须使用 AOT）
- **AOT 模式**会生成相应平台的**机器码**，不会将 Mono 虚拟机打入包中。

使用 Mono AOT 编译模式可以提高应用程序的**启动速度**和**运行时性能**，因为它将 IL 代码编译为本地机器码，减少了运行时的解释和优化过程。但这样 C# 就和编译型语言或者解释型语言没有区别了，失去了**动态代码特性**（例如反射、动态生成优化代码等）。

既然 Mono 是一个强大的跨平台工具，为什么 Unity 还要推出 IL2CPP 呢？主要有以下几个原因：
1. 运行效率一般。
2. 由于版权原因，Unity 无法使用最新版本的 Mono，这限制了开发者对 .NET 新特性的使用。
3. Mono VM 在各个平台移植、维护非常耗时，有时甚至不可能完成。
> Mono 的跨平台是通过 Mono VM 实现的，有几个平台，就要实现几个VM，像Unity这样支持多平台的引擎，Mono 官方的 VM 肯定是不能满足需求的。所以针对不同的新平台，Unity 的项目组就要把 VM 给移植一遍，同时解决 VM 里面发现的 bug。这非常耗时耗力。这些能移植的平台还好说，还有比如 WebGL 这样基于浏览器的平台。要让 WebGL 支持 Mono 的 VM 几乎是不可能的。

{% note orange 'fas fa-lightbulb' %}
**为什么有了 CLR 还需要用 Mono？它们的功能不是一样的吗？**

相信很多人在刚接触 Mono 的时候都会有这样的疑问。CLR 最初**仅支持 Windows**，没有办法做到把游戏发布到多平台上。Mono 在这样的历史前提下诞生了，它最初的设计目标就是实现**跨平台**。
相比于现代（.NET 5+）CLR，Mono 自身还是存在不足之处的，例如它的性能（**CG**、**JIT** 等方面）不如原生 CLR。随着 .NET Core 的崛起，Mono 逐渐被边缘化。
{% endnote %}

# IL2CPP
**IL2CPP** (Intermediate Language To C++) 是一种由 Unity 开发的新脚本后端，可在为各种平台构建项目时**替代** Mono。使用 IL2CPP 构建项目时，IL2CPP 会将 IL 转换为 **C++**，然后利用目标平台的 **C++ 编译器**生成本地可执行文件。由于 C++ 是一种**编译型语言**，需要在程序运行之前通过编译器将源代码转换为机器码或目标代码，因此 IL2CPP 使用**提前编译**（AOT），在运行之前编译整个应用程序，可以提高项目的性能、安全性和平台兼容性。 相较于 Mono，IL2CPP 主要有以下优点：

- **性能提升**
IL2CPP 将 IL 转换为 C++ 代码，然后使用平台特定的 C++ 编译器进行优化。同时 C++ 代码可以直接运行在本地环境中，减少了与虚拟机相关的开销。根据 Unity 官方的数据，使用 IL2CPP 后程序的运行效率可以提高 **1.5** 到 **2** 倍。
- **安全性**
IL2CPP 生成的 C++ 代码更加难以被反编译，这意味着游戏代码更加安全，难以被破解或修改。
- **平台兼容性**
IL2CPP 能够支持更多的平台，包括那些不支持 Mono VM 的平台（如 WebGL）。
- **版权问题**
IL2CPP 是 Unity 自己开发的，不像 Mono 那样会受到版本授权的限制，开发者可以使用最新的 C# 特性。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA00daf86ed09f6c0455857dfe9c1a277f.png" width=500 />

由上图可见，IL2CPP 打包的主要流程如下：

1. C#，Unity Spcript 和 Boo 通过各自的编译器转成中间语言（IL）
2. IL2CPP 将 IL 转换为 C++，并保留类型信息、方法调用等关键信息
3. 生成的 C++ 代码经过优化和整理后生产本地源码
4. 由于 C++ 没有 GC 一类的功能，生成的本地源码还包含了一些 **IL2CPP 运行时库**（即上图中的 **IL2CPP VM**）的功能，将 .NET 的垃圾回收等功能嵌入到生成的 C++ 代码中，以便在运行时管理内存的分配和释放
    - IL2CPP VM 在应用程序运行时起作用，它主要负责处理应用程序的运行时行为，包括**内存管理**、**线程创建**、**类型转换**、**异常处理**等服务性工作。由于不需要动态解析 IL，IL2CPP 的大小可以很小（降低最终包体）
    - IL2CPP VM 是作为编译后的应用程序的一部分打包到最终的可执行文件中的，而不是作为独立的组件进行加载

当然，IL2CPP 也存在缺点。因为是通过 AOT 而非 JIT 模式编译的，开发者无法在程序中使用动态语言特性（例如运行时生成代码）。

# 参考资料
https://zh.wikipedia.org/wiki/Java
https://docs.unity3d.com/cn/2021.1/Manual/scripting-backends.html
https://www.mono-project.com/docs/about-mono/
Unity将来时：IL2CPP是什么？ - https://zhuanlan.zhihu.com/p/19972689
【Unity游戏开发】Mono和IL2CPP的区别 - https://zhuanlan.zhihu.com/p/352463394