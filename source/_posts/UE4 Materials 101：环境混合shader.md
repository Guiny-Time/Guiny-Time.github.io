---
tags: ['TA','Unreal','技术美术','shader','demo']
title: UE4 Materials 101：环境混合shader
created: '2022-01-16'
categories: UE4-Materials101
cover: https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220116220135.png
---

环境混合shader适用于混合材质达成诸如“长了草的石头”、“下雪了的石头”这一类的效果。首先丢一个成品图：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/%E8%8D%89%E5%9D%AA.gif"/>

# 开始混合
我们可以简单的用Lerp函数进行插值混合。但Lerp的局限在于，只能全局的根据混合系数混合两张贴图，无法做到“上草地下石头”的效果：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220116205324.png"/>

# 引入分界线
我们可以用石头的法线贴图来实现分割的效果。由于蓝色的法线贴图是切线空间的法线贴图，我们需要先利用TransformVector节点将切线空间转换到世界空间，再进行操作。
由于在Unreal中，表示“上”的方向是z坐标，因此我们利用mask节点过滤出B值（如果过滤的是其他值，会得到草往侧边长的效果）。
> **错误的过滤值**
当我们选择过滤G值（也就是y值）的时候，我们得到的结果如下。可以看到，“长草”的一面往Y轴方向生长：
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220116214236.png" width=300/>

完成生长之后，我们将一个常量值与通道结果相加，用来控制草坪的覆盖情况。仅仅这样是可以直接接到lerp的控制alpha值中的，但是我们没有这样做，而是加上了一个指数操作，为什么呢？
在之前的项目中，我们用指数来实现了镂空边缘的火光效果。这和这里的原理是一样的：用来增强颜色，同时起到收束的作用。最后，将值限定在[0, 1]范围内，防止出现错误效果。
> **没有使用指数情况下的材质**
当没有使用指数时，草坪显得很暗淡（左）；当加上指数后，草坪的效果得到了增强（右）：
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220116215009.png" width=280/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220116215117.png" width=300/>

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220116212157.png"/>
