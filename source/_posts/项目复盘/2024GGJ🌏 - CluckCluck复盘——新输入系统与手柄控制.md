---
title: "2024GGJ🌏 - CluckCluck复盘——新输入系统与手柄控制"
tags: ['游戏编程','经验分享','项目复盘','新输入系统','Unity']
categories: 项目复盘
date: '2024-02-01'
updated: '2024-02-16'
copyright_author: 时光
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA858c8347ff07ec951cb49ffb04eea56b.png
---

# 新输入系统(Input System)
Unity的新输入系统是一个基于事件的新系统，它在编辑器里提供了一个中间层（Input Action Asset），用户可以在其中自定义一系列玩家动作和其对应的操作。例如在CluckCluck中，我设定了移动（left joystick）、冲刺（B）、拾取道具（right trigger）三个控制：

![](image0.png)

完成操作设置之后，保存Input Action资产，在其检查器中可以点击生成对应的C#类，接下来就可以在代码中通过事件绑定的方法使用这些操作监听。

![这里已经生成过C#类了，所以没有generate C# class的选项](image.png)

# 同屏双手柄（1p2p）

由于我们做的是一个比较欢脱的双人派对竞技游戏，这次jam还有一个重要的需求是同屏双手柄控制。我不想像懂哥一样用“PlayerController1.cs”和“PlayerController2.cs”来区分不同玩家（因为如果采取这种模式，如果玩家数量增多，需要维护的脚本就会变多），所以如何在一个脚本中为不同玩家绑定不同的控制器尤为重要。
首先是声明p1/p2，它们都用了同一套InputAction “Player2”（之所以是这个命名是因为Player1是键盘控制）。通过游戏对象的名称是否包含“1”来判断玩家是否是p1，如果是则为该玩家绑定全部游戏控制器列表的第一个控制器；否则绑定第二个控制器。
```csharp
p1 = new Player2();
p2 = new Player2();
var allGamepads = Gamepad.all;
if (this.gameObject.name.Contains("1"))
{
    var user1 = InputUser.PerformPairingWithDevice(allGamepads[0]);
    user1.AssociateActionsWithUser(p1);
}
else
{
    var user2 = InputUser.PerformPairingWithDevice(allGamepads[1]);
    user2.AssociateActionsWithUser(p2);
}
```


# 参考资料
https://docs.unity3d.com/Packages/com.unity.inputsystem@1.2/api/UnityEngine.InputSystem.IInputActionCollection2.html
https://learn.microsoft.com/en-us/dotnet/api/system.idisposable?view=net-8.0
https://zhuanlan.zhihu.com/p/106396562
https://www.reddit.com/r/Unity3D/comments/eqq07o/multiple_controllers_through_input_action_system/
