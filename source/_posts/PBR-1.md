---
tags: ['技术美术', '次世代渲染基础', 'PBR', '基于物理的渲染']
title:  PBR白皮书笔记-1
date: 2021-11-12
categories: 次世代渲染基础
cover: https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211203153248.png
---

# PBR概述

第一次参加TA的面试，面试官上来几个问题还能答上来，结果问到PBR给卡住了，因为之前了解过一些但还没有深入看过。通过这次面试知道了PBR作为次世代渲染的基础非常重要，是每个TA必须掌握的技能。于是我打算专门开个新的专栏来写PBR相关的笔记，避免下次踩坑。

# 概述
PBR指的是Physically-Based Rendering，即基于物理的渲染，能够在不同的光照环境下展示出真实正确的效果从而更精确的描绘光和表面之间的作用，分为Metallic/Roughness工作流和Specular/Glossiness工作流两种工作流。PBR最重要的三个东西是：
- PBR材质
- PBR灯光
- PBR相机
<center><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211121005612.png" width=600/></center>

基于真实物理的材质、灯光和相机相结合，我们才得到了真正的完整的PBR系统。在这之前，我们首先来看一下PBR的理论基础。

## PBR理论基础
<center><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211120003206.png" width=800/></center>

因为是基于真实物理的渲染，PBR的三个理论基础也是仿照现实世界中的物体定义的。
### 微平面理论（Microfacet Theory）
我们认为一切物体的表面都存在凹凸不平的微小的理想镜面，微平面的凹凸程度与物体的光滑程度相关。在实际的PBR工作流中，这种物体表面的不规则性用粗糙度贴图（Metallic/Roughness工作流）或者高光度贴图（Specular/Glossiness工作流）来表示。
<center><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211120003509.png" width=500/></center>

粗糙度与微平面的具体关系在于：越粗糙的表面，微平面凹凸不平的就越厉害，光照射上去之后就会沿着非常不同的方向反射开。而粗糙度越低，光反射的方向越为集中。通常，我们用统计学来展示不同粗糙度表面的反射情况。
<center><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211121001839.png" width=500/></center>

然而光与物体相交不仅仅只有反射一个选项，还有折射和吸收两个选项（如下图）。
<center><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211121002346.png" width=500/></center>

我们可以概括为：
- 光照到金属物体上--->部分光被电子吸收（吸收，左图）
- 光照到非金属物体上--->部分光在物体内部多次反射，最后射出物体或被吸收（折射+吸收，右图）

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211121003146.png" width=300 style="display:inline"/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211121003235.png" width=300 style="display:inline"/>

光在非金属内吸收+折射最终从物体表面射出的过程叫做**次表面散射（Subsurface scattering，SSS）**。SSS对于非金属材质非常重要，能够更真实的模拟材质的表面：
<center><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211121004622.png" width=600/></center>

#### 漫反射和次表面散射

在物理学上，漫反射和次表面散射其实是相同的物理现象，本质都是折射光的次表面散射的结果。唯一的区别是相对于观察尺度的散射距离。散射距离相较于像素来说微不足道，次表面散射便可以近似为漫反射。也就是说，光的折射现象，建模为漫反射还是次表面散射，取决于观察的尺度。

### 能量守恒（Energy Conservation）
我们认为照射到物体上的光最终被被反射或者，被反射的光和被吸收的光的总和等于入射光（下图中的光被完全反射了，没有吸收），所以能量守恒可以描述为出射光线的总和不超过入射光线
<center><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211120003741.png" width=500/></center>

### 菲涅尔反射（Fresnel Reflectance）
光线以不同角度入射会有不同的反射率。相同的入射角度，不同的物质也会有不同的反射率。我们认为一切物体都存在菲涅尔反射（越靠近边缘反射率越高），只是存在强弱之分。
菲涅尔反射中的F0指的是入射角度为0时的菲涅尔反射值。大多数非金属的F0范围是0.02\~0.05，大多数金属的F0范围是0.7\~1.0。
<center><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211120004018.png" width=500/></center>
<center><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211120004624.png" width=500/></center>

除此之外，PBR还涉及线性空间、伽马矫正、色调映射等内容，我在Unity Shader的专栏中提到过。

