---
title: "2024GGJ🌏 - CluckCluck复盘——新输入系统与手柄控制"
tags: ['游戏编程','经验分享','项目复盘','Unity']
categories: 项目复盘
date: '2024-02-01'
copyright_author: 时光
cover: https://ae01.alicdn.com/kf/S0da7b43026144fa7862df33e82e6d23bw.png
---

# 新输入系统(Input System)
Unity的新输入系统是一个基于事件的新系统，它在编辑器里提供了一个中间层（Input Action Asset），用户可以在其中自定义一系列玩家动作和其对应的操作。例如在CluckCluck中，我设定了移动（left joystick）、冲刺（B）、拾取道具（right trigger）三个控制：

<img src="https://ae01.alicdn.com/kf/S3c2e433acb414d73a030e4ca233e66eet.png" alt="image.png" width=600 />

完成操作设置之后，保存Input Action资产，在其检查器中可以点击生成对应的C#类，接下来就可以在代码中通过事件绑定的方法使用这些操作监听。

<img src="https://ae01.alicdn.com/kf/S02f9e988164e40c3a41a3cf86519e1304.png" alt="这里已经生成过C#类了，所以没有generate C# class的选项" title="这里已经生成过C#类了，所以没有generate C# class的选项" width=400/>

## 代码解析
生成的C#代码取决于用户设定的控制逻辑的复杂程度，实现了 **IInputActionCollection2** 和 **IDisposable** 接口。
### 接口
#### IInputActionCollection2
从命名上可以看出该接口用于过渡 **IInputActionCollection** 接口的功能。

```csharp
using System.Collections;
using System.Collections.Generic;

namespace UnityEngine.InputSystem
{
  public interface IInputActionCollection2 : 
    IInputActionCollection,
    IEnumerable<InputAction>,
    IEnumerable
  {
    IEnumerable<InputBinding> bindings { get; } // 迭代操作集合中的所有绑定

    InputAction FindAction(string actionNameOrId, bool throwIfNotFound = false);  // 通过名称或 id（字符串形式）在集合中查找 InputAction

    int FindBinding(InputBinding mask, out InputAction action); // 查找与给定mask匹配的第一个绑定的索引
  }
}
```

#### IDisposable
IDisposable是.NET里标准的释放 **非托管资源(unmanaged resource)** 的接口。非托管资源指无法在公共语言运行时（CLR）上运行的资源，例如文件。当用户完成对对象的工作并且不再需要它时，就可以对对象调用接口中的 **Dispose** 方法释放它。由于读取了Input Action文件导致无法被CLR自动GC，因此继承了该接口。

```csharp
    public void Dispose()
    {
        UnityEngine.Object.Destroy(asset);
    }
```

这个方法表示在资源释放时应该怎么办，他们直接摧毁了资产（是的（简单直接的

### 其他部分
我做了一些简化处理：
<details>
  <summary>生成的源码内容，进行了部分简化</summary>
  <markdown>

```csharp
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine.InputSystem;
using UnityEngine.InputSystem.Utilities;

public partial class @Player2: IInputActionCollection2, IDisposable
{
    public InputActionAsset asset { get; }
    public @Player2()
    {
        asset = InputActionAsset.FromJson(@"{
    ""name"": ""player2"",
    ""maps"": [
        {
            ""name"": ""Player"",
            ""id"": ""2b28ef6a-c40d-4bd2-bf6a-fe66aa27d755"",
            ""actions"": [
                {
                    ""name"": ""Move"",
                    ""type"": ""Value"",
                    ""id"": ""936e8d41-2517-4056-b264-8acb73f6efb5"",
                    ""expectedControlType"": ""Vector2"",
                    ""processors"": """",
                    ""interactions"": """",
                    ""initialStateCheck"": true
                },
                {
                    ""name"": ""Dash"",
                    ""type"": ""Button"",
                    ""id"": ""b76c25aa-6294-43a5-97fe-3d085d434ece"",
                    ""expectedControlType"": ""Button"",
                    ""processors"": """",
                    ""interactions"": """",
                    ""initialStateCheck"": false
                },
                {
                    ""name"": ""OpenMouse"",
                    ""type"": ""Button"",
                    ""id"": ""bbe3327d-20a0-4e47-ae92-c6f7dd5f315d"",
                    ""expectedControlType"": ""Button"",
                    ""processors"": """",
                    ""interactions"": """",
                    ""initialStateCheck"": false
                }
            ],
            ""bindings"": [
                {
                    ""name"": """",
                    ""id"": ""978bfe49-cc26-4a3d-ab7b-7d7a29327403"",
                    ""path"": ""<Gamepad>/leftStick"",
                    ""interactions"": """",
                    ""processors"": """",
                    ""groups"": ""Gamepad"",
                    ""action"": ""Move"",
                    ""isComposite"": false,
                    ""isPartOfComposite"": false
                },
                {
                    ""name"": """",
                    ""id"": ""3ea4d645-4504-4529-b061-ab81934c3752"",
                    ""path"": ""<Joystick>/stick"",
                    ""interactions"": """",
                    ""processors"": """",
                    ""groups"": ""Joystick"",
                    ""action"": ""Move"",
                    ""isComposite"": false,
                    ""isPartOfComposite"": false
                },
                {
                    ""name"": """",
                    ""id"": ""cc628607-7fbf-494e-8ba2-57e262d69ef6"",
                    ""path"": ""<Gamepad>/buttonEast"",
                    ""interactions"": """",
                    ""processors"": """",
                    ""groups"": ""Gamepad"",
                    ""action"": ""Dash"",
                    ""isComposite"": false,
                    ""isPartOfComposite"": false
                },
                {
                    ""name"": """",
                    ""id"": ""0479fa50-cde1-479a-8a77-779f0157cf5e"",
                    ""path"": ""<Gamepad>/rightTrigger"",
                    ""interactions"": """",
                    ""processors"": """",
                    ""groups"": ""Gamepad"",
                    ""action"": ""OpenMouse"",
                    ""isComposite"": false,
                    ""isPartOfComposite"": false
                }
            ]
        }
    ],
    ""controlSchemes"": [
        {
            ""name"": ""Gamepad"",
            ""bindingGroup"": ""Gamepad"",
            ""devices"": [
                {
                    ""devicePath"": ""<Gamepad>"",
                    ""isOptional"": false,
                    ""isOR"": false
                }
            ]
        }
    ]
}");
        // Player
        m_Player = asset.FindActionMap("Player", throwIfNotFound: true);
        m_Player_Move = m_Player.FindAction("Move", throwIfNotFound: true);
        m_Player_Dash = m_Player.FindAction("Dash", throwIfNotFound: true);
        m_Player_OpenMouse = m_Player.FindAction("OpenMouse", throwIfNotFound: true);
        
    }

    public void Dispose()
    {
        UnityEngine.Object.Destroy(asset);
    }

    public InputBinding? bindingMask
    {
        get => asset.bindingMask;
        set => asset.bindingMask = value;
    }

    public ReadOnlyArray<InputDevice>? devices
    {
        get => asset.devices;
        set => asset.devices = value;
    }

    public ReadOnlyArray<InputControlScheme> controlSchemes => asset.controlSchemes;

    public bool Contains(InputAction action)
    {
        return asset.Contains(action);
    }

    public IEnumerator<InputAction> GetEnumerator()
    {
        return asset.GetEnumerator();
    }

    IEnumerator IEnumerable.GetEnumerator()
    {
        return GetEnumerator();
    }

    public void Enable()
    {
        asset.Enable();
    }

    public void Disable()
    {
        asset.Disable();
    }

    public IEnumerable<InputBinding> bindings => asset.bindings;

    public InputAction FindAction(string actionNameOrId, bool throwIfNotFound = false)
    {
        return asset.FindAction(actionNameOrId, throwIfNotFound);
    }

    public int FindBinding(InputBinding bindingMask, out InputAction action)
    {
        return asset.FindBinding(bindingMask, out action);
    }

    // Player
    private readonly InputActionMap m_Player;
    private List<IPlayerActions> m_PlayerActionsCallbackInterfaces = new List<IPlayerActions>();
    private readonly InputAction m_Player_Move;
    private readonly InputAction m_Player_Dash;
    private readonly InputAction m_Player_OpenMouse;
    public struct PlayerActions
    {
        private @Player2 m_Wrapper;
        public PlayerActions(@Player2 wrapper) { m_Wrapper = wrapper; }
        public InputAction @Move => m_Wrapper.m_Player_Move;
        public InputAction @Dash => m_Wrapper.m_Player_Dash;
        public InputAction @OpenMouse => m_Wrapper.m_Player_OpenMouse;
        public InputActionMap Get() { return m_Wrapper.m_Player; }
        public void Enable() { Get().Enable(); }
        public void Disable() { Get().Disable(); }
        public bool enabled => Get().enabled;
        public static implicit operator InputActionMap(PlayerActions set) { return set.Get(); }
        public void AddCallbacks(IPlayerActions instance)
        {
            if (instance == null || m_Wrapper.m_PlayerActionsCallbackInterfaces.Contains(instance)) return;
            m_Wrapper.m_PlayerActionsCallbackInterfaces.Add(instance);
            @Move.started += instance.OnMove;
            @Move.performed += instance.OnMove;
            @Move.canceled += instance.OnMove;
            @Dash.started += instance.OnDash;
            @Dash.performed += instance.OnDash;
            @Dash.canceled += instance.OnDash;
            @OpenMouse.started += instance.OnOpenMouse;
            @OpenMouse.performed += instance.OnOpenMouse;
            @OpenMouse.canceled += instance.OnOpenMouse;
        }

        private void UnregisterCallbacks(IPlayerActions instance)
        {
            @Move.started -= instance.OnMove;
            @Move.performed -= instance.OnMove;
            @Move.canceled -= instance.OnMove;
            @Dash.started -= instance.OnDash;
            @Dash.performed -= instance.OnDash;
            @Dash.canceled -= instance.OnDash;
            @OpenMouse.started -= instance.OnOpenMouse;
            @OpenMouse.performed -= instance.OnOpenMouse;
            @OpenMouse.canceled -= instance.OnOpenMouse;
        }

        public void RemoveCallbacks(IPlayerActions instance)
        {
            if (m_Wrapper.m_PlayerActionsCallbackInterfaces.Remove(instance))
                UnregisterCallbacks(instance);
        }

        public void SetCallbacks(IPlayerActions instance)
        {
            foreach (var item in m_Wrapper.m_PlayerActionsCallbackInterfaces)
                UnregisterCallbacks(item);
            m_Wrapper.m_PlayerActionsCallbackInterfaces.Clear();
            AddCallbacks(instance);
        }
    }
    public PlayerActions @Player => new PlayerActions(this);

    
    private int m_KeyboardMouseSchemeIndex = -1;
    public InputControlScheme KeyboardMouseScheme
    {
        get
        {
            if (m_KeyboardMouseSchemeIndex == -1) m_KeyboardMouseSchemeIndex = asset.FindControlSchemeIndex("Keyboard&Mouse");
            return asset.controlSchemes[m_KeyboardMouseSchemeIndex];
        }
    }
    private int m_GamepadSchemeIndex = -1;
    public InputControlScheme GamepadScheme
    {
        get
        {
            if (m_GamepadSchemeIndex == -1) m_GamepadSchemeIndex = asset.FindControlSchemeIndex("Gamepad");
            return asset.controlSchemes[m_GamepadSchemeIndex];
        }
    }
    public interface IPlayerActions
    {
        void OnMove(InputAction.CallbackContext context);
        void OnDash(InputAction.CallbackContext context);
        void OnOpenMouse(InputAction.CallbackContext context);
    }
    
}
```

  </markdown>
</details>


#### Input Action资产读取
在声明该类的时候，脚本读取了保存为JSON的Input Action资产（因为将整个资产的JSON都生成在代码里了所以这部分可能会比较庞大，我在例子中只保留了玩家控制部分的内容），并储存控制的对应设置。

<details>
  <summary>读取保存为JSON的Input Action资产</summary>
  <markdown>

```csharp
asset = InputActionAsset.FromJson(@"{
    ""name"": ""player2"",      // Input Action的名字
    ""maps"": [                 // 资产里的控制设定
        {
            ""name"": ""Player"",
            ""id"": ""2b28ef6a-c40d-4bd2-bf6a-fe66aa27d755"",
            ""actions"": [
                {
                    ""name"": ""Move"",
                    ""type"": ""Value"",
                    ""id"": ""936e8d41-2517-4056-b264-8acb73f6efb5"",
                    ""expectedControlType"": ""Vector2"",
                    ""processors"": """",
                    ""interactions"": """",
                    ""initialStateCheck"": true
                },
                ......(冲刺、拾取)
            ],
            ""bindings"": [         // 控制设定绑定的键位
                {
                    ""name"": """",
                    ""id"": ""978bfe49-cc26-4a3d-ab7b-7d7a29327403"",
                    ""path"": ""<Gamepad>/leftStick"",
                    ""interactions"": """",
                    ""processors"": """",
                    ""groups"": ""Gamepad"",
                    ""action"": ""Move"",
                    ""isComposite"": false,
                    ""isPartOfComposite"": false
                },
                ......(为控制绑定的控制器)
            ]
        }
      ],
      ""controlSchemes"": [         // 控制器们
        {
            ""name"": ""Gamepad"",
            ""bindingGroup"": ""Gamepad"",
            ""devices"": [
                {
                    ""devicePath"": ""<Gamepad>"",
                    ""isOptional"": false,
                    ""isOR"": false
                }
            ]
        },
        ......(其他种类的控制器，如键鼠、摇杆、XR、触摸屏)
    ]
  }");
```

  </markdown>
</details>

找到设置并储存，找不到就抛出异常：

```csharp
// Player
m_Player = asset.FindActionMap("Player", throwIfNotFound: true);
m_Player_Move = m_Player.FindAction("Move", throwIfNotFound: true);
m_Player_Dash = m_Player.FindAction("Dash", throwIfNotFound: true);
m_Player_OpenMouse = m_Player.FindAction("OpenMouse", throwIfNotFound: true);
```

同时读取所有存在于（也可能为空，所以返回类型的后面有一个问号）Input Action资产中的绑定和设备。

```csharp
public InputBinding? bindingMask
{
    get => asset.bindingMask;
    set => asset.bindingMask = value;
}

public ReadOnlyArray<InputDevice>? devices
{
    get => asset.devices;
    set => asset.devices = value;
}
```

#### 调用资产

类里面提供了一些对外暴露的可调用的方法，例如启用/禁用InputAction。其中还有一个名为PlayerActions的结构体，里面提供了设置回调函数和移除回调函数的方法（但我没找着调用它们的地方（...

前面提到过新输入系统是基于事件的，系统实际上是在监听我们设置的那些操作，然后在监听到的时候执行回调函数处理传回的信息。每个操作都有以下五个状态：
- Canceled，操作取消
- Disabled，禁用操作
- Performed，操作执行中
- Started，开始操作
- Waiting，等待操作

在 Start、Performed 和 Canceled 阶段会分别触发三个对应的 C# 事件（事件类型为event Action<InputAction.CallbackContext>）

#### 声明回调函数

在使用到InputAction的脚本中（例如玩家控制器等），可以为不同的控制的行为附加回调函数。通常来说可以在OnEnable中统一订阅，在OnDisable中统一取消订阅

<img src="https://ae01.alicdn.com/kf/Sc2699a74f2fe4e2a9f6fdefaaad00c8bD.png" alt="OnEnable中的订阅行为" title="OnEnable中的订阅行为" />

以移动为例，Move函数长这样：

```csharp
/// <summary>
/// 移动
/// </summary>
/// <param name="context"></param>
private void Move(InputAction.CallbackContext context)
{
    moveVec = context.ReadValue<Vector2>();
}
```

context代表了事件传回的信息，譬如用手柄的左摇杆进行控制移动时，传回的就是摇杆的偏移向量（一个vec2）。当然我们这样只是拿到了摇杆的便宜向量，角色还没有真正动起来，我们只需要在Update（因为这次运动我用了物理，所以在FixedUpdate里）调用这个随时变化的moveVec即可：

```csharp
private void FixedUpdate()
{
    player.AddForce(moveVec * speed);
}
```

# 同屏双手柄（1p2p）

由于我们做的是一个比较欢脱的双人派对竞技游戏，这次jam还有一个重要的需求是同屏双手柄控制。我不想像懂哥一样搞个PlayerController1.cs和PlayerController2.cs，所以如何在一个脚本中区分两个玩家并为其绑定不同的控制器尤为重要。
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
