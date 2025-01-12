---
title: UnityUI程序设计与UI性能优化（五）：UGUI性能优化
tags: ['UI','UGUI','优化','Unity']
categories: 引擎功能
date: '2023-07-20T11:23:22.046Z'
updated: '2023-07-22T12:52:10.543Z'
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA1fc5ff9e9083059d3dab93de58d27b40.png
---

UGUI是一个很强大的系统，通过它我们可以拼出多姿多彩的UI界面，让玩家更好的与游戏UI进行交互。但作为开发者，我们也需要注意UGUI中潜在的性能优化问题，防止游戏因为 drawcall 过多、频繁 rebatch / rebuild 而造成发热和卡顿（特别是对于低端机型来说）。

那么，该怎么优化呢？我们优化的主要目的就是减少绘制调用（drawcall）。绘制调用发生在 CPU 与 GPU 中间，当 CPU 打包好渲染数据（例如顶点、纹理、材质等）之后，就会发起一次绘制调用，将这些信息打包到显存以通知 GPU 进行绘制。也就是说，有多少个绘制调用，CPU 就需要向 GPU 发送多少次通知。drawcall 太高，意味着有频繁地改变渲染状态，这是很慢的操作（相对于GPU执行其内部的渲染流水线来说），因此**绘制调用的次数越多对性能影响越大**。

# 动静分离
Canvas有两个重要的操作：重合批和重建。所有的 UI 元素都需要挂载在 canvas 下，而当一个 canvas 中包含的 mesh（只需要一个）发生改变时（例如SetActive、transform的改变、颜色改变、文本内容改变等等）就会**弄脏**整个 canvas。此时 canvas 会重新生成网格，并向 GPU 发出绘制调用（drawcall），以便实际显示 UI。这个过程需要分析整个 canvas，因此代价很高，比如在一个包含数千个元素的单个 Canvas 上改变一个元素时，游戏可能会经历耗时**数毫秒**的 CPU 峰值。

> **什么是弄脏**？
我们在上一篇文章中介绍了重建，其中当几何数据/材质数据发生变化时就会重新构建，而检测是否发生了改变靠的就是 `m_VertsDirty` 和 `m_MaterialDirty` 两个布尔值。UGUI 在改变发生时调用 `SetDirty` 方法标记脏数据，弄脏的意思就是数据产生了变化（因此需要重新渲染）。

但我们有时不可避免地会改动到 UI 元素，该怎么办呢？注意，UGUI 是基于 canvas 来进行合并计算的，上文中提到的这些变化影响的范围是以 **canvas** **为界限**的，即**其他** **canvas**、**子** **canvas** 内的变化都与**当前** **canvas** **无关**。

{% note success %}
所以我们可以**分割** **canvas**，用多个 canvas 或子 canvas（嵌套canvas）将频繁变动的 UI 元素与其他元素分割开，就可以减少 rebatch 和 rebuild 。这个优化思想就是**动静分离**，将静态 UI 元素保存在单独的画布上，并将同时更新的动态元素保存在较小的子画布上。

另外，确保每个画布上的所有 UI 元素具有相同的 Z 值、材质和纹理，原因会在下文中聊到。
{% endnote %}

# 使用图集（Atlas）
Unity 通常会为场景中的每个纹理发出一个绘制调用。多个 UI 元素使用相同的纹理只会触发一次绘制调用，但多个 UI 元素使用不同的纹理就会触发多次了。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA1e20a615f927ff8d883ab836095c5ab7.png" alt="多个 UI 元素使用相同的纹理" />

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAa734381af96b3bc2216e18abf7e22206.png" alt="但多个 UI 元素使用不同的纹理" />

图集的作用就是可以**把多张小图打包成一张大图**。右键点击 Create -> SpriteAtlas 就能生成一个图集，将想要合并的图片（或者放置图片的文件夹）拉进图集的 Objects for Packing 就能将图片自动打成图集，如下图。

> 如果找不到，可能是 Project Settings -> Editor -> Sprite Packer -> Mode 没有选择为 Enable。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA5cce33c83a1ff4b473b8c2e933a6c897.png" width=350 />


{% note success %}
所以我们可以将多张纹理打包到一个图集，这样大家都用的一个图集，就相当于用的同一个纹理，只会触发一次dc了。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA465db69964346f04dab2d4ff804db640.png" width=400/>

图集的管理选择以单个 UI 面板来管理图集，而不是使用文件夹形式。
{% endnote %}

> 另外注意：因为对象是按**层**渲染出来的，若有如果 *文本* 和 *图像* **有所重叠**的话，也会增加绘制调用的次数。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAa66466d4fe4702e51f409aff2ad2ecb3.png" />
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAf6ed6d0cab29f118a21629a2746273f0.png" alt="文本置顶"/>
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAa91d89b8ed416fc2ab28cb3c426c9b6e.png" />
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA40ff8fedd1c27b6e1f8edb5b672f85e4.png" alt="文本插在图片中间" />

文本插在两个重叠的同材质同图集的 UI Image 中间，绘制调用比文本直接置顶多了一个，这是因为不同材质的对象（在这里是文本）插入层中就会打断合批，因此 Image 和 Image(1)、Image(2) 不算一个层了，需要单独发起一个绘制调用。

<img src="batch.gif" alt="渲染顺序" width=300 />

***

Unity 的合批规则是进行**重叠检测**，然后**分层合并**。

- 第一步计算每个UI元素的**层级号**：
    - 如果有一个 UI 元素，它所占的矩形范围内没有任何 UI 在它的底下，那么它的层级号就是**0**（最底）；
    - 如果有一个 UI 在其底下且该 UI 可以和它合批（材质相同），那它的层级号与底下的 UI 层级一样；
    - 如果有一个 UI 在其底下但是无法与它合批，那它的层级号为底下的 UI 的层级 + 1；
    - 如果有多个 UI 都在其下面，那么按前两种方式遍历计算所有的层级号，其中最大的那个作为自己的层级号。
2. 第二步合并相同层级中可以合批的元素作为一个批次，并对批次进行排序：有了层级号之后，Unity会将每一层的所有元素进行一个排序（按照材质、纹理等信息），合并掉可以Batch的元素成为一个批次。经过以上排序，就可以得到一个有序的批次序列了。这时Unity会再做一个优化，即如果相邻间的两个批次正好可以Batch的话就会进行Batch。合批的Batch数据，最后会分别放在CanvasMesh的SubMesh里。

# 限制Graphic Raycaster并禁用Raycast Target
**Graphic Raycaster**用于对画布进行射线投射，查看画布上的所有图形并确定是否有任何图形被射线击中。它是将玩家输入转换为 UI 事件的组件，允许将屏幕**点击**或屏幕**触摸输入**转换为 UI 事件，然后将它们发送给感兴趣的 UI 元素。按钮、滑条等需要交互的 UI 元素的 canvas （包括子 canvas）必须挂载 Graphic Raycaster，否则事件将无法生效。

Graphic Raycaster会循环遍历屏幕上的每个输入点并检查它们是否位于 UI 元素的 RectTransform 内（以便触发事件回调）。当画布中存在大量不需要交互的 UI 元素（例如通常情况下的 Text 和 Image）时，这样的遍历将会是一个开销。

{% note success %}
所以我们可以从非交互式的 UI canvas中移除 Graphic Raycaster 并关闭静态或非交互式元素的Raycast Target。
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAfb8d749932d9320fcc916e732fbdf64e.png" width=400 />
{% endnote %}

# 避免使用布局（Layout）

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAe4a010b52970168dbda508cb1d5fa71c.png" />

Layout Group 是一个很方便的功能，它能帮助自动布局，自动调整一个或多个的元素大小、位置、间距等。这个功能很高级，但在实际项目中却不那么常见原因就在于当 Layout Group 上的一个或多个子 UI 元素发生变化时，布局就会变“脏”。改变的子元素使拥有它的 LayoutGroup 无效。

每个将其 Layout Group 标记为“脏”的 UI 元素将至少执行一次 `GetComponent` 调用。此调用会在当前 LayoutGroup 的父级上查找有效的 LayoutGroup。如果找到，它会继续沿着 Transform 层次结构向上走，直到停止寻找 LayoutGroup 或到达层次结构**根**。因此，每个 LayoutGroup 都会向每个子 LayoutElement 的弄脏过程添加一个 `GetComponent` 调用，这使得嵌套 LayoutGroup 的性能极其糟糕。

不仅如此，在 `Rebuild` 时 **Layout components** 需要排序，这也是很耗时的操作。

{% note success %}
所以我们不建议使用 Layout Group，建议使用锚点进行比例布局或自己用代码计算布局。
{% endnote %}

# 正确的UI对象池
开发者在实现对象池时，可能是如下步骤：

1. 将需要入池的 UI 元素父对象设置成池子对象
2. 隐藏 UI 元素（SetActive(false)），实现入池

然而这样的操作会带来开销。修改父节点会弄脏一次层次结构，隐藏（在上一篇文章中提到过）则会触发第二次，导致两次重绘。

{% note success %}
所以正确的入池操作应该是：
1. 隐藏 UI 元素
2. 将需要入池的 UI 元素父对象设置成池子对象

这样只会在隐藏时弄脏一次数据。同样的，如果需要将对象取出，先修改父对象再启用它。
{% endnote %}

# 正确的隐藏UI元素
如果用 setactive（false）去隐藏 UI 元素，会导致所在 canvas 的**顶点缓冲区**（VBO）数据失效。所以再次 setactive（true）的时候，会导致整个 canvas 去 rebuild 和 rebatch，从而对 CPU 造成负担。我们其实在上一篇文章中提到过如何避免重绘，其中提到可以通过改变 UI 的 Scale (1->0) 来代替改变 UI 元素的 Enable 属性。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA39fc9c2c93333d370d5dea90fa862da4.png" />

除此之外，直接禁用 **Canvas 组件**将阻止 Canvas 向 GPU 发出绘制调用。这样整个 canvas 就不可见了，但它不会丢弃其**顶点缓冲区**，canvas 会保留其所有网格和顶点，当重新启用时不会触发重建，只会再次开始绘制它们。或者我们也可以通过下面的代码：

```C#
CanvasRenderer.enabled = false
```

禁用 Canvas 组件也不会通过 Canvas 层次结构触发昂贵的 OnDisable/OnEnable 回调，是隐藏 UI 元素时的一个好选项，缺点就是影响范围太大了。

# 全遮挡UI
全屏UI的性能并不好，因为虽然全屏UI挡住了整个游戏，但其实游戏的其余部分（包括被该canvas遮挡的其他canvas）仍在后台渲染，这会造成不必要的开销。

{% note success %}
为了避免额外开销，应该：
- 禁用相机渲染3D场景（可以关闭主相机，只保留UI相机）
- 隐藏其他 Canvas，或为被遮挡的 canvas 添加 **CanvasGroup** 组件，然后在被挡住时将其 alpha 值设为 `0`
- 通过 `Application.targetFrameRate` 降低目标帧率，因为不需要以 60 fps 的速度更新 UI
{% endnote %}

# 参考资料
https://unity.com/cn/how-to/unity-ui-optimization-tips
https://docs.unity3d.com/Packages/com.unity.ugui@1.0/manual/script-GraphicRaycaster.html
https://docs.unity3d.com/cn/2021.3/Manual/script-LayoutElement.html
https://blog.csdn.net/wang980502/article/details/127173427
https://zhuanlan.zhihu.com/p/364785849
https://xinzhuzi.github.io/2020/05/11/Unity/UI/Unity_UGUI/
https://xinzhuzi.github.io/2020/05/08/Unity/UI/Unity_UI%E4%BC%98%E5%8C%96/