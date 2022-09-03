---
tags: ['Unreal','shader']
title: UE4 Materials 101：shader性能优化
created: '2022-01-16'
categories: UE4-Materials101
cover: https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220117010246.png
---

优化是一个很棘手的问题。当shader写的很烂，就会造成大量不必要的计算和复杂度，造成时间和空间上的浪费。比如我们想得到一个动效，结果动的很慢等等。好在UE4内置了几种用来优化shader性能
# 性能探寻
## Shader复杂度观察
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220116232426.png" width=300 style="float:left"/>

我们可以在视图界面找到Shader Complexity View Mode的入口（上图所示）。进入该模式之后，摄像机视野中所有物体都将展示出对应的优化程度所对应的颜色。
我们可以看到，视图下方有一个渐变色栏，这代表了shader的复杂程度，与视窗中的材质颜色对应。
这种方法只能帮助你找到开销大的shader，并不能提供优化它的方法。此外，着色器复杂度的衡量方式是指令的数量，这并不能很好的判断一个着色器的开销是否太过庞大。

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220116233158.png" width=400/>
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220116233432.png" width=400/>
<br/>

## Shader指令数量
当我们进入每一个着色器的节点图时，打开统计数据，我们就会看到统计数据中显示的shader指令数量(base pass shader)。
这里的指令数量指的是什么？当shader graph生成之后，UE4会帮助我们将Shader Graph转换成HLSL代码，再编译成**编译指令**，最后传递给图形驱动完成渲染。而统计数据内的指令数量指的就是最终传递给图形驱动的编译指令，代表了该着色器需要向驱动传递执行多少条指令才能完成渲染。
但并不是所有指令的执行速度都是一样的，用指令数量来衡量shader的优劣并不绝对。

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220116234019.png"/>

## 实机测试
这是测试shader性能最好的方法！把shader应用到实质的模型上，再导出到对应的平台进行测试，得出相应的结论。当然，实施这一步也是最麻烦的，因为需要许多准备步骤。

# Shader优化
## 删掉所有不要的东西
这看上去是废话，但是很重要。比如说庞大的运算的结果是0；贴图是纯黑的，那么这些就是不必要的、可以用constant来代替的。
此外，不要给一个shader写太多功能，但又只用到其中的几个，这就是一种浪费。就像面向对象的单一职责原则一样，一个shader最好也是实现一种单一的功能。

## 重构数学公式
如果可以用更简单的公式得到同样的结果，那么就用更简单的公式。复杂的数学计算也会拖累shader的渲染速度。
在前一篇文章（环境混合shader）中，我们用到了指数来增强图像对比度。实际上，在时间复杂度理论中，指数是最浪费的一种复杂度。在实际的数学运算中，指数的开销也是很大的，呈献一种飙升的趋势。我们可以通过乘法的方式得到类似指数的效果，这样做的话开销就会小很多。
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220117003756.png"/>

## 流水线化
当一个shader graph里有相同的部分的时候，我们可以考虑把它提取出来做成一个单独的部分，但是又可以代表原本的相同部分。
比如说，我们可以用一个Vector4去存储两个Vector2，完成原本两个Vector2的重复动作。下图的功能就是一致的：
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220117005248.png"/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220117005337.png"/>

## 打包纹理通道
当我们需要对很多纹理进行采样（这开销很大）时，比如在PBR流程中采样好几张纹理，我们可以尝试对纹理的通道进行打包处理。这个意思是我们可以把几张纹理合在一张纹理的不同通道里，得到一张新的贴图。
我们可以把金属度贴图、光泽度贴图和粗糙度贴图以及AO合在一起。因为这几种贴图通常只采样单通道的结果，是纯灰阶的。但对于法线贴图就不能这么做，因为它的三个通道都储存了对应信息，不能更改。
