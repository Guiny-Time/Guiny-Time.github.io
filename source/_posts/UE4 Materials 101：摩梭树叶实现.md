---
tags: ['TA','Unreal','技术美术','shader','demo']
title: UE4 Materials 101：摩梭树叶实现
created: '2022-01-19'
categories: UE4-Materials101
cover: https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220120011115.png
---

抬头看树林的树叶被风吹动时沙沙作响。要如何实现树叶沙沙作响的动画？对每片叶子进行调整，这工作量未免太地狱了一些。首先看一下最终实现的效果图：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/%E6%A0%91%E5%8F%B6.gif"/>

# 遮罩的使用
这次我们需要使用一张有alpha通道的贴图实现镂空的效果：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220119221917.png" width=250/>

首先，我们需要把材质的混合模式设置成遮罩，再把贴图的alpha通道与PBR根节点的不透明蒙版相连接，就能实现部分全透明部分不透明的效果了：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220119222202.png" width=300/>

# 让画面动起来
我们现在要实现树叶来回摆动的效果。
当提到摆动，我们应该能立刻想到具有周期性质的三角函数。我们可以利用时间的持续递增性，结合三角函数与控制变量实现整体的摆动效果：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/%E6%A0%91%E5%8F%B6-1.gif"/>
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220120001557.png"/>

# 让叶子产生摆动
现在，我们希望树枝上的叶子能够摆动起来，仅仅是整个树枝摆动没有什么生气。在这个步骤中，我们会使用到蒙版图，它长这样：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220120003005.png" width=300/>

你会发现我们用三种颜色（RGB）在一些比较独立的叶子上进行了叠加涂色。你不需要精心绘制遮罩层，只需要让每个颜色遮罩正确的覆盖需要覆盖的东西就好了（比如上图的马赛克一样的绘图）。这样，我们就可以通过过滤通道的数据获得三组不同的叶子组。接下来，我们回到shader graph中：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220120005033.png"/>

- 将我们先前实现的摇摆效果的constant变量替换成constant3Vector，这样做的目的是对三个遮罩通道产生不同的影响。
- 将遮罩纹理与摇摆的结果相乘，这样我们就得到了不同的动效。
- 将结果的三个通道累加并于texcoord相加，我们就实现了不同遮罩的不同动效。

# 调整法线
虽然我们已经得到了很不错的叶子摆动效果，并且影子的效果也很棒，但是它似乎还是少了点什么。是的，叶片的表面缺乏光影的变化，这让叶子们看起来像纸片，而非3D模型（虽然这里确实是纸片没错）。
实际上，我们可以对模型的法线动一点小手脚。我们不需要专门制作额外的法线贴图，可以简单的对叶片表面的法线进行扰动(0.9的作用是降低扰动程度)：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220120010159.png"/>

这样，我们就能够得到更棒的光影效果。