---
title: UnityUI程序设计与UI性能优化（三）：UGUI的渲染流程
tags: ['UI','UGUI','源码','Unity']
categories: UnityUI程序设计与UI性能优化
date: '2023-07-11T11:23:22.046Z'
updated: '2023-07-20T12:52:10.543Z'
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA96d03380d5b9586247c6962d0c4f1351.jpg
---

# UGUI的渲染过程

Unity中渲染的物体都是由**网格**(Mesh)构成的，而网格的绘制单元是**图元**(点、线、三角面)。在 Unity 中渲染一个 2D 或 3D 对象时，需要由 CPU 提起一个 `draw call`（绘制调用），经过一个完整的<a href="https://cattyhouse-guiny.xyz/2021/07/13/%C2%A71-1%EF%BC%9A%E6%B8%B2%E6%9F%93%E6%B5%81%E6%B0%B4%E7%BA%BF/">渲染管线流程</a>，经由 GPU 处理最后输出像素到屏幕上。那么对于 UGUI 元素（例如Image）来说是不是也是一样的呢？

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAc097eec523b5f6b5740979bd3eb56945.png" alt="和未来的同学们一起为GMTK 2023做的游戏的标题页面，其中渐变部分就是一张半透明的UI Image" width=456 />

绘制一个 UI，简而言之就是绘制 UI 的形状与图元：

1. **生成 mesh**：mesh 由顶点和三角形组成，顶点里包括 UV 坐标、顶点颜色。
2. **渲染图元**：将纹理渲染到 mesh 上。

我们将以绘制一张 Image 为例，学习 UGUI 的渲染过程。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA815cca8fdee931369f6554f005da7e24.png" alt="新建一个Image默认搭载的脚本" width=300 />

可以看到，新建一个 `Image` 对象，默认挂载了**Rect Transform**、**Canvas Renderer** 和 **Image** 三个组件。Image 的源码位于 Runtime/UI/Core/Image.cs，这个文件有 1000 多行，初见感觉挺吓人的，但其实大部分都是注释。Image 继承自 **MaskableGraphic**（同样继承自此抽象类的还有 `RawImage` 和 `Text`，顾名思义它们都是可被遮罩的），MaskableGraphic 继承自 **Graphic**，再往上就是所有 UI 元素都要继承的 **UIBehavior** 了。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAce1e88f42c973d369ae744f85c54b56f.png" alt="Image的继承信息" width=250 />

**Canvas Renderer** 是 UGUI 的渲染器，和用于渲染 2D sprite 的 **Sprite Renderer** 有所不同（渲染器的不同也是 **Image** 和 **2D Sprite** 之间的主要差异）。每个 Canvas 都挂载有 Canvas Renderer，因为 UGUI 是**以 Canvas 为单位进行合批渲染**的，换句话说，每个 Canvas 都会开始一个全新的 DrawCall。我们可以在检查器上找到 **Canvas Renderer**，但是没有展示任何属性。这个类提供了许多关键绘制信息，比如被渲染物体的颜色、材质和 Mesh 等，主要作用就是渲染包含在 Canvas 中的 UI 对象。

在渲染画布前，UGUI 会进行三个操作：**重建布局**（Rebuild Layout）、**重建画布图形**（Rebuild Canvas Graphics）和注册到 **CanvasUpdateRegistry**（一个注册处，可以获取指定 Canvas 所包含的 Graphic）上。

{% note pink 'fas fa-lightbulb' %}
**布局重建**（Layout Rebuild）：
当UI元素的尺寸、位置、或布局规则发生变化时，会触发布局重建。
例如改变 RectTransform 的尺寸或位置、修改布局组件（如Grid Layout、Vertical Layout Group）等。
{% endnote %}

{% note orange 'fas fa-lightbulb' %}
**绘制重建**（Graphic Rebuild）：
当UI元素的外观发生变化时，会触发绘制重建。
例如改变UI元素的颜色、纹理、或其他图形属性。
{% endnote %}

{% note green 'fas fa-lightbulb' %}
**CanvasUpdateRegistry**：
CanvasUpdateRegistry 中注册了当前一帧需要重新绘制的 Canvas，具体实现方式是在 `CanvasUpdateRegistry()` 方法中，为 `Canvas.willRenderCanvases` 添加 `PerformUpdate` 函数。
```C#
protected CanvasUpdateRegistry() {
    Canvas.willRenderCanvases += PerformUpdate;
}
```
{% endnote %}

由上文可知，**CanvasUpdateRegistry** 监听了 Canvas 的 `willRenderCanvases` 事件，该事件会**每帧调用**并执行 `PerformUpdate` 函数。这个 `PerformUpdate` 就是我们实现**布局重建**和**图元重绘**的方法了，除此之外它还实现了**裁剪**（Culling）的功能。

```C#
private void PerformUpdate() {
    UISystemProfilerApi.BeginSample(UISystemProfilerApi.SampleType.Layout);
    CleanInvalidItems();

    m_PerformingLayoutUpdate = true;

    m_LayoutRebuildQueue.Sort(s_SortLayoutFunction);
    // 布局重建，遍历 m_LayoutRebuildQueue 中的 UI 元素，执行重建
    for (int i = 0; i <= (int)CanvasUpdate.PostLayout; i++) {
        for (int j = 0; j < m_LayoutRebuildQueue.Count; j++) {
            var rebuild = instance.m_LayoutRebuildQueue[j];
            try {
                // 重建
                if (ObjectValidForUpdate(rebuild))
                    rebuild.Rebuild((CanvasUpdate)i);
            }
            catch (Exception e) {
                Debug.LogException(e, rebuild.transform);
            }
        }
    }
    // 布局重建完成
    for (int i = 0; i < m_LayoutRebuildQueue.Count; ++i)
        m_LayoutRebuildQueue[i].LayoutComplete();
    // 清空队列
    instance.m_LayoutRebuildQueue.Clear();
    m_PerformingLayoutUpdate = false;

    // 执行裁剪
    ClipperRegistry.instance.Cull();
    // 图元重建，遍历 m_GraphicRebuildQueue 中的 UI 元素，执行重建
    m_PerformingGraphicUpdate = true;
    for (var i = (int)CanvasUpdate.PreRender; i < (int)CanvasUpdate.MaxUpdateValue; i++) {
        for (var k = 0; k < instance.m_GraphicRebuildQueue.Count; k++) {
            try {
                // 重建
                var element = instance.m_GraphicRebuildQueue[k];
                if (ObjectValidForUpdate(element))
                    element.Rebuild((CanvasUpdate)i);
            }
            catch (Exception e) {
                Debug.LogException(e, instance.m_GraphicRebuildQueue[k].transform);
            }
        }
    }
    // 图元重建完成
    for (int i = 0; i < m_GraphicRebuildQueue.Count; ++i)
        m_GraphicRebuildQueue[i].GraphicUpdateComplete();

    instance.m_GraphicRebuildQueue.Clear();
    m_PerformingGraphicUpdate = false;
    UISystemProfilerApi.EndSample(UISystemProfilerApi.SampleType.Layout);
}
```

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA26cd71eb726cfeb7a6839ad4fc09e57d.png" />

- **布局重建**：当 UI 的布局元素发生改变时，设置脏数据，将其放入 **m_LayoutRebuildQueue** 队列中执行重建，调用元素的 `Rebuild` 方法。
- **裁剪**：即 `ClipperRegistry.instance.Cull()` 方法执行裁剪，只绘制布局未被遮挡的 UI 以优化性能。
- **图元重建**：类似布局重建，当 UI 的外观发生改变时，设置脏数据，将其放入 **m_GraphicRebuildQueue** 队列中执行重建，调用元素的 `Rebuild` 方法。

{% note warning %}
**注意**，一个 UI 元素若要重建，必须继承自 **ICanvasElement** 接口，因为执行重建操作的时候会调用该接口中的 `Rebuild` 函数。
{% endnote %}

来看看 `Rebuild` 函数吧：

```C#
public virtual void Rebuild(CanvasUpdate update) {
    if (canvasRenderer == null || canvasRenderer.cull)
        return;

    switch (update) {
        case CanvasUpdate.PreRender:    // 在渲染前（图形重建前）调用
            if (m_VertsDirty) {     // 如果顶点发生变化
                UpdateGeometry();   // 更新顶点
                m_VertsDirty = false;
            }
            if (m_MaterialDirty) {  // 如果材质发生变化
                UpdateMaterial();   // 更新材质
                m_MaterialDirty = false;
            }
            break;
    }
}
```

这个函数主要做两件事情：一是检查 **Canvas Renderer** 是否存在，二是在渲染前检查画布对象的**顶点**和**材质**是否发生了改变（包括改变颜色、大小、材质参数等），可以在其他地方通过调用**SetVerticesDirty** 和 **SetMaterialDirty** 将数据标记为“脏”来实现。如果发生变化，则进行**更新**。

对于 Image 来说，它也存在顶点（及其构成的**网格**）和材质，这是渲染它的关键要素。

## 顶点和网格
### Graphic.UpdateGeometry
在 `Rebuild` 中，如果顶点发生变化就会调用 `UpdateGeometry`，它负责将图形的几何数据（即顶点和网格数据）更新到 Canvas Renderer上：

```C#
protected virtual void UpdateGeometry() {
    if (useLegacyMeshGeneration) {  //Image 的构造函数已经将其设为 false 了
        DoLegacyMeshGeneration();
    } else {
        DoMeshGeneration(); // 所以调用它
    }
}
```

### Graphic.DoMeshGeneration
在 **DoMeshGeneration**里，如果绘制对象是合法的（RectTransform的长宽为正），就会调用 **OnPopulateMesh** 绘制所有顶点，把顶点和三角形信息保存到 `s_VertexHelper` 里。**VertexHelper** 储存了所有的绘制信息，除了顶点外还包括**法线**、**UV**、**颜色**、**切线**以及一些函数，下面是它的部分代码：

```C#
public class VertexHelper : IDisposable {
     private List<Vector3> m_Positions;
     private List<Color32> m_Colors;
     private List<Vector4> m_Uv0S;
     private List<Vector4> m_Uv1S;
     private List<Vector4> m_Uv2S;
     private List<Vector4> m_Uv3S;
     private List<Vector3> m_Normals;
     private List<Vector4> m_Tangents;
     private List<int> m_Indices;
​
     public void FillMesh(Mesh mesh) {
         InitializeListIfRequired();
​
         mesh.Clear();
​
         if (m_Positions.Count >= 65000)
             throw new ArgumentException("Mesh can not have more than 65000 vertices");
​
         mesh.SetVertices(m_Positions);
         mesh.SetColors(m_Colors);
         mesh.SetUVs(0, m_Uv0S);
         mesh.SetUVs(1, m_Uv1S);
         mesh.SetUVs(2, m_Uv2S);
         mesh.SetUVs(3, m_Uv3S);
         mesh.SetNormals(m_Normals);
         mesh.SetTangents(m_Tangents);
         mesh.SetTriangles(m_Indices, 0);
         mesh.RecalculateBounds();
    }
}
```

代码中的 `IMeshModifier` 是一个接口，所有**需要依据顶点信息的组件**（例如Shadow）都直接或间接继承自它。调用它们的 **ModifyMesh** 方法，修改网格信息，最后将 `s_VertexHelper` 里修改后的信息赋值给 `workerMesh`，并将 `workerMesh` 设置给**Canvas Renderer**。

```C#
private void DoMeshGeneration() {
    if (rectTransform != null && rectTransform.rect.width >= 0 && rectTransform.rect.height >= 0)
        OnPopulateMesh(s_VertexHelper); // 假装绘制了一个矩形Mesh
    else
        s_VertexHelper.Clear(); // 清除 vertex helper，以免绘制出无效图形
    // 获取 Mesh Modifier
    var components = ListPool<Component>.Get();
    GetComponents(typeof(IMeshModifier), components);

    for (var i = 0; i < components.Count; i++)
        ((IMeshModifier)components[i]).ModifyMesh(s_VertexHelper);

    ListPool<Component>.Release(components);
    // 生成网格
    s_VertexHelper.FillMesh(workerMesh);
    // 为渲染器设置网格
    canvasRenderer.SetMesh(workerMesh);
}
```

### Image.OnPopulateMesh
**OnPopulateMesh** 从名字上看好像生成了网格，其实只是根据 Image 类型的不同调用了不同的函数来绘制顶点，然后装入 vertex helper 中。Image 分为**四种类型**（Simple/Sliced/Tiled/Filled），可以看看<a href="https://zhuanlan.zhihu.com/p/398813288">这篇文章</a>的介绍，每种类型都要其生成顶点的方法，例如下图展示的是 Filled 类型的 Image 在 Radial360 模式下顶点和网格的变化。

![](mesh.gif)

一般来说默认创建的 Image 都是 Simple 类型的，也就是拉伸填充。这里的 `useSpriteMesh` 决定 vertex helper 是使用 **TextureImporter** 生成网格（生成的网格更贴合具有透明度的图形）还是使用简单的四边形网格显示 UI 图像。

> **注意**
注意：如果纹理导入器的 **SpriteMeshType** 属性设置为 **SpriteMeshType.FullRect**，那么 **GenerateSprite** 只会生成一个四边形（而不是预期中紧密贴合的网格）。因此，在启用此属性时，还必须确保纹理导入器的 **SpriteMeshType** 属性设置为 **Tight**。

```C#
protected override void OnPopulateMesh(VertexHelper toFill) {
    // 如果UI元素没有显示的图形（也就是默认的白色方形）
    if (activeSprite == null) {
        base.OnPopulateMesh(toFill);
        return;
    }
    // 根据Image的类型不同调用不同的生成函数
    switch (type) {
        case Type.Simple:   // Image的默认模式，即拉伸填充
            if (!useSpriteMesh)
                GenerateSimpleSprite(toFill, m_PreserveAspect);
            else
                GenerateSprite(toFill, m_PreserveAspect);
            break;
        case Type.Sliced:   // 九宫格图，应用于需要拉伸 UI（例如框）
            GenerateSlicedSprite(toFill);
            break;
        case Type.Tiled:    // 重复平铺
            GenerateTiledSprite(toFill);
            break;
        case Type.Filled:   //只显示图片的一部分，可以用作技能冷却、计时器效果等
            GenerateFilledSprite(toFill, m_PreserveAspect);
            break;
    }
}
```

### Image.GenerateSimpleSprite
**GenerateSimpleSprite** 的内部实现非常简单，首先拿到绘制的尺寸 `v`，也就是四个顶点的位置，然后根据 `activeSprite` （Image显示的图像）拿到纹理的UV值。接着通过传入的 vertex helper 创建了四个顶点，并把它们连接成两个三角形。

```C#
void GenerateSimpleSprite(VertexHelper vh, bool lPreserveAspect) {
    Vector4 v = GetDrawingDimensions(lPreserveAspect);
    var uv = (activeSprite != null) ? Sprites.DataUtility.GetOuterUV(activeSprite) : Vector4.zero;

    var color32 = color;
    vh.Clear();
    // 创建四个顶点，顶点索引(Index)依次为0，1，2，3
    // 第三个参数是纹理坐标
    vh.AddVert(new Vector3(v.x, v.y), color32, new Vector2(uv.x, uv.y)); // 左下
    vh.AddVert(new Vector3(v.x, v.w), color32, new Vector2(uv.x, uv.w)); // 左上
    vh.AddVert(new Vector3(v.z, v.w), color32, new Vector2(uv.z, uv.w)); // 右上
    vh.AddVert(new Vector3(v.z, v.y), color32, new Vector2(uv.z, uv.y)); // 右下
    // 连接顶点成三角形
    vh.AddTriangle(0, 1, 2);
    vh.AddTriangle(2, 3, 0);
}
```

如果打开线框模式，我们就会发现默认创建的 Image 线框是下图这样的。UI 元素一般是按四边形来渲染, 上面代码中的两个三角形正好。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAf88635760c58d04a7789090075ee3105.png" alt="默认情况下生成的mesh" width=300 />

### Image.GenerateSprite
这个就要相对复杂一些了，具体的实现过程不展开谈。GenerateSprite可以生成贴合sprite的网格，但对于有较多圆形的sprite来说，效果可能很差。例如下图是我之前帮一个朋友做背景时画的八卦元素图（放大看边框上有很多小圆圈）：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAa1329154ab1bbafc3e94a1c43c655cb5.png" width=100/>

把它添加到 Image 上，设置 `useSpriteMesh` 为 true 后，生成的网格结果如下所示：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA39fdad7e1abf43fa97e70fec6c03d9a3.png" alt="开启useSpriteMesh之后，生成了非常多mesh" width=250 />
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAf2fd1691645099d55f8b232008849629.png" />

中间三条矩形横杠的地方网格贴合的效果很好，但除开这 12 个顶点，外圈的环和小圆圈居然生成了 3877 个顶点和 11670 个顶点索引（面数*3），这是非常吓人的。所以如果想开启 `useSpriteMesh` ，最好不要让 UI 图像太精致，或者有太多圆。

## 材质

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA88e15cbc0f4cdb78d38466d8d86d9d83.png" width=300 />

**Graphic.Rebuild** 中的 **UpdateMaterial** 函数被 Image 覆写了，所以调用的是如下代码：

```C#
/* base长这样：
protected virtual void UpdateMaterial() {
    if (!IsActive())
        return;

    canvasRenderer.materialCount = 1;
    canvasRenderer.SetMaterial(materialForRendering, 0);
    canvasRenderer.SetTexture(mainTexture);
}
*/
protected override void UpdateMaterial() {
    base.UpdateMaterial();

    // 检查sprite是否有相关的 alpha 纹理（将 RGBA = RGB + A 分割为两个不带 alpha 的纹理时生成）

    if (activeSprite == null) {
        canvasRenderer.SetAlphaTexture(null);
        return;
    }

    Texture2D alphaTex = activeSprite.associatedAlphaSplitTexture;

    if (alphaTex != null) {
        canvasRenderer.SetAlphaTexture(alphaTex);
    }
}
```

在 Inspector 窗口设置的材质（如下图）有最高优先级, 其次是默认的 defaultMaterial。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAce6fd203ab83f4031e78d0f68cd42ae3.png" />

## 纹理

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA73a2ec34112d425dbbdab472ca5cb521.png" width=250 />

我们可以在 **Graphic.UpdateMaterial** 中找到这么一行：

```C#
canvasRenderer.SetTexture(mainTexture);
```

如果直接查看 Graphic 里的 mainTexture，会发现是 **S_WhiteTexture**，也就是我们新建一个 UI Image 时展示出来的白色方块。不过 mainTexture 被 Image 覆写了：

```C#
public override Texture mainTexture {
    get {
        if (activeSprite == null) {
            if (material != null && material.mainTexture != null) {
                // 返回材质的main texture
                return material.mainTexture;
            }
            // 返回默认白色方块
            return s_WhiteTexture;
        }
        // 返回当前Image展示的图像
        return activeSprite.texture;
    }
}
```

这里的逻辑关系就很清楚了：优先使用 **activeSprite** ，如果不存在就看材质也没有 **mainTexture**，这个也没有就直接返回默认白色方块。 **activeSprite** 就是 Image 所显示的图像，之所以存在这个与 **sprite**（也就是检查器添加的source image）不同的概念，是因为一些特殊的 UI 场景需要 sprite 切换（例如按钮）。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAd843c35d5604639e7b0240280b9ed360.png" />

```C#
private Sprite activeSprite { get { return m_OverrideSprite != null ? m_OverrideSprite : sprite; } }
```

---

纹理、顶点、网格、材质，这些信息都有了之后，**CanvasRenderer** 就会负责 Image (也包括所有 Graphic 的子类) 的渲染了。

Image 在颜色变化、变更类型(Simple、Sliced、Tiled、Filled)、变更是否应保留Sprite宽高比(Image.preserveAspect属性的变更)，FillCenter属性变更(是否渲染平铺或切片图像的中心)、变更填充方式(Horizontal、Vertical、Radial360....)、变更图像填充率(fillAmount)、变更图像顺逆时针填充类型(Image.fillClockwise)、变更填充过程的原点(Image.FillOrigin)时都会被添加进重建队列并调用 `Rebuild` 重建并渲染。

> 在项目中需要尽量减少这些情况的出现，因为重建是消耗资源的。我们可以用一些取巧的方法，例如用改变UI的Scale(1->0)来代替改变UI的Enable属性、以Image的Scale代替Slider来进行百分比展示等。

对于上述可能引发重建的操作，可以采取以下优化措施：

1. 避免频繁更新：尽量减少对 RectTransform、布局组件和图形属性的频繁更新。
2. 分离动态和静态 UI 元素：将频繁更新的动态UI元素与静态UI元素分开，避免不必要的重建。
3. 使用额外的 Canvas：将频繁更新的 UI 元素放在单独的 Canvas 上，这样可以避免影响到其他不需要重建的UI元素。


# 参考资料
https://www.young40.com/post/2021-12-26-ugui-source-reading-01/
https://jonyzhao.gitbooks.io/gamedev/content/Unity/UGUI/UGUIRenderSystem.html
https://discussions.unity.com/t/whats-canvasupdate/548584/4
https://zhuanlan.zhihu.com/p/398813288
https://zhuanlan.zhihu.com/p/448293298
https://zhuanlan.zhihu.com/p/5051331939