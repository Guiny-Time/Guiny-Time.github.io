---
tags: ['Unreal']
title: UE4 Materials 101：凹凸偏移与视差闭塞映射
created: '2022-01-16'
categories: UE4-Materials101
cover: https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220117020757.png
---

这个概念其实在以前的文章里介绍过了（UnityShader2-2：纹理映射与贴图），这里讲一下对应贴图在UE4中的实战应用。
# Bump Offset
Bump Offset使用的是高度图。高度图是一种灰阶图，其中白色代表高度较高，黑色代表高度较低。
在UE4中，我们可以把高度图与Bump Offset节点连接，再连接到贴图的uv接口使用。需要注意的是，使用高度图的同时也要使用PBR流程中的其他几张贴图，如下图所示。
凹凸偏移并不会改变顶点的位置，但可以营造出一种“视角变化时产生遮挡”的效果，使凹凸效果更加逼真。
> **为什么高度图一般是模糊的？**
&emsp;&emsp;其实原因很简单，我们不希望在视角变化的过程中出现“断口”（如下图所示），所以要尽量模糊物体表面的高低变化。
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220117013722.png"/>

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220117013142.png"/>

# Parallax Occlusion Mapping
视差闭塞映射比凹凸偏移要昂贵很多，因为在节点内部进行了多次纹理采样，同时还包含了光线追踪（对应于视差映射的视线投射）等步骤。这可以实现出最真实的凹凸效果，即使是视线和平面的掠角几乎达到90°的情况下。
仔细看Parallax Occlusion Mapping节点的输入值，我们会发现：
- Min Steps和Max Steps规定了投射光线的多少，这将决定当物体表面的采样程度。
- Hight Ratio则决定了视差的大小，值越大高度相差也越大。
- HeightMap Channel则表示高度图使用哪个通道作为真实的高度图（如果高度图是被打包进某个纹理贴图的，那么这个设置至关重要）

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220117015643.png"/>
