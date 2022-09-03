---
tags: ['TA','Unreal','技术美术','shader','demo']
title: UE4 Materials 101：雨水系列1-潮湿的表面
created: '2022-01-19'
categories: UE4-Materials101
cover: https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220120160903.png
---

&emsp;&emsp;当一个东西被水浸湿，它看起来会是什么样子的？首先，吸水性差的物体吸水后，整体的颜色会变暗，并且色彩会更加鲜艳。其次，物体的表面会反射出水的光泽，就如下图的效果图所示：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/%E6%B9%BF%E6%B6%A6%E8%A1%A8%E9%9D%A2.gif"/>

# 吸水变暗
&emsp;&emsp;首先，我们要实现吸水之后色彩变得更鲜艳同时变暗的效果。要完成这个操作，我们需要对拉高材质的饱和度：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220120142622.png"/>

&emsp;&emsp;Desaturation节点其实直观的来看是用来调整材质表面的冷暖程度的（因为我们现在的控制变量是constant而不是constant3Vector），控制程度的变量 > 0时，越大材质越蓝；控制变量 < 0时，越小材质越暖。我们在这里设置变量值为-1，再限制在[0, 1]之间，就能得到鲜艳的输出颜色。
> **Desaturation和Saturate节点**
&emsp;&emsp;乍一看很像，但是用途完全不同。前者的作用是调整输出颜色的饱和度，而后者则是把输出的值归一化到[0, 1]范围内。

&emsp;&emsp;但是这个效果还不够，我们还需要加上吸水“变黑”的颜色。这个实现的方法很简单，把输出的颜色乘上一个比较黑的数即可：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220120143517.png"/>

# 混合
&emsp;&emsp;物理常识：由于水真的很光滑，粗糙度可以设置在0.07，而镜面反射系数是0.3（一般物体是0.5，水面略低）。现在我们用R通道储存水的粗糙度，用G通道储存水的镜面反射：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220120145036.png"/>

&emsp;&emsp;然后，我们可以把属于水面的反射/粗糙度值与原本材质的值进行混合。由于该材质并没有设定镜面反射程度，所以我们利用append节点默认加上0.5作为镜面反射系数。混合完成之后，我们可以利用mask节点拆出对应的数值输入根节点。除开两个参数（光泽度/粗糙度）之外，我们还需要对颜色进行混合（调整过颜色的结果和原本输出的结果）。
&emsp;&emsp;现在，我们得到了两个Lerp混合节点，我们可以添加一个用于控制湿润程度的变量连接两个lerp节点的alpha输入。

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220120145904.png"/>

&emsp;&emsp;至此，我们就完成了湿润表面的制作。我们可以把相关的代码块包装成函数，因为在许多情境下我们都需要使用到潮湿表面的函数。