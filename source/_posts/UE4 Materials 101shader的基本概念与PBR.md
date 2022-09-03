---
title: UE4 Materials 101：shader的基本概念与PBR
created: '2022-01-14'
tags: ['Unreal','技术美术','PBR','shader']
categories: UE4-Materials101
cover: https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220115191313.png
---

# Shader可以做什么？
shader即着色器，可以用来做很多工作。比如说材质表面的处理（颜色、反射、发现、金属度、粗糙度）、光照处理（漫反射、反射光、阴影、环境光遮罩、环境反射、环境光）等等。
在基于物理的渲染中，光照部分由引擎计算完成，shader的主要工作在用处理表面的情况。在Unreal中，shader是通过类似Unity中的shader graph来实现的（可视化节点编程），底层的shader代码由引擎生成。
## Unreal中的shader
在初创项目中，材质被统一放进了materials文件夹，如下：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220114160920.png"/>

在双击打开材质球之后，就会进入虚幻4的材质编辑界面。可以看到，材质球是由多个节点块连接组成的：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220114161018.png"/>

### 开始创建你的第一个shader！
创建新材质的方法很简单，右键找到材质即可。需要注意的是虚幻4对材质命名格式的要求为“M_材质名”，请尽量遵守命名规范，这对整体的格式规范有利。至此，我们的第一个材质球创建好了：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220114161424.png"/>

双击打开，发现默认情况下只有一个节点（并且是基于物理的）：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220114161602.png" width=150/>

选中一个节点之后，可以在左侧查看节点的具体信息（比如赋值情况、贴图等等），右侧则是可以创建的所有节点。创建节点更好的方式是直接在节点界面空白处右键输出想要的卡片创建。

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220114163427.png"/>

之后，我们就可以根据需要将卡片相连接，得到所需要的效果：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220114163944.png"/>

### 材质函数
当我们需要对多个材质调用相同的代码块时，可以材质函数让事情变得更简单。我们可以在主菜单的文件面板中右键选择materials&texture，找到materials function创建你的第一个材质函数！

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220120150559.png" width=350/>

在编辑器内创建想要的函数，记得使用FunctionInput节点和FunctionOutput节点对输入输出进行定义：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220120152219.png"/>

我们还可以根据希望的输入输出顺序对节点进行排序：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220120155053.png"/>


# 什么是PBR
PBR即基于物理的渲染，在前面的文章中我们已经提到过了。PBR是经验光照模型之上的一大飞跃，从传统的近似、模拟光照进化到以真实世界的物理为依据的光照渲染。PBR主要有两方面的属性：**光照属性**和**表面属性**。
> **光照属性**
>- 直接漫反射
>- 间接漫反射
>- 直接镜面反射
>- 间接镜面反射
>- 阴影
>- 环境光遮罩

> **表面属性**
>- 基础颜色
>- 法线
>- 镜面反射
>- 粗糙度
>- 金属度

## 光照属性
光照分为直接光照和间接光照两种。其中，直接光照指的是光从光源出发，直接照射到材质表面的过程。而间接光照指的是光照射到其他材质表面之后，再反射照射到材质表面的过程。
而当光照射到物体表面之后，也存在散射和吸收两种过程，往下还可以继续细分：

<center><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211121002346.png" width=500/></center>

在对光的计算中，可以拆分出直接光照的漫反射（计算最简单）、镜面反射以及间接光照的漫反射（计算最复杂）、镜面反射。
- 间接光照的漫反射光需要模拟无数光线在场景中反射碰撞得到的结果，计算量非常庞大，所以一般是靠离线渲染将结果储存在光照贴图中方便之后使用的。对于复杂场景的离线渲染的时间可能长达几小时。
- 间接光照的高光反射可以由反射探针、光线追踪（最昂贵，但效果最好）等方式实现，计算量同样比较大。

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220114171216.png" width=300/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220114171241.png" width=300/>
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220114171342.png" width=300/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220114171758.png" width=300/>

## 表面属性
表面属性涉及到几张贴图：albedo(base)、normal、specular、roughness、metallic。
- base color表示了物体的漫反射颜色。但需要注意的是，不能将颜色调的过暗或过亮（RGB颜色模型大于240或小于20），因为当物体这么暗的时候，几乎没有反射光射出；而物体太亮的时候，又几乎是全白的，这是非常不真实的效果，在真实世界中不存在（绝对黑体还没被造出来呢，那只是理想情况）。
并且需要注意的是，当一个物体较为粗糙时，base color的颜色应该控制在20~50之间。当颜色太亮，粗糙的程度会被覆盖，展示出来的效果不够真实
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220115153300.png"/>

- normal即法线，展示了材质表面的法线扰动情况（切线空间的法线）。法线贴图的颜色并不代表真实的颜色，而是对向量进行了编码再以颜色的形式储存到图片中的。
法线不能手绘，通常，我是用Photoshop的3D滤镜来生成法线贴图的。

- specular指的是物体的表面反射的情况、能够反射多少光。specular的值越大，物体的反射程度越高，越接近金属。一般来说，非金属的反射率为4%（specular为0.5）。当specular为1时代表8%的反射率，通常用于水晶等非常镜面的物体。
当直视一个物体的时候，视线中心对应的反射率即这个物体的通常反射率。而当达到视线掠角（视线与边缘相切呈现90°夹角）的时候，反射率达到100%，这就是菲涅尔效应。对于真实世界的任何物体来说，菲涅尔反射都存在，只是有强弱之分。镜面反射使用了镜面反射贴图，越白表示反射率越高，越黑反射率越低，默认为0.5。
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220115153226.png"/>
> 裂缝贴图(Crevice Map)乘以0.5可以得到很棒的镜面反射贴图。而！！我没用查到这个是什么
视频中作者的做法似乎是用高度图的R通道实现的

- roughness表示了物体表面的粗糙程度。当值为1（白色）时越粗糙，值为0（黑色）则非常镜面。粗糙度贴图控制的是反射的聚焦程度，当粗糙度越大时，反射越模糊；粗糙度越小时，反射越清晰、强烈。
粗糙度贴图可以根据高度图来制作，并且可以随意的调整不同区域的光滑/粗糙程度。
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220115153153.png"/>

- metallic则反映了物体的金属度程度。对于真正的金属来说，漫反射是黑色的，而base color则被镜面反射贴图取代，也就是说镜面反射的区域是整个物体。
当金属度为1时，镜面反射贴图的输入会被忽略。因为非金属的反射率通常在0% ~ 8%，对应了镜面反射贴图的（0，1）.而金属的镜面反射通常可以达到60% ~ 100%，显然镜面反射贴图是不适用的。
所以对于完全非金属物体来说，金属度贴图是全黑的。当金属度贴图有部分白色的时候，白色的部分会进入金属工作流（下下图）。由于现实世界中的物质很少有介于金属与非金属之间的，所以金属度贴图一般是非黑即白的，需要尽量避免灰阶出现。
对于纯金属来说，需要注意base color的选择
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220115153559.png"/>
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220115173252.png"/>
> 部分金属的base color选择
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220115173836.png"/>

### 在UE4中的应用
当我们准备完PBR所需的几张贴图之后，可以直接用textureSample节点展开每张贴图，并连接到相应的节点上：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220115174753.png"/>

其中最后一张贴图是AO（环境光遮蔽），在本视频中没有详细介绍





