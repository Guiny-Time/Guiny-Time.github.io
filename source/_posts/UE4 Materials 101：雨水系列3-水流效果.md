---
tags: ['TA','Unreal','技术美术','shader','demo']
title: UE4 Materials 101：雨水系列3-水流效果
created: '2022-01-20'
categories: UE4-Materials101
cover: https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220121050613.png
---

&emsp;&emsp;这个效果指的是雨水从材质的侧边流下的效果。与雨滴效果不同的是，雨滴效果只展示在材质顶端，而水流聚集在物体侧面。看一下效果图：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/%E6%BB%91%E6%B0%B4.gif"/>

# 使用蒙版
&emsp;&emsp;本文同样使用了Ben神的贴图，不同通道包含了不同的信息：
- RG通道：法线信息
- B通道：边缘略微模糊的黑白图，描述了水流发生的位置，可以拿来当高度图用，也可以用于后续的粗糙度等场合
- A通道：蒙版，利用值的不同产生不一样的流动效果（几柱水流不会一起动，不然很鬼畜）

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220121013036.png" width=200 style="display:inline"/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220121013244.png" width=200 style="display:inline"/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220121013422.png" width=200 style="display:inline"/>

# 世界空间投影
&emsp;&emsp;这是通过本文学习到的重要的知识之一。世界空间投影的意思是：不管施加材质的对象有多大、多远、如何旋转等等，贴图总是从世界空间的一个方向投射来，映射在物体上。你可以想象把物体沿一定的方向放进有图案的油漆桶里，不过要记得在一个方向上的投射情况基本一致（例如：投射的正前方和正后方），侧边会有所不同。

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220121015732.png" width=230 style="display:inline"/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220121015752.png" width=200 style="display:inline"/>

&emsp;&emsp;在shader graph中，我们可以用这种方式来实现:

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220121015537.png"/>

&emsp;&emsp;你会发现在这里我们过滤出了RB两个通道，但是舍弃了G。这样做的原因是我们需要对特定的方向进行投影，所以G是不需要的。并且，如果加上G通道会导致报错，因为uv坐标只接受Constant2Vector

## 控制投影面
&emsp;&emsp;上一步完成之后，我们发现侧面的效果像被拉长了一样，很糟糕。我们的目的是能从正面和侧面完整的看到投影，而不是拉长的意大利面。我们通过以下几步进行修改：
- **添加侧面的世界空间投影**
这是我们的第二个投影，它将弥补被拉长的侧面
- **双线性插值**
我们对两个面的投影进行一次双线性插值，这将更好的混合
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220121030551.png"/>
- **创建蒙版**
我们需要创建几个蒙版，用于“可以看到效果”的面，比如说正面和某个侧面。要实现这个效果，可以从片元法线下手。
我们利用Sign节点将大于0的值变为1；将小于0的值变为-1，再经过saturate节点的过滤，我们就只剩下六个面中的三个了（另一半全因为Sign处理的值为-1而被舍弃了）
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220121032903.png"/>
我们将蒙版添加到双线性插值的alpha输入中，这样就实现了根据面的方向进行混合的效果。
- **添加视角信息**
我们现在有两个面的混合信息了（正面和侧面），我们需要确定一件事：摄像机是从哪一面看的？是正面还是侧面？因此，我们需要额外添加以下代码，并输入两个面的混合的alpha中：
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220121033707.png"/>
这样，就会同时在正面和侧面进行投影的。

# 制作蒙版动画
&emsp;&emsp;我们需要实现水流“从上往下”的效果。这种效果在之前是没有实现过的，我们之前实现的动画效果仅仅围绕在噪声解析、三角函数（sin）和利用frac的周期性上，这几种方法都无法满足需求。
在本文中，我们将介绍使用蒙版贴图进行动画。

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220121035214.png"/>

&emsp;&emsp;首先引入我们所需要的遮罩纹理，进行缩放之后仅保留部分面的结果（选取了RB通道，即不包括上下两面的剩下所有面）。接下来，我们引入Time参数让画面动起来：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/%E6%BB%91%E6%B0%B4-1.gif" width=300/>

&emsp;&emsp;但是这样还不够。忘了被包装在贴图A通道里的信息了吗？我们可以把它与Time节点相加：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/%E6%BB%91%E6%B0%B4-2.gif" width=300/>

&emsp;&emsp;还不够！现在每条路径上的水的流速是一样快的，但是现实世界中，不同水路上的流速差别可以是很大的，我们需要对其进行调整。

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220121042101.png"/>

&emsp;&emsp;现在就只剩下最后一步了：让水在水流的路径上走。这很简单，只需要让储存在包装好的纹理的B通道的信息与遮罩纹理相乘即可：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/%E6%BB%91%E6%B0%B4-3.gif" width=300/>

# 修改法线
&emsp;&emsp;现在我们的法线是定死在材质上的，这样很难看，也很不科学，因为水路不会一直存在，它们是伴随着水滴的。
我们可以对法线做一点小修改，只需要乘上流动的白色水滴：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220121044543.png" width=350/>

# 让水流只从侧边流下
&emsp;&emsp;当我们把模型切换成球的时候，我们会发现水从球的顶端冒了出来，而我们希望水只从侧边冒出、流动、消失。因此，我们需要手动加上一个额外的蒙版来限制水流的范围：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220121045532.png"/>
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/%E6%BB%91%E6%B0%B4-4.gif" width=300/>