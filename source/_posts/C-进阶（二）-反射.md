---
title: C#进阶（二）-反射
date: 2023-02-17 23:12:36
tags: ['C#', '反射', '特性', '语言基础']
categories: 语言
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAfe5b9f1fc358476fcf0edc84ea82ed93.png
---

# 什么是反射？
反射指程序可以**访问、检测和修改它本身或其他程序集状态或行为**的一种能力。开发者可以使用反射动态地创建类型的实例，将类型绑定到现有对象，或从现有对象中获取类型，还可以调用类型的方法或访问其字段和属性。

换句话说，反射指的是程序在运行时可以查看其他程序集或自身的元数据。一个运行的程序查看本身或者其他元数据的行为就叫做反射。函数在执行时，通过反射可以得到其他程序集或自己程序集代码的各种信息（即**元数据**信息，包括类、函数、变量、对象等等），可以实例化它们、执行它们或操作它们。

反射可以在程序编译后获得信息，提高了程序的拓展性和灵活性。
1. 程序运行时得到所有元数据，包括元数据的特性
2. 程序运行时实例化对象、操作对象
3. 程序运行时创建新对象，用这些对象执行任务


>- **程序集**
程序集是由编译器编译得到的、供进一步编译执行的中间产物，在windows系统中一般表现为后缀为.dll（库文件）或.exe（可执行文件）的文件

>- **元数据**
描述数据的数据，例如程序中的类、类中的函数、变量等信息就是程序的元数据。元数据保存在程序集中

# 反射语法
## Type
Type类是**类的信息类**，也是实现反射的基础。Type是访问元数据的主要方式，可以通过它的成员获取有关类型**声明的信息**和有关类型的**成员**（如构造函数、方法、字段、属性和类的事件）。

如何获取Type呢？有以下三种方法（通过这三种方式得到的type指向堆中的同一个内存，每个类型的type都是唯一的）：

```C#
int a = 1;  // a属于System.Int32类型
// 1、通过Object下的GetType获取
Type type = a.GetType();
// 2、通过typeof关键字传入类名获取
Type type2 = typeof(int);
// 3、通过类的名字（包含命名空间）获取
Type type3 = Type.GetType("System.Int32");
```

得到Type之后，我们就可以进一步得到类里的信息了，比如程序集信息：

```C#
Console.WriteLine(type.Assembly);
```

假设我们有一个类名字叫Test，它长这样：

```C#
public class Test{
    // 成员变量
    public string name = "Adam";
    public int age = 1;
    private bool gender = false;
    // 默认无参构造函数
    public Test(){ }
    // 有参构造函数
    public Test(bool gender){
        this.gender = gender;
    }
    public Test(bool gender, string name):this(gender){
        this.name = name;
    }
    // 成员函数
    public void Intro(){
        Console.WriteLine("Hello, my name is " + this.name);
    }
}
```

我们就可以通过反射获取Test类中的所有**公共成员**（私有变量gender无法得到），这是一个 `MemberInfo` 类型的数组，数据内容为类型 + 成员名，例如 `Void Intro`。公共成员不仅包含了公共变量，还包含公共的构造函数和公共的函数。

```C#
using System.Reflection;
// ....
Type t = typeof(Test);
MemberInfo[] infos = t.GetMembers();
```

当然我们也可以专门获得Test类的**公共构造函数**，这是一个 `ConstructorInfo` 类型的数组。获取的数据内容为 `Void .ctor()`，括号内为构造函数的输入类型，例如 `System.String`。

获取到构造函数之后，还可以通过反射调用它们：

```C#
// 获取所有公共构造函数
ConstructorInfo[] ctors = t.GetConstructors();
// 根据参数类型获取一个构造函数，例如下面是无参构造的构造函数：
ConstructorInfo ctor = t.GetConstructor(new Type[0]);
Test obj = ctor.Invoke(null) as Test;  // 执行
// 获取一个有参构造的构造函数：
ConstructorInfo ctor = t.GetConstructor(new Type[]{typeof(int)});
Test obj2 = ctor.Invoke(new obect[]{23}) as Test;  // 执行，按顺序传入的参数类型一定要和构造函数匹配
```

既然可以拿到Test类的所有公共构造函数，那我们当然也可以拿到Test类的所有**公共成员变量**并动态修改它们的值，方法如下：

```C#
// 获取所有公共成员变量
FieldInfo[] fields = t.GetFields();
// 通过名字查找指定的公共成员变量信息
FieldInfo f = t.GetField("name");    // System.String name
// 获取和设置对象的值
Test lsl = new Test(true, "STARON");
// 获取名字
Console.WriteLine(f.GetValue(lsl));
// 设置名字
f.SetValue(lsl, "SORTZE");
```

最后，我们可以通过类似的操作获取Test类的**公共成员方法**。如果方法存在重载，则用Type数组表示方法类型（类似公共构造函数小节）：

```C#
MethodInfo[] methods = t.GetMethods();
// 根据名字和参数列表获取对应方法
MethofInfo method = t.GetMethod("Intro", new Typr[0]);
// 传入参数调用（如果是静态方法，Invoke中的第一个参数传null即可）
method.Invoke(null, null);  // 第一个参数是要执行的对象，第二个参数是object[]，传入的是参数
```

除此之外Type类还可以获取类的枚举、事件、接口和属性，方法都是类似的，即通过
- GetEnumNames
- GetEvents
- GetInterfaces
- GetPropertys

方法获取。

## Activator
`Activator` 用于快速实例化对象的类，用于将Type对象快速实例化为对象。

回忆一下通过 `Type` 实例化一个对象的过程：首先需要获取特定参数（也可以是无参）的**构造函数**，然后利用 `Invoke` 传入对应的参数数组（也可以是无参）去调用这个构造函数，最后返回实例化完成的对象。

这其实有点复杂，但通过 `Activator` ，我们可以这么做：

```C#
Type t = typeof(Test);
// 无参构造
Test obj = Activator.CreateInstance(t) as Test;
// 有参构造，第二个参数为变长参数数组1object[]，因此直接传入参数类型即可
Test yby = Activator.CreateInstance(t, false, "GALITY") as Test;
```

`Activator` 非常方便，但参数一定要记得与构造函数的一一对应。

## Assembly
`Assembly` 类就是程序集类，主要用途是用于**加载其他程序集**（比如.dll文件）。加载完成后，才能通过 `Type` 获取对应程序集中的信息。正是因为 `Assembly` 的存在，反射可以轻松创建、调用和修改其他程序集中的内容，使其成为一个强大的功能。`Assembly` 类中有三种加载程序集的函数：

```C#
// 加载同一文件夹下的其他程序集
Assembly ass = Assembly.Load("程序集名称");
// 加载不同文件夹下的程序集
Assembly ass2 = Assembly.LoadFrom(@"C:\Users\Guiny Time\RiderProjects\ConsoleAppFuncTest\ConsoleAppFuncTest\bin\Debug\ConsoleAppFuncTest"); // 包含程序集清单的名称或路径
Assembly ass3 = Assembly.LoadFile("要加载程序集的完全限定路径");    // 要加载程序集的完全限定路径
```

获得程序集之后，可以获得其中的所有类型：

```C#
// 获取所有Types
Type[] types = ass2.GetTypes();
// 获取指定的Type
Type t = ass2.GetType("ConsoleAppFuncTest.Program");
```

有了具体的 `Type` 之后，我们就可以动态调用其他程序集的代码了。

# 如何自己创建类库（.dll）
针对不同的应用程序有不同种类的类库（例如针对Windows的、针对.NET的）。在Unity开发的过程中，我们通常可以创建一个C#（.NET Framework）的类库，把一系列工具封装到类库中，再在实际工厂中通过反射获取并执行其中的方法。

创建类库的方式很简单，现代编辑器都提供了默认的模板，比如在我个人常用的Rider在新建项目时就有Class Library的选项：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAae3203716307d4ee95be3ae4e23fd3b0.png" />

有别于控制台应用，类库是纯写代码的，没有办法运行，只能被“**生成**”，打包得到一个.dll文件（程序集）。

# 参考资料
https://www.runoob.com/csharp/csharp-reflection.html
https://www.bilibili.com/video/BV1Ar4y1K7AK