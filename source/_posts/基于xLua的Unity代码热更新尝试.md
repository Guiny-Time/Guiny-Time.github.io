---
title: 热更新其一：基于xLua的Unity代码热更新尝试
date: '2024-09-10T11:23:22.046Z'
tags: ['Lua','xLua','编程', '热更新', 'demo', 'Unity']
categories: 探索发现
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA9084d78948da0d4d0bc77e1bfeb0ea1c.png
---

最近用 Unity 做了一个移动端纯 UI 应用的尝试，应用的功能挺简单的，涉及到UI界面的响应、适配和切换等等，其中有一个部分是“通知”：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA447b2f4229b1aad0ba2dfc6e1d21e9d5.png" width=200 />

一般情况下“通知”都会显示为<u>暂无通知</u>，但是在实际项目中，开发者需要依照需求对通知的内容进行更改，比如改成“<u>关于十周年庆典的活动说明</u>”：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAdb2e86356d2dba0f078b92c548e16fac.png" width=200 />

每次更新通知的时候，替换文本的内容、重新打包成apk发送给用户让他们下载就可以实现版本更新了。但是这样的体验对用户来说显然是糟糕透顶的，为了用这玩意每次更新用户都得重新安装整个应用，太浪费流量了，对用户留存率来说是一个威胁。

在实际的游戏项目中，我们通常采用**热更新**的方式来实现版本更新。游戏热更新是指在不需要重新编译打包游戏的情况下，在线更新游戏中的一些代码和资源，比如活动运营和打补丁。用户每次只需要在本地获取需要更新的资源就可以了，不用重新下载客户端。

在实际开发中，我们通常会把**业务逻辑**与**整体架构**分离，将业务逻辑放在可以热更新的模块中，方便开发者随时修改业务数据（比如数值策划提出需要修改某个伤害计算公式）。

<img src="https://media.9game.cn/gamebase/ieu-gdc-pre-process/images/20230825/12/12/f5ef099e70928bdd2d8757a536862ecd.jpg" />

那么该怎么实现热更新呢？热更新分为**资源**热更新（例如图片素材、音频）和**代码**热更新，在这篇文章中我们将先介绍基于 **xLua** 的代码热更新。**Lua** 热更新的原理是将逻辑代码转化为脚本、脚本转化为文本资源，最后以更新资源的形式更新程序。

> 所以在下一篇文章，我会通过 Assets Bundle 和 Addressables 做一些关于资源更新方面的尝试。

# 为什么热更新采用Lua？

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAe8c98ca8b1f41d9a6160bda96591880d.png" width=300 />

Lua 是一种基于 **C** 实现的、轻量级的嵌入式脚本语言，可以方便地通过内置的 API 借助 Lua 虚拟**栈**与其他语言（例如C、C++）相互调用。但我们在 Unity 中使用的语言是 C#，为什么要多此一举地用 Lua 来做热更新呢？只是因为它轻量吗？

C# 是一种**编译型语言**，在 Unity 项目打包时经过编译器处理被编译成 **IL**（中间语言）后再借助 Unity 的 **mono** 或 **IL2CPP** 虚拟机（类似于 java 的 JVM）编译成汇编代码供不同平台执行。正因如此，打包后的代码难以修改，但也不是完全没有办法修改的，比如我们可以把一些需要频繁改动的业务逻辑打包成动态链接库（.dll文件），利用AB包上传下载后再利用 C# 的<a href="https://cattyhouse-guiny.xyz/2023/02/17/c-%E8%BF%9B%E9%98%B6%EF%BC%88%E4%BA%8C%EF%BC%89-%E5%8F%8D%E5%B0%84/">反射</a>进行代码更新。

既然思路存在，为什么我们不这么做呢？原因在于**IOS不允许获取具有可执行权限的内存空间，禁止应用程序动态生成机器码并执行**。我们若想采用 .dll 的方式进行热更新，需要依赖 **JIT**（即时编译器）在运行时将外部代码映射到应用的地址空间，并将其解释为平台对应的机器码。这一过程中，JIT 会开辟一段内存空间，该内存需同时满足**可读、可写和可执行**的权限要求（RWX）。随后，JIT 会将生成的机器码写入这段内存，并修改 CPU 的指令指针寄存器，使其指向该内存地址，从而执行新生成的机器码。IOS的这种限制导致 JIT 要以 **full AOT** 模式编译，这种模式会在**生成之前而不是运行期间**把 IL 直接翻译成机器码，进行了这种操作的 C# 从某种角度来说和 C++ 一样成为了编译型语言，失去了**运行时解释的功能**。

> **采用C#实现的热更：ILRuntime和HybridCLR（huatuo）**
如果能绕过 JIT 限制，C# 能否实现热更新呢？这里就存在两种 C# 热更框架：
>- **ILRuntime**：将C#代码编译成 IL 代码后在运行时通过 IL 解释器将其转换成机器码执行。内置一个 C# 脚本语言解释器，**解释执行** .net 字节码。
>- **HybridCLR**：IL2CPP 可以将 .net 字节码转成静态 C++ 代码，然后编译成机器码执行。HybridCLR 拓展了 IL2CPP 的运行时库，往里添加一个可以装载和**解释执行** .net 的字节码的功能。解释执行时直接使用 .net 内存对象，没有数据跨域的问题。

而 Lua 本身就是一种**解释型脚本语言**，将逻辑以脚本形式运行，不是通过 JIT 将 IL 转为机器码。Lua 使用 **C** 代码进行解释，不用开辟满足 RWX 要求的内存空间，也不会有新代码在执行，执行的始终是 Lua 虚拟机本身。同样作为解释型语言的 **Python** 也可以作为热更新使用的脚本语言，但 Lua 更轻便，也有很多成熟的框架，所以首选 Lua。

## Lua与xLua
我们可以在 Unity 项目的 C# 环境里构建一个**Lua虚拟环境**，Lua 脚本就可以通过这个虚拟机调用 C# 代码，并把值传递给 C# 代码使用。这个交互是通过一个**栈**结构来实现的，栈的特点是**先进后出**，存入栈的数据类型包括数值, 字符串, 指针, talbe, 闭包等，它们会被封装进 lua 的 TValue 结构体中。我们可以通过 lua API 操作这个栈（压入/弹出数据）。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA0923b2adc0f80859559c482ccc9e92f3.png" width=350 />

Lua虚拟环境怎么构建呢？这里我们将采用一种在商业项目中常用的框架——<a href="https://github.com/Tencent/xLua">xLua</a>。xLua 为 Unity、.Net、Mono 等 C# 环境增加 Lua 脚本编程的能力，借助 xLua，这些 Lua 代码可以方便的和 C# 相互调用。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA357627c784740543737302dd8e7259b1.png" width=200 />

下载xLua仓库的源码，打开Asset文件夹，将里面的文件导入Unity。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA9b6dc034ddd162845a237946b2ba29e8.png" width=300 />

在项目中，创建一个Lua管理器脚本（LuaMgr.cs）用于创建和管理 Lua 虚拟环境。通过这个虚拟环境，我们可以直接用 `DoString` 方法执行一些 Lua 代码：

```C#
using UnityEngine;
using XLua;

public class LuaMgr : MonoBehaviour
{
    private XLua.LuaEnv _luaEnv;    // lua环境

    void Start()
    {
        _luaEnv = new XLua.LuaEnv();    // 创建lua环境
        _luaEnv.DoString(@"
            -- 可以直接写lua代码
            a = 10;
            print(a+5);
        ");
    }
}
```

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAfbe3f1392911fd092554712ad3866881.png" alt="输出结果" />

在同一个 Lua 环境中（比如上面的 `_luaEnv` ）定义的变量的通用的。比如在 Update 中输出 a + 5 的值，得到的还会是 `LUA: 15` 这个输出。

```C#
void Update() {
    _luaEnv.DoString(@"
        print(a+5);
    ");
}
```

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAa9285eced983575f1d94a3ca8f3312f2.png" width=400 alt="输出结果" />

## Lua与Unity交互
### Lua获取C#脚本
那 Lua 要怎么和 Unity 进行交互呢？譬如我想通过 Lua 更改通知文本的值？首先，我们需要通过命名空间把需要交互的代码包裹起来，引用 xLua 并在需要与 Lua 交互的类上方标注 `[LuaCallCSharp()]`，如下所示：

```C#
using TMPro;
using UnityEngine;
using XLua;

namespace LuaTest
{
    [LuaCallCSharp()]
    public class UISys : Singleton<UISys>
    {
        public TMP_Text newsLetter;
        
        // 获取当前通知
        public string GetNews()
        {
            return newsLetter.text;
        }
        // 设置通知
        public void SetNews(string news)
        {
            newsLetter.text = news;
        }
    }
}
```

在 LuaMgr 中，我们就可以直接通过 `DoString` 在 Lua 环境中实例化 UI 管理器对象。有了对象之后，我们就可以调用其中的方法了：

```C#
_luaEnv.DoString(@"
    -- 实例化UI管理器
    UISys = CS.LuaTest.UISys.GetInstance()
    curNews = UISys:GetNews()
    print(curNews)
");
```

在 Unity Editor 的 xLua 选项卡点击“generate code”，完成之后运行程序：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAa30f682d1431550f611dae4008e3e539.png" />

### C#调用Lua
在 C# 中，可以将 Lua 方法传入对应的委托容器中，然后执行就可以了：

```C#
[CSharpCallLua]
public delegate int CheckUpdateNews();
private CheckUpdateNews _checkUpdateNews;
public delegate void UpdateNews();
private UpdateNews _updateNews;

void Start() {
    _luaEnv = new XLua.LuaEnv();    // 创建lua环境
    _luaEnv.DoString(@"
        UISys = CS.LuaTest.UISys.GetInstance()
        oldNews = UISys:GetNews()
        newNews = '十周年庆典活动说明'

        function needUpdateNews()
            if oldNews == newNews then
                return 1
            end
        end

        function updateNews()
            CS.LuaTest.UISys.GetInstance().SetNews(newNews)
        end
    ");
    // C# 通过委托获取Lua函数检查是否需要更新
    _checkUpdateNews = _luaEnv.Global.Get<CheckUpdateNews>("needUpdateNews");
    _updateNews = _luaEnv.Global.Get<UpdateNews>("updateNews");
    if (_checkUpdateNews() == 0) {
        UISys.GetInstance().SetNews(_luaEnv.Global.Get<string>("newNews"));
        _updateNews();
    }
}
```

所以在上面的逻辑中，我们只需要动态替换 `newNews` 的值就可以实现广播内容的替换逻辑了。我们把这个脚本打成 AB 包传到服务器，客户端在检查时发现存在新版本就会提示玩家拉取更新了。

在下一篇文章中，我会介绍一下如何用 AB 包和 Addressables 实现资源热更新。

# 参考资料
https://github.com/Tencent/xLua
https://www.jianshu.com/p/71fffc9a9cac
https://www.cnblogs.com/sevenyuan/p/4511808.html