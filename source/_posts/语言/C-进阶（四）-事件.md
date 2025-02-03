---
title: C#进阶（四）-事件
date: 2023-02-27 12:23:36
tags: ['C#', '事件', '特性', '语言基础']
categories: 语言
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA694daa11e9bbd58f8a9deffbdbee5b84.png
---

# 什么是事件？
`event` 是C#的一个关键字，是对于委托的一种**更安全**的封装方式（只能作为成员存在于类/接口或结构体中），这使得被 `event` 关键字修饰的委托只能在声明它的类中被调用。

```C#
// 定义一个委托类型，用于事件处理程序
public delegate void NotifyEventHandler(object sender, EventArgs e);
// 用event修饰委托实例
public event NotifyEventHandler _processCompleted;
```

你一定好奇这里 `objrct` 类型的 `sender` 和 `EventArgs` 类型的 `e` 分别是什么。其实从命名上也可以看出来，`EventArgs` 表示事件参数（EventArguments）的类的基类，它可以用来记录事件传递的额外信息（例如触发的时间、位置等等）。在实际使用中，开发者需要自行定义派生自 `EventArgs` 类的事件参数，例如：

```C#
public class ButtonClickEventArgs : EventArgs {
    public string Time { get; set; }
}
```

`sender` （或者你随便起的名字，比如说 `o`）就是发送事件触发消息的源对象的**引用**，比如我们拿取 `sender` 和 `e` 的值并打印：

```C#
public delegate void NotifyEventHandler(object sender, ButtonClickEventArgs e);

// 发布者类
public class ProcessBusinessLogic {

    public event NotifyEventHandler ProcessCompleted;

    // 触发事件的方法
    protected virtual void OnProcessCompleted(ButtonClickEventArgs e) {
        ProcessCompleted?.Invoke(this, e);
    }

    // 触发事件
    public void StartProcess() {
        Console.WriteLine("Process Started!");
        ButtonClickEventArgs args = new ButtonClickEventArgs() {
            Time = DateTime.Now.ToString()
        };
        OnProcessCompleted(args);
    }
}

// 订阅者类
public class EventSubscriber {
    public void Subscribe(ProcessBusinessLogic process) {
        process.ProcessCompleted += Process_ProcessCompleted;
    }

    private void Process_ProcessCompleted(object sender, ButtonClickEventArgs e) {
        Console.WriteLine(sender.GetType() + " Process Completed!");
        Console.WriteLine("Completed time: " + e.Time);
    }
}

class Program {
    static void Main(string[] args) {
        ProcessBusinessLogic process = new ProcessBusinessLogic();
        EventSubscriber subscriber = new EventSubscriber();

        // 订阅事件
        subscriber.Subscribe(process);

        // 启动过程
        process.StartProcess();
    }
}
```

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAd87bce6ee3b1e3ce62c5f56f2751cffb.png" alt="输出结果" />
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA95292bb6fe4258262398f1c29b01270e.png" alt="日期时间，是相一致的" />

# 注意
事件不是委托实例，只是成对的 `add` / `remove` 方法（类似于属性的取值方法/赋值方法），因此在声明事件的类外部我们只能增加/移除事件的回调函数。事件之所以只能在声明它的类这调用，是因为其包装的委托是该类的一个**私有委托类型**字段，因此无法在外部访问。

下面是反编译的结果：

```C#
[CompilerGenerated]
[DebuggerBrowsable(DebuggerBrowsableState.Never)]
// 私有委托
private NotifyEventHandler ProcessCompleted;

public event NotifyEventHandler ProcessCompleted {
    // add方法
    [CompilerGenerated] add {
        NotifyEventHandler notifyEventHandler1 = this.ProcessCompleted;
        NotifyEventHandler notifyEventHandler2;
        do {
            notifyEventHandler2 = notifyEventHandler1;
            notifyEventHandler1 = Interlocked.CompareExchange<NotifyEventHandler>(ref this.ProcessCompleted, (NotifyEventHandler) Delegate.Combine((Delegate) notifyEventHandler2, (Delegate) value), notifyEventHandler2);
        }
        while (notifyEventHandler1 != notifyEventHandler2);
    }
    // remove方法
    [CompilerGenerated] remove {
        NotifyEventHandler notifyEventHandler1 = this.ProcessCompleted;
        NotifyEventHandler notifyEventHandler2;
        do {
            notifyEventHandler2 = notifyEventHandler1;
            notifyEventHandler1 = Interlocked.CompareExchange<NotifyEventHandler>(ref this.ProcessCompleted, (NotifyEventHandler) Delegate.Remove((Delegate) notifyEventHandler2, (Delegate) value), notifyEventHandler2);
        }
        while (notifyEventHandler1 != notifyEventHandler2);
    }
}
```

# 参考资料
https://www.runoob.com/csharp/csharp-event.html
https://learn.microsoft.com/zh-cn/dotnet/api/system.eventargs?view=net-8.0
https://www.cnblogs.com/erlongxizhu-03/p/12511275