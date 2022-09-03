---
tags: ['Unreal']
title: UE4 Materials 101：纹理压缩设置
created: '2022-01-16'
categories: UE4-Materials101
cover: https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220117031021.png
---

# 纹理细节
对于每一张导入UE4的纹理，我们在双击纹理时，可以在右侧的**细节**栏看到相关的纹理信息（包括分辨率、大小等）。
> 对于每一张纹理，他的大小最好是$2^{n} * 2^{n}$的，比如说16 * 16、1024 * 1024、2048 * 2048等。

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220117022644.png"/>

- 已流送
贴图将在游戏中动态加载。
- DXT5
最有效的一种压缩格式。
- Mipmap
即多级渐远纹理，记录了一组逐渐变小的纹理链，最小的纹理是4*4。当我们从远处看或者以倾斜的角度观察物体时，物体远处的纹理是比较小的。这时可以调用mipmap链中较小的纹理进行映射，达成比较好的视觉效果。

# 压缩
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220117023828.png"/>

- 无透明度压缩
不管这张贴图的alpha通道有没有东西，当勾选了无透明度压缩之后，都会当成alpha通道完全不存在来处理。当alpha不重要或者这张图根本没有alpha通道时，务必勾选此项。
- 压缩设置
这里面其实有许多可选项，最好是根据贴图类型进行选择（这和在导入贴图进Unity时为贴图选择正确的类型类似）。默认的DXT1非常强，可以把贴图压缩到原本的1/8。
>- 我们对法线贴图选择normal，对打包的纹理选择masks。masks的alpha通道的压缩偏少，所以最好把比较重要的内容放进alpha通道。
>- 对于高度图这样的**小**的**灰阶**纹理，我们可以选择GrayScale，这会改善被压缩的高度图
>- Alpha适合仅需要alpha通道信息的纹理的压缩
>- BC7是一种新推出的高质量纹理压缩格式，仅支持DX11或以上版本。它压缩完的效果更好，但是体积比DXT1要大两倍（只压缩4倍）。支持三通道和四通道纹理。

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220117024335.png"/>

# 纹理
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220117025237.png"/>

这里最重要的是srgb勾选框。需要注意的是，一切工作在线性空间（不经过伽马矫正）的纹理都**不能勾选**这个选项。这些纹理包括粗糙度贴图、AO、金属度贴图、高度图等等。基本上，我们只对base color打开srgb选项。

# 在Shader中
我们还需要特别注意shader中的texture Sample节点是如何定义贴图类型的。当采样器中的贴图类型与贴图实际的压缩类型不一致时，会出现采样错误导致纹理无法正确显示。
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220117030729.png"/>
