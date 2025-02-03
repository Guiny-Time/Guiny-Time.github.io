---
title: UnityUI程序设计与UI性能优化（四）：UGUI的点击事件
date: 2023-07-15 16:41:18
updated: 
tags: ['UI','UGUI','源码','Unity']
categories: 引擎功能
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAa198651e59fd8df47130a8f8b489dbc5.png
---

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA6b6935c66455dcdd22208dca3f17bef8.png" />

在 Unity 的 UI 系统中，Button 是最常见的**可交互组件**之一（除此之外还有 Slider、Toggle、Input Field、Scroll Bar、Drop Down），其点击事件不仅是开发者构建用户交互的基础，更是一个复杂而精巧的功能模块。本文将从 UGUI 的源码层面出发，全面剖析 Button 点击事件的实现原理，理解其设计思路与内部逻辑。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA3184b2a76d5f8e46a76d47ea3f1462be.png" alt="初始创建的Button(TMP)" width=300/>

# 按钮点击的触发流程
相信每个介绍 UGUI 的新手视频都会讲到 Button 的点击事件。在 Unity 中，如果我们想为一个按钮添加事件有**两种实现方式**，一种是直接在检查器上为该按钮绑定事件:

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA1759456b91beba83619ddd34f6534b3d.png" alt="Button组件上的OnClick" width=352 />

另一种是通过代码为按钮的 `onClick` 加上方法监听:

```C#
GetComponent<Button>().onClick.AddListener(() => { 
    ...//按钮点击的处理逻辑
});
```

这两者是等价的。让我们沿着 **onClick** 的调用链一步一步看看是怎么回事吧。

## Button：Press -> OnPointerClick
我们来看看 Button 的源码吧！在 UGUI 的源码中，Button 是一个继承自 **Selectable** 的类。Selectable 提供了 UI 组件的基础**选择功能**，如焦点状态、交互状态等。Button 在 Selectable 的基础上扩展了点击事件处理能力（`onClick` 方法）。

```C#
using System;
using System.Collections;
using UnityEngine.Events;
using UnityEngine.EventSystems;
using UnityEngine.Serialization;

namespace UnityEngine.UI {
    /// <summary>
    /// 当点击时发送事件的标准按钮
    /// </summary>
    [AddComponentMenu("UI/Button", 30)]
    public class Button : Selectable, IPointerClickHandler, ISubmitHandler {
        [Serializable]
        /// 按钮点击事件
        public class ButtonClickedEvent : UnityEvent {}

        // 点击时触发的事件委托
        [FormerlySerializedAs("onClick")]
        [SerializeField]
        private ButtonClickedEvent m_OnClick = new ButtonClickedEvent();

        protected Button(){}

        // 也就是我们前文中提到的 onClick
        public ButtonClickedEvent onClick {
            get { return m_OnClick; }
            set { m_OnClick = value; }
        }

        // 如果按钮处于活跃状态并且可交互(Interactable设置为true)，则触发事件
        private void Press() {
            if (!IsActive() || !IsInteractable())
                return;
            UISystemProfilerApi.AddMarker("Button.onClick", this);
            // 执行onClick委托上的方法
            m_OnClick.Invoke();
        }

        // 鼠标点击按钮时调用该函数，继承自 IPointerClickHandler 接口
        // 此操作需要确保您的场景有一个**事件系统**（EventSystem）
        public virtual void OnPointerClick(PointerEventData eventData) {
            if (eventData.button != PointerEventData.InputButton.Left)
                return;
            // 触发点击
            Press();
        }

        /// 按钮提交时调用该函数，继承自 ISubmitHandler 接口
        // "提交"键可以在 Edit->Project Settings->Input->Submit 中自定义
        public virtual void OnSubmit(BaseEventData eventData) {
            Press();
            // 如果点击过程中被设置为禁用，就不要运行该协程
            if (!IsActive() || !IsInteractable())
                return;

            DoStateTransition(SelectionState.Pressed, false);
            StartCoroutine(OnFinishSubmit());
        }

        private IEnumerator OnFinishSubmit() {
            var fadeTime = colors.fadeDuration;
            var elapsedTime = 0f;

            while (elapsedTime < fadeTime) {
                elapsedTime += Time.unscaledDeltaTime;
                yield return null;
            }
            // 状态转换，包括动画/颜色切换等
            DoStateTransition(currentSelectionState, false);
        }
    }
}
```

我们发现，鼠标点击时最终调用的是 `Press` 方法，里面执行了 **onClick 委托链**上挂载的方法。`Press` 被包裹进 `OnPointerClick` 方法里，而这个 `OnPointerClick` 是 `IPointerClickHandler` 接口下的方法实现（这个接口也仅包括该方法）：

```C#
public interface IPointerClickHandler : IEventSystemHandler {
    // 使用此回调检测点击事件
    void OnPointerClick(PointerEventData eventData);
}
```

## ExecuteEvents：Execute -> s_PointerClickHandler -> pointerClickHandler
查找 `OnPointerClick` 的调用链，我们会发现该方法是由 **ExecuteEvents** 类下的 `Execute` 方法调用的。**ExecuteEvents 类**相当于**事件执行器**，提供了许多通用的事件处理方法，针对按钮点击类型的 `Execute` 只是其中的一种**重载**。`Execute` 方法被赋值给 **s_PointerClickHandler** 字段，该字段由 `pointerClickHandler` 方法封装提供。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA80ee9af3d47796a41509459b385813cd.png" alt="OnPointerClick的调用" />

```C#
// Execute方法赋值给s_PointerClickHandler字段
private static readonly EventFunction<IPointerClickHandler> s_PointerClickHandler = Execute;

private static void Execute(IPointerClickHandler handler, BaseEventData eventData) {
    handler.OnPointerClick(ValidateEventData<PointerEventData>(eventData));
}

public static EventFunction<IPointerClickHandler> pointerClickHandler {
    get { return s_PointerClickHandler; }
}
```

## BaseInput：ReleaseMouse(Execute) -> ProcessMousePress -> ProcessMouseEvent -> Process
沿着调用链继续往上，我们会找到两个脚本： **TouchInputModule** 和 **StandaloneInputModule**。这两个类都继承自 **BaseInput**，主要作用是**输入处理**：

- **TouchInputModule**：专为触摸屏设备（如：手机）设计，主要用于处理移动设备上的触摸输入，它支持单点触摸、多点触摸等功能。不支持鼠标和键盘
- **StandaloneInputModule** 是一个通用的输入模块，能够支持多种输入方式，包括鼠标、键盘和触摸屏输入。

假设我们的项目是一个电脑游戏，那么我们就会用鼠标去点击这个 Button，也就对应了 **StandaloneInputModule** 中的下面两个方法：

```C#
// 计算和处理任何鼠标按钮状态的变化
// Process函数间接对其进行调用（调用链过长，不一一展示)
protected void ProcessMousePress(MouseButtonEventData data) {
    ...// 省略部分代码
    var pointerEvent = data.buttonData;
    var currentOverGo = pointerEvent.pointerCurrentRaycast.gameObject;
    // 鼠标按键抬起时调用（按键包括鼠标左键、中间滑轮和右键）
    if (data.ReleasedThisFrame()) {
        ReleaseMouse(pointerEvent, currentOverGo);
    }
    ...
}
​
// 满足松开鼠标的条件时调用
// currentOverGo ：当前选中的游戏物体
private void ReleaseMouse(PointerEventData pointerEvent, GameObject currentOverGo) {
    // pointUp事件
    ExecuteEvents.Execute(pointerEvent.pointerPress, pointerEvent, ExecuteEvents.pointerUpHandler);

    var pointerClickHandler = ExecuteEvents.GetEventHandler<IPointerClickHandler>(currentOverGo);

    if (pointerEvent.pointerPress == pointerUpHandler && pointerEvent.eligibleForClick) {
        // pointClick事件，传入ExecuteEvents.pointerClickHandler委托
        ExecuteEvents.Execute(pointerEvent.pointerPress, pointerEvent, ExecuteEvents.pointerClickHandler);
    }  
    ...// 省略部分代码
}
```

追寻 `ProcessMousePress` 方法的调用链，最后来到了输入模块的 `Process` 方法这里。该方法在 **EventSystem** 的 `Update` 里逐帧检查事件是否被触发。`Process` 方法会处理鼠标的按下、抬起等事件，当鼠标抬起时调用 `ReleaseMouse` 方法，并最终调用 `Execute` 方法并触发 **IPointerClick** 事件。

所以兜兜转转一圈，居然又回到了 **ExecuteEvents** 的 `Execute` 方法上。不过这里调用的 `Execute` 和前面出现的并不相同，而是：

```C#
// **target**：需要执行事件的游戏对象
public static bool Execute<T>(GameObject target, BaseEventData eventData, EventFunction<T> functor) where T : IEventSystemHandler {
    var internalHandlers = ListPool<IEventSystemHandler>.Get();
    // 获取target对象的事件
    GetEventList<T>(target, internalHandlers);

    var internalHandlersCount = internalHandlers.Count;
    for (var i = 0; i < internalHandlersCount; i++) {
        T arg;
        try {
            arg = (T)internalHandlers[i];
        } catch (Exception e) {
            var temp = internalHandlers[i];
            Debug.LogException(new Exception(string.Format("Type {0} expected {1} received.", typeof(T).Name, temp.GetType().Name), e));
            continue;
        }

        try {
            // 执行EventFunction<T>委托,例如pointerClickHandler(arg,eventData)
            functor(arg, eventData);
        } catch (Exception e) {
            Debug.LogException(e);
        }
    }

    var handlerCount = internalHandlers.Count;
    ListPool<IEventSystemHandler>.Release(internalHandlers);
    return handlerCount > 0;
}
```

# UGUI的事件系统
在刚才梳理 Button 点击响应的过程中，我们看到了不少新概念，相信大家也产生了一些疑问，比如：最终调用的 `Execute` 方法的 **target** 参数是如何通过 **EventData** 拿到对应 UI 元素的值的？在更深入理解 Button 的点击事件之前，我们需要先简单了解一下 UGUI 的**事件系统**。以下是事件系统的文件目录：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA0411c3886a4b627ce147669890c074c4.png" />

UGUI 的事件系统是一个模块化的、基于事件驱动的交互框架，从目录结构中可以看到，**EventSystem** 的核心由以下几部分组成：

1. **EventSystem**：事件分发中心，负责**管理**用户输入与事件分发，一个场景只能包含一个 EventSystem。它主要包含以下功能：
    - 管理哪个游戏对象被认为是选中的
    - 管理正在使用的输入模块
    - 管理射线检测（如果需要）
    - 根据需要更新所有输入模块
2. **InputModules**：输入模块，包含 **StandaloneInputModule** 和 **TouchInputModule** 两种具体实现，负责**处理**不同平台的输入行为。输入模块的主要任务有三个，分别是：
    - 处理输入
    - 管理事件状态
    - 发送事件到场景对象
3. **Raycasters**：射线检测模块，将输入位置映射到 UI 元素上，检测当前输入事件需要发送到哪里。系统提供了以下几种类型的 Raycaster:
    - **Graphic Raycaster**：检测 UI 元素
    - **PanelRaycaster**：检测 UI Toolkit的面板实例
    - **Physics 2D Raycaster**：用于 2D 物理元素
    - **Physics Raycaster**：用于 3D 物理元素

这些模块共同协作，为 UGUI 提供了灵活的交互能力。

## EventSystem
EventSystem 与多个模块协同工作，主要负责**保持（保存）状态**并将功能委托给特定的组件。

当事件系统启动时，它会搜索连接到同一游戏对象的任何 **BaseInputModules**，并将它们添加到内部列表中。更新时，每个附加模块都会收到 **UpdateModules** 调用，模块可以在此修改内部状态。每个模块更新后，活动模块将执行 **Process** 调用。

### 管理输入模块
EventSystem 的源码采用了 **BaseInputModule** 类型的 List 和变量保存输入模块：

```C#
// 系统输入模块
private List<BaseInputModule> m_SystemInputModules = new List<BaseInputModule>();
// 当前输入模块
private BaseInputModule m_CurrentInputModule;
```

在 **BaseInputModule** 的 `OnEnable` 和 `OnDisable` 中，脚本会查找场景中所有的输入模块并赋值给 **m_SystemInputModules** 字段。

``` C#
public void UpdateModules() {
    // 获取所有输入模块
    GetComponents(m_SystemInputModules);
    var systemInputModulesCount = m_SystemInputModules.Count;
    // 移除非活跃状态的输入模块
    for (int i = systemInputModulesCount - 1; i >= 0; i--) {
        if (m_SystemInputModules[i] && m_SystemInputModules[i].IsActive())
            continue;

        m_SystemInputModules.RemoveAt(i);
    }
}
```

接下来，EventSystem 会在 `TickModules` 方法中逐个更新每一个输入模块。

``` C#
private void TickModules() {
    var systemInputModulesCount = m_SystemInputModules.Count;
    // 更新输入模块
    for (var i = 0; i < systemInputModulesCount; i++) {
        if (m_SystemInputModules[i] != null)
            m_SystemInputModules[i].UpdateModule();
    }
}
```

`UpdateModule` 方法用于**更新输入模块的状态**，主要目的是处理触摸事件（对 StandaloneInputModule 的覆写则涵盖了鼠标事件等）的状态变化，确保窗口失去焦点时不会留下未处理的拖拽或输入状态。

```C#
public override void UpdateModule() {
    if (!eventSystem.isFocused && ShouldIgnoreEventsOnNoFocus()) {
        if (m_InputPointerEvent != null && m_InputPointerEvent.pointerDrag != null && m_InputPointerEvent.dragging) {
            ReleaseMouse(m_InputPointerEvent, m_InputPointerEvent.pointerCurrentRaycast.gameObject);
        }

        m_InputPointerEvent = null;

        return;
    }
    // 记录鼠标的当前位置与上一次位置
    m_LastMousePosition = m_MousePosition;
    m_MousePosition = input.mousePosition;
}
```

EventSystem 对输入模块主要的管理在 `Update` 生命周期函数里，通过前文介绍过的 `TickModules` 方法更新输入模块，并 在满足条件的情况下调用当前模块的 `Process` 方法：

``` C#
protected virtual void Update() { 
    TickModules();
​
    // 遍历 m_SystemInputModules
    bool changedModule = false;
    var systemInputModulesCount = m_SystemInputModules.Count;
    // 遍历 m_SystemInputModules
    for (var i = 0; i < systemInputModulesCount; i++) {
        var module = m_SystemInputModules[i];
        // 判断输入模块是否支持当前平台
        if (module.IsModuleSupported() && module.ShouldActivateModule()) {
            // 如果支持并且可以激活，则将其赋值给当前输入模块并break
            if (m_CurrentInputModule != module) {
                ChangeEventModule(module);
                changedModule = true;
            }
            break;
        }
    }
​
    // 如果上面没找到符合条件的模块，则使用第一个支持当前平台的模块
    if (m_CurrentInputModule == null) {
        for (var i = 0; i < systemInputModulesCount; i++) {
            var module = m_SystemInputModules[i];
            if (module.IsModuleSupported()) {
                ChangeEventModule(module);
                changedModule = true;
                break;
            }
        }
    }
​
    // 如果当前模块没有发生变化并且当前模块不为空
    if (!changedModule && m_CurrentInputModule != null)
        m_CurrentInputModule.Process();
}
```

以 StandaloneInputModule 为例，其中的 `Process` 覆写长这样：

``` C#
public override void Process() {
    // 如果当前窗口未聚焦且配置为忽略无焦点事件，不处理事件
    if (!eventSystem.isFocused && ShouldIgnoreEventsOnNoFocus())
        return;
    // 更新选中对象
    bool usedEvent = SendUpdateEventToSelectedObject();

    // 由于鼠标模拟层的存在，需要优先处理触摸事
    if (!ProcessTouchEvents() && input.mousePresent)
        ProcessMouseEvent();
    // 如果启用了导航事件，会处理移动事件和提交事件
    if (eventSystem.sendNavigationEvents) {
        if (!usedEvent)
            usedEvent |= SendMoveEventToSelectedObject();

        if (!usedEvent)
            SendSubmitEventToSelectedObject();
    }
}

protected bool SendUpdateEventToSelectedObject() {
    // 未选中任何对象
    if (eventSystem.currentSelectedGameObject == null)
        return false;

    var data = GetBaseEventData();
    // updateSelected 事件
    ExecuteEvents.Execute(eventSystem.currentSelectedGameObject, data, ExecuteEvents.updateSelectedHandler);
    return data.used;
}
```

这里的 `Process` 方法就是我们在先前在 Button 点击事件调用链上找到的 `Process`，主要作用是**捕获各种输入事件**（如点击、拖拽等），通过 `ExecuteEvents.Execute` 方法执行 **updateSelected 事件**，更新 **EventSystem** 中当前选中的 **GameObject**(即 **m_CurrentSelected**)。有个这个“选中对象”，我们就知道触发事件的对象具体是哪个 UI 元素了，那么 EventSystem 是如何管理选中的游戏对象的呢？

### 管理选中的游戏对象
EventSystem 是通过一个用于储存当前选中对象的字段 **m_CurrentSelected** 来管理选中物体的。当场景中的可交互 UI 元素（例如 Button、Dropdown、InputField 等）被选中时，会通知之前选中的对象执行**被取消**(**OnDeselect**)事件，通知当前选中的对象执行**选中**(**OnSelect**)事件，部分代码如下：

``` C#
private GameObject m_CurrentSelected;
public void SetSelectedGameObject(GameObject selected, BaseEventData pointer) {
    ......//省略部分代码
    // 通知之前被选中取消选中
    ExecuteEvents.Execute(m_CurrentSelected, pointer, ExecuteEvents.deselectHandler);
    m_CurrentSelected = selected;
    // 通知当前物体被选中
    ExecuteEvents.Execute(m_CurrentSelected, pointer, ExecuteEvents.selectHandler);
    m_SelectionGuard = false;
}
```

### 管理射线检测
那么问题来了，我们怎么知道一个 UI 元素被选中了呢？这就是**射线**（Raycast）的作用了。**EventSystem** 中有一个非常重要的函数 `RaycastAll`，主要作用就是获取目标。它被 **PointerInputModule** 类调用，即当鼠标设备可用或触摸板被使用时被调用。

``` C#
public void RaycastAll(PointerEventData eventData, List<RaycastResult> raycastResults) {
    // 清空结果
    raycastResults.Clear();
    // 获取BaseRaycast对象
    var modules = RaycasterManager.GetRaycasters();
    var modulesCount = modules.Count;
    for (int i = 0; i < modulesCount; ++i) {
        var module = modules[i];
        if (module == null || !module.IsActive())
            continue;
        // 调用Raycast方法，
        module.Raycast(eventData, raycastResults);
    }
​    // 结果排序
    raycastResults.Sort(s_RaycastComparer);
}
```

它首先获取所有的 **BaseRaycast** 对象，然后调用它们的 `Raycast` 方法，用以获取屏幕某个点下的所有目标，最后对得到的结果进行排序。大部分情况下排序都是根据**深度**(Depth)进行排序的，在一些情况下也会使用**距离**(Distance)、**排序顺序**（SortingOrder，如果是UI元素则是根据Canvas面板的 Sort order 值，3D 物体默认是 0）或者**排序层级**(Sorting Layer)等作为排序依据。排序过后，**raycastResults** 中最前的目标就被认为成射线击中的对象。

总结一下，**EventSystem** 会在 `Update` 中调用输入模块的 `Process` 方法来处理输入消息，**PointerInputModule** 会调用 **EventSystem** 中的 `RaycastAll` 方法进行射线检测，`RaycastAll` 又会调用 **BastRaycaster** 的 `Raycast` 方法执行具体的射线检测操作，主要是获取被选中的目标信息。

## InputModules
在讲 Button 的时候我们提到鼠标的点击事件是在 **BaseInputModule** 中触发的，除此之外，**EventInterface** 接口中的其他事件（也就是我们通过 **Event Trigger** 能为对象添加的所有事件类型）也都是由输入模块产生的。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAce4fb7f15f44ee908354db64f18f3d6d.png" alt="EventTrigger里的事件类型" width=400 />

各个不同的事件的具体触发条件如下：

- **pointerEnterHandler**、**pointerExitHandler**：当鼠标或触摸进入、退出当前对象时执行。
- **pointerDownHandler、pointerUpHandler**：在鼠标或者触摸按下、松开时执行。
- **pointerClickHandler**：在鼠标或触摸松开并且与按下时是同一个响应物体时执行。
- **beginDragHandler**：在鼠标或触摸位置发生偏移（偏移值大于一个很小的常量）时执行。
- **initializePotentialDrag**：在鼠标或者触摸按下且当前对象可以响应拖拽事件时执行。
- **dragHandler**：对象正在被拖拽且鼠标或触摸移动时执行。
- **endDragHandler**：对象正在被拖拽且鼠标或触摸松开时执行。
- **dropHandler**：鼠标或触摸松开且对象未响应 **pointerClickHandler** 情况下，如果对象正在被拖拽则执行。
- **scrollHandler**：当鼠标滚动差值大于 **0** 执行。
- **updateSelectedHandler**：当输入模块切换到 **StandaloneInputModule** 时执行。（不需要Input类）
- **selectHandler**、**deselectHandler**：当鼠标移动导致被选中的对象改变时，执行。
- **导航事件**：导航可用的情况下，
    - 按上下左右键执行 **moveHandler**
    - 按确认键执行 **submitHandler**
    - 按取消键执行 **cancelHandler**。

更加底层的调用还是UnityEngine.Input类，但可惜的是这部分Unity并没有开源。

{% note warning %}
每次事件系统中只能有**一个**输入模块处于活跃状态，并且必须与 **EventSystem** 组件处于相同的游戏对象上。
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA9f97b74ec565416932aa2eb4d6e056a1.png" width=300 />
{% endnote %}

### 执行事件
**InputModule** 可以处理设备输入，然后发送事件到场景对象，那这些事件是怎么执行的呢？在讲 Button 的时候，我们提到过 **ExecuteEvent** 类，其实事件的执行都是通过这个类进行的，不过也需要 **EventInterface** 接口配合。

**InputModule** 类中定义了许多接口，比如鼠标按下、点击、拖拽等。**ExecuteEvent** 类中提供了一个方法让外部统一调用以执行事件，也就是前面提到的泛型 `Execute` 方法，主要就是查找 **target** 对象上的 **T** 类型的组件列表，并遍历执行。

除此之外，还有一个 **GetEventHandler** 方法，它主要是通过**冒泡**的方式查找到能够处理指定事件的对象。冒泡是什么意思？比如我们在场景中创建了一个 Button，这个 Button 还包含了一个 Text 组件，当鼠标点击到按钮上的文本时就会调用 `GetEventHandler` 函数。该函数的 root 参数其实是 Text，但是会通过冒泡的方式查找到它的父物体 Button，然后调用 Button 的点击事件。

``` C#
// 在游戏对象上冒泡指定的事件，找出哪个对象将实际接收事件。 
public static GameObject GetEventHandler<T>(GameObject root) where T : IEventSystemHandler {
    if (root == null)
        return null;
​
    Transform t = root.transform;
    // 冒泡查找，如果物体本身不能处理输入的事件，交予parent处理
    while (t != null) {
        if (CanHandleEvent<T>(t.gameObject))
            return t.gameObject;
        t = t.parent;
    }
    
    return null;
}
​
// 指定的游戏对象是否能够处理指定的事件
public static bool CanHandleEvent<T>(GameObject go) where T : IEventSystemHandler {
     var internalHandlers = s_HandlerListPool.Get();
     GetEventList<T>(go, internalHandlers);
     var handlerCount = internalHandlers.Count;
     s_HandlerListPool.Release(internalHandlers);
     return handlerCount != 0;
 }
```

## Raycasters
**BaseRaycaster** 是其他 **Raycaster** 的基类。在它的 `OnEnable` 里将自己注册到 **RaycasterManager**，并在 `OnDisable` 的时候从 **RaycasterManager** 中移除。这个 **RaycasterManager** 是一个静态类，维护了一个 **BaseRaycaster** 类型的 List，功能比较简单，包含**获取**(Get)、**添加**(Add)、**移除**(Remove)方法。

**BaseRaycaster** 中最重要的就是 `Raycast` 方法了。对于 UI 元素来说，**BaseRaycaster** 有两个子类：**PanelRaycaster** 和 **GraphicRaycaster**，它们都对该方法进行了重写，以实现对应的射线方法。我们来分析一下 **GraphicRaycaster** 吧。

### GraphicRaycast
**GraphicRaycast** 用于检测 UI 元素，它依赖于 Canvas，我们在场景中添加 Canvas 默认都会包含一个 **GraphicRaycast** 组件：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA301b768b9ece4cb0c2775c2753aeb2d5.png" alt="i创建Canvas时自动创建的GraphicRaycast" />

**GraphicRaycast** 的 `Raycast` 方法先获取**鼠标坐标**，将其转换为 **Camera** 的**视角坐标**，然后分情况计算**射线的距离**（hitDistance），调用 **Graphic** 的 `Raycast` 方法来获取鼠标点下方的元素，最后将满足条件的结果添加到 **resultAppendList** 中。

``` C#
// 针对与 canvas 相关联的图形列表执行射线投射。
public override void Raycast(PointerEventData eventData, List<RaycastResult> resultAppendList) {
    if (canvas == null)
        return;
​
    // 返回Canvas上的所有包含Graphic脚本并且允许被射线检测的对象
    var canvasGraphics = GraphicRegistry.GetRaycastableGraphicsForCanvas(canvas);
​
    if (canvasGraphics == null || canvasGraphics.Count == 0)
        return;
​
    int displayIndex;
    // 画布在 ScreenSpaceOverlay 模式下默认为 null
    var currentEventCamera = eventCamera;
​
    if (canvas.renderMode == RenderMode.ScreenSpaceOverlay || currentEventCamera == null)
        displayIndex = canvas.targetDisplay;
    else
        displayIndex = currentEventCamera.targetDisplay;
​
    // 获取鼠标位置
    var eventPosition = Display.RelativeMouseAt(eventData.position);
    // 多 display 相关
    if (eventPosition != Vector3.zero) {
        // 支持多 display 和基于事件位置的显示识别
        int eventDisplayIndex = (int)eventPosition.z;
        // 丢弃不属于该 display 的事件，这样用户就不会同时与多个 display 交互。
        if (eventDisplayIndex != displayIndex)
            return;
    } else {
        // 并非所有平台都支持多display，当不支持时返回的位置将全为 0
        // 为了安全起见，当返回的索引为 0 时，我们将默认为事件数据。
        eventPosition = eventData.position;
    }
​
    // 将鼠标在屏幕上的坐标转换成摄像机的视角坐标
    Vector2 pos;
    if (currentEventCamera == null) {
        float w = Screen.width;
        float h = Screen.height;
        if (displayIndex > 0 && displayIndex < Display.displays.Length) {
            w = Display.displays[displayIndex].systemWidth;
            h = Display.displays[displayIndex].systemHeight;
        }
        pos = new Vector2(eventPosition.x / w, eventPosition.y / h);
    } else
        pos = currentEventCamera.ScreenToViewportPoint(eventPosition);
​
    // 如果超出相机范围则return
    if (pos.x < 0f || pos.x > 1f || pos.y < 0f || pos.y > 1f)
        return;
​
    float hitDistance = float.MaxValue;
​
    Ray ray = new Ray();
​
    // 如果 currentEventCamera 不为空，摄像机发射射线
    if (currentEventCamera != null)
        ray = currentEventCamera.ScreenPointToRay(eventPosition);
​
    if (canvas.renderMode != RenderMode.ScreenSpaceOverlay && blockingObjects != BlockingObjects.None) {
        float distanceToClipPlane = 100.0f;
        if (currentEventCamera != null) {
            float projectionDirection = ray.direction.z;
            distanceToClipPlane = Mathf.Approximately(0.0f, projectionDirection)
                ? Mathf.Infinity
                : Mathf.Abs((currentEventCamera.farClipPlane - currentEventCamera.nearClipPlane) / projectionDirection);
        }
        // 计算 hitDistance 的值
        #if PACKAGE_PHYSICS
            if (blockingObjects == BlockingObjects.ThreeD || blockingObjects == BlockingObjects.All) {
                if (ReflectionMethodsCache.Singleton.raycast3D != null) {
                    var hits = ReflectionMethodsCache.Singleton.raycast3DAll(ray, distanceToClipPlane, (int)m_BlockingMask);
                    if (hits.Length > 0)
                        hitDistance = hits[0].distance;
                }
            }
        #endif
            #if PACKAGE_PHYSICS2D
            if (blockingObjects == BlockingObjects.TwoD || blockingObjects == BlockingObjects.All) {
                if (ReflectionMethodsCache.Singleton.raycast2D != null) {
                    var hits = ReflectionMethodsCache.Singleton.getRayIntersectionAll(ray, distanceToClipPlane, (int)m_BlockingMask);
                    if (hits.Length > 0)
                        hitDistance = hits[0].distance;
                }
            }
        #endif
    }
​
    m_RaycastResults.Clear();
​
    // 调用 Raycast 函数重载，该方法位于 GraphicRaycaster 第326行
    // 不同于该 Raycast（提供给EventSystem使用），这个 Raycast 是内部调用的
    Raycast(canvas, currentEventCamera, eventPosition, canvasGraphics, m_RaycastResults);
​
    //遍历 m_RaycastResults
    int totalCount = m_RaycastResults.Count;
    for (var index = 0; index < totalCount; index++) {
        var go = m_RaycastResults[index].gameObject;
        bool appendGraphic = true;
​        // 判断 Graphic 的方向向量和 Camera 的方向向量是否相交
        if (ignoreReversedGraphics) {
            if (currentEventCamera == null) {
                // 如果没有摄像机，则始终面向前方
                var dir = go.transform.rotation * Vector3.forward;
                appendGraphic = Vector3.Dot(Vector3.forward, dir) > 0;
            } else {
                // 如果有摄像机，则面向摄像机的前方
                var cameraForward = currentEventCamera.transform.rotation * Vector3.forward * currentEventCamera.nearClipPlane;
                appendGraphic = Vector3.Dot(go.transform.position - currentEventCamera.transform.position - cameraForward, go.transform.forward) >= 0;
            }
        }
​        // 判断 Graphic 是否在 Camera 的前面，并且距离小于等于 hitDistance
        if (appendGraphic) {
            float distance = 0;
            Transform trans = go.transform;
            Vector3 transForward = trans.forward;
​
            if (currentEventCamera == null || canvas.renderMode == RenderMode.ScreenSpaceOverlay)
                distance = 0;
            else {
                distance = (Vector3.Dot(transForward, trans.position - ray.origin) / Vector3.Dot(transForward, ray.direction));
​
                // 检查对象是否在摄像机后面
                if (distance < 0)
                    continue;
            }
​            // 超出射线检测距离，忽视
            if (distance >= hitDistance)
                continue;
​            // 打包成 RaycastResult 添加到 resultAppendList 里
            var castResult = new RaycastResult {
                ......
            };
            resultAppendList.Add(castResult);
        }
    }
}
```

这个提供给 **EventSystem** 使用的 `Raycast` 方法里调用了 `Raycast` 的重载，该方法位于 **GraphicRaycaster** 的第326行，其作用是向屏幕投射射线并收集屏幕下方所有挂载了 Graphic 脚本的游戏对象，将结果储存到 **m_RaycastResults** 字段中。该重载方法的内容为：

``` C#
// 向屏幕进行射线投射，收集屏幕下方的所有图形
[NonSerialized] static readonly List<Graphic> s_SortedGraphics = new List<Graphic>();

private static void Raycast(Canvas canvas, Camera eventCamera, Vector2 pointerPosition, IList<Graphic> foundGraphics, List<Graphic> results) {
    // 遍历场景内 Graphic 对象(挂载了 Graphic 脚本的对象)
    int totalCount = foundGraphics.Count;
    for (int i = 0; i < totalCount; ++i) {
        Graphic graphic = foundGraphics[i];
​
        // -1 表示 canvas 尚未对其进行处理，也就是没有实际绘制
        if (!graphic.raycastTarget || graphic.canvasRenderer.cull || graphic.depth == -1)
            continue;
​
        // 目标点是否在矩阵中
        if (!RectTransformUtility.RectangleContainsScreenPoint(graphic.rectTransform, pointerPosition, eventCamera, graphic.raycastPadding))
            continue;
​
        // 是否超出摄像机范围
        if (eventCamera != null && eventCamera.WorldToScreenPoint(graphic.rectTransform.position).z > eventCamera.farClipPlane)
            continue;
​
        // 这里又有一个 Raycast 方法，是位于 Graphic 下的
        if (graphic.Raycast(pointerPosition, eventCamera)) {
            s_SortedGraphics.Add(graphic);
        }
    }
​    // 排序
    s_SortedGraphics.Sort((g1, g2) => g2.depth.CompareTo(g1.depth));
    totalCount = s_SortedGraphics.Count;
    for (int i = 0; i < totalCount; ++i)
        results.Add(s_SortedGraphics[i]);
​
    s_SortedGraphics.Clear();
}
```

从上面的源码中可以发现，这个 `Raycast` 方法中又套了一层 `Raycast` 方法（已经套了三层了），该方法位于 **Graphic** 类（RawImage、Image 和 Text 都间接继承自 Graphic）下。这个 `Raycast` 向场景中进行光线投射时，它主要会做两件事：

- 使用 **RectTransform** 的值过滤元素
- 使用 **Raycast** 函数确定射线击中的元素

``` C#
// sp：screen point
public virtual bool Raycast(Vector2 sp, Camera eventCamera) {
    if (!isActiveAndEnabled)
        return false;
​
    // UI 元素,比如 Image，Button 等
    var t = transform;
​
    var components = ListPool<Component>.Get();
​
    bool ignoreParentGroups = false;
    bool continueTraversal = true;
​
    while (t != null) {
        t.GetComponents(components);
        for (var i = 0; i < components.Count; i++) {
            var canvas = components[i] as Canvas;
            if (canvas != null && canvas.overrideSorting)
                continueTraversal = false;
​
            // 获取 ICanvasRaycastFilter 组件(Image，Mask，RectMask2D)
            var filter = components[i] as ICanvasRaycastFilter;
​
            if (filter == null)
                continue;
​
            var raycastValid = true;
​
            // 判断点是否在有效的范围内
            var group = components[i] as CanvasGroup;
            if (group != null) {
                if (ignoreParentGroups == false && group.ignoreParentGroups) {
                    ignoreParentGroups = true;
                    raycastValid = filter.IsRaycastLocationValid(sp, eventCamera);
                } else if (!ignoreParentGroups)
                    raycastValid = filter.IsRaycastLocationValid(sp, eventCamera);
            } else {
                raycastValid = filter.IsRaycastLocationValid(sp, eventCamera);
            }
​
            if (!raycastValid) {
                ListPool<Component>.Release(components);
                return false;
            }
        }
        // 遍历它的父物体
        t = continueTraversal ? t.parent : null;
    }
    ListPool<Component>.Release(components);
    return true;
}
```

源码中有一个频繁出现的方法 `IsRaycastLocationValid`。该方法位于 **ICanvasRaycastFilter** 接口，用来判断测试点（sp）是否有效。如果无效则会返回 **false**，图形不会被加入前面的 **s_SortedGraphics** 列表中。

`IsRaycastLocationValid` 方法的实现很简单：

``` C#
public virtual bool IsRaycastLocationValid(Vector2 screenPoint, Camera eventCamera) {
    // 小于阈值(alphaHitTestMinimumThreshold)的 Alpha 值将导致射线事件穿透图像。 
    // alphaHitTestMinimumThreshold 为 1 将导致只有完全不透明的像素在图像上注册相应射线事件。
    if (alphaHitTestMinimumThreshold <= 0)
        return true;
​
    if (alphaHitTestMinimumThreshold > 1)
        return false;
​
    if (activeSprite == null)
        return true;
​
    Vector2 local;
    if (!RectTransformUtility.ScreenPointToLocalPointInRectangle(rectTransform, screenPoint, eventCamera, out local))
        return false;
​
    Rect rect = GetPixelAdjustedRect();
​
    // 转换为以左下角为参考点
    local.x += rectTransform.pivot.x * rect.width;
    local.y += rectTransform.pivot.y * rect.height;
​
    local = MapCoordinate(local, rect);
​
    // 将本地坐标转换为纹理空间
    Rect spriteRect = activeSprite.textureRect;
    float x = (spriteRect.x + local.x) / activeSprite.texture.width;
    float y = (spriteRect.y + local.y) / activeSprite.texture.height;
​
    try {
        return activeSprite.texture.GetPixelBilinear(x, y).a >= alphaHitTestMinimumThreshold;
    } catch (UnityException e) {
        Debug.LogError("Using alphaHitTestMinimumThreshold greater than 0 on Image whose sprite texture cannot be read. " + e.Message + " Also make sure to disable sprite packing for this sprite.", this);
        return true;
    }
}
```

# 再看点击事件
最后，我们再来回顾一下 **Button** 的点击事件是怎么触发的。首先是 **EventSystem** 在 `Update` 中调用当前输入模块的 `Process` 方法处理所有的鼠标事件，并且输入模块会调用 `RaycastAll` 来得到目标信息，通过冒泡的方式找到事件实际接收者并执行点击事件。

1. 用户输入捕获：
EventSystem 检测到鼠标点击或触摸操作，将输入数据封装为 PointerEventData 对象。

2. 射线检测：
GraphicRaycaster 遍历场景中的 UI 元素，根据输入位置确定被点击的对象。

3. 事件分发：
EventSystem 调用目标对象上实现的接口方法，如 IPointerClickHandler.OnPointerClick。

4. 事件回调：
Button 的 OnPointerClick 方法被调用，进而触发 onClick 事件。

## 自定义 Button 行为
在实际开发中，Button 的默认功能可能无法满足需求。通过继承 Button 类，我们可以扩展其行为，例如：

1. **添加双击功能**
```C#
public class DoubleClickButton : Button {
    private float lastClickTime;
    private const float doubleClickThreshold = 0.3f;

    public UnityEvent onDoubleClick;

    public override void OnPointerClick(PointerEventData eventData) {
        base.OnPointerClick(eventData);

        if (Time.time - lastClickTime < doubleClickThreshold) {
            onDoubleClick.Invoke();
        }
        lastClickTime = Time.time;
    }
}
```

2. **改变点击区域**

通过重写射线检测逻辑，可以自定义 Button 的点击区域：
```C#
public class CustomHitButton : Button {
    public override bool IsRaycastLocationValid(Vector2 sp, Camera eventCamera) {
        // 自定义点击区域逻辑
        return RectTransformUtility.RectangleContainsScreenPoint(rectTransform, sp, eventCamera);
    }
}
```

## 性能优化

在复杂 UI 场景中，大量 Button 可能导致性能问题。以下是一些优化建议：

1. 减少不必要的事件监听：如果 Button 的状态变化不需要动画，可以禁用 Animator。

2. 合并 UI 元素：使用 Canvas 的批处理功能减少绘制调用。

3. 合理规划事件回调：避免在 onClick 中执行复杂逻辑，尽量将耗时操作放在后台线程。

# 参考资料
https://zhuanlan.zhihu.com/p/437704772
https://kendevlog.wordpress.com/2019/05/16/unity-%E6%8A%80%E5%B7%A7-%E7%95%8C%E9%9D%A2%E5%A4%A7%E8%AE%8A%E8%BA%AB-%E5%9F%BA%E7%A4%8E%E7%AF%87/