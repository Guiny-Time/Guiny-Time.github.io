---
tags: ['Unreal','shader','TA','技术美术','demo']
title: UE4 Materials 101：布料模拟
created: '2022-01-18'
categories: UE4-Materials101
cover: https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220119040418.png
---

我们将使用经验模型模拟布料材质。老规矩，先放一下效果图（边缘光和布料）

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/%E8%8F%B2%E6%B6%85%E5%B0%94UE4.gif"/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/%E6%AF%9B%E8%A1%A3.gif"/>

# John Havel法
John Havel的布料模拟和探索发现栏目里的边缘光有异曲同工之妙（下图）：
- 求出视线方向与片元法线的夹角并求反
- 利用指数增强边缘效果
- 利用指数修改内部效果（1-NdotV）
- 布料乘数 = 内环 + 外环 + 兰伯特（漫反射）
- 最终漫反射光乘上布料乘数

实际上，兰伯特的名词一出现，就需要知道这是一个经验模型，并不是完全基于物理的。我们并不能用shader graph真实的实现出布料的效果，所以这回先简单的模拟一下布料。


## 外环
外环的实现其实就是边缘查找，并且边缘查找的方式和边缘光一致。我们可以参考边缘光文章的shaderlab代码在shader graph中进行实现：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220119023039.png"/>

> **更有趣便捷的节点**
有一个数学节点的名字是“1-x”，它可以实现上图中subtract节点的效果（当我们要做的是1-x时）
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220119022709.png"/>

## 内环
我们对内环的处理和外环只差了一步，那就是少掉了“1-x”的求反步骤，其余是一致的：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220119023744.png"/>

## 优化
我们可以把片元法线替换成法线贴图的法线（记得从切线空间转换到世界空间！），这样就很棒了！加上贴图和法线之后，我们可以调整内环/外环的大小和强度来对布料进行模拟（以毛衣为例）:

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220119035349.png"/>
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220119035702.png"/>
