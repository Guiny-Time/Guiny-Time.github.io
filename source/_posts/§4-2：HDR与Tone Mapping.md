---
tags: ['HDR', '计算机图形学']
title: §4-2HDR与色调映射
date: 2021-09-01
categories: UnityShader
cover: https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211102161806.png
---

# HDR（High Dyna Range）
我们似乎经常在项目中见到这个词，比如相机中的HDR设置、选择颜色时的HDR选项等等：
<center><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211102132919.png" width=280/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211102133328.png" width=280/></center>

## 定义
HDR指的是High Dynamic Range，意味着图像具有高动态范围，这个动态范围体现在曝光程度上。通常来讲，在不同曝光程度下拍摄所获得的照片的动态范围都是有限的，我们很难同时清晰的呈现暗部与亮部，这类图像被称作LDR（Low Dynamic Range）图像，比如：
左边这张LDR图的曝光程度更高，但是同时我们也无法看清亮部细节：
<center><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211102134618.png" width=550/></center>
左边这张LDR图的曝光程度更低，但是亮部和暗部的细节都不清晰：
<center><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211102134754.png" width=550/></center>

而两张HDR图片的动态范围都要更大一些，能够覆盖亮部和暗部的细节

# 色调映射（Tone Mapping）
为什么需要色调映射？实际上，显示器（比如CRI，阴极射线管成像显示屏）和打印机只能呈现出有限的动态范围，对于HDR来说是远远不够的，因此需要一个色调映射的算法把大的范围映射到更小的范围来。将所有HDR值转换成在[0.0, 1.0]范围的LDR(Low Dynamic Range,低动态范围)的过程就是色调映射。
但值得注意的是，色调映射并不是一次将整个图像映射到监视器的亮度范围，而是**局部调整对比度**，以便图像的每个区域都使用整个范围来达到最大的对比度(具体的实现取决于使用的色调映射算法，比如Reinhard算法等等)。
如果不经过色调映射，一张HDR图像看起来可能很“平坦”，展现不出深度的细节，比如：
<center><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211102140904.png" width=300/></center>

原因是HDR中过大或者过小的范围没有经过映射，直接被压缩进了适合显示器的动态范围中。如果我们强行调亮亮度来看照片上的人，那么天空还会因为过于明亮而变成一片白色，效果很糟。经过色调映射之后，图像变好看了：
<center><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211102141431.png" width=300/></center>

# 泛光（Bloom）
在后处理（Post-Processing）的过程中，我们能够接触到一个名字叫Bloom的组件：
<center><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211102142524.png" Width=380/></center>

在经过这个组件的处理之后，携带了自发光材质的物体表面会出现泛光的效果：
<center><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211102142615.png" width=150/></center>

HDR非常适合Bloom的效果，可以造就相当真实的环境。而如果关闭了HDR，光照显得更加死板（左图在相机设置中关闭了HDR，而右图打开了）：
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211102143146.png" width=380 style="display: inline"/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211102143340.png" width=380 style="display:inline"/>

这是因为HDR有着更高的亮度范围，虽然在屏幕上被色调映射了，但在Bloom面前就凸显出优势了。**当我们拿到更高精度的数据去处理图像的时候，那么得到的也将是更高精度的结果**XD
<center><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20211102145123.png" width=480/></center>
