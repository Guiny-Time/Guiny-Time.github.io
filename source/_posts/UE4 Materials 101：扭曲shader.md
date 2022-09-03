---
tags: ['Unreal','shader','demo','技术美术','TA']
title: UE4 Materials 101：扭曲shader
created: '2022-01-15'
categories: UE4-Materials101
cover: https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220116010418.png
---

这种方法可以用来模拟水面的涟漪、火焰的效果等扭曲动效。丢一个完成的效果图：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/%E6%9D%90%E8%B4%A8%E7%90%83.gif"/>

这算是对前三期学习的一个总结篇。来解析一下shader graph里的组成！

# 噪声扭曲模块
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220116001431.png"/>

可以看到，噪声扭曲模块由**时间节点**、Vector2、**乘法**、纹理展开、**相加**、噪声纹理几个节点组成。
- 时间（Time）
时间随着时间而流逝（...），意思就是在程序运行的过程中，time值是不断变大的。
- Vector2
与时间相乘，用来调节时间的影响程度。当vector的x为0时，时间只影响v坐标，造成纹理在v方向移动；当vector的y为0时，时间只影响u坐标，造成纹理在u方向移动。x/y的值越大，影响的速度越快，反之越慢。当x或y为1时，时间就是时间本身。
- 纹理展开
我们需要让变化的结果施加到纹理上，因此需要texcoord来放纹理展开的情况。与乘法的结果相加后，纹理映射的结果也开始变化。
- 噪声纹理
噪声用来扭曲我们之后使用的base color。为了使扭曲不重复，引入了两组相同但是时间乘数不同的噪声。因为手上没有RGB通道不同的噪声，就直接用灰阶噪声来模拟一下（这样做出来效果没那么好就是了）。

# 纹理贴图模块
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220116004811.png"/>

我们把噪声扭曲模块得到的结果相加，再乘上一个用来控制扭曲强弱的乘数（在这里被定成0.2了），之后和调整好的纹理映射结果相加，输入到纹理贴图的uv中。
这时候，纹理就动起来了。把结果输出到base color和emission（为了让效果看起来更棒）中，就得到了不错的熔岩效果。
以下是完整shader graph：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220116001303.png"/>
