---
tags: ['Unreal','shader','TA','技术美术','demo']
title: UE4 Materials 101：体积冰实现
created: '2022-01-19'
categories: UE4-Materials101
cover: https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220119212421.png
---

体积冰的效果适用于可以看透但是不能穿过的半透明物体，例如冰块、石英柱。首先放效果图：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/%E5%86%B0.gif"/>

# 制造体积感
我们第一步要做的事情是营造出**冰层**的体积感。我们需要贴图“下沉”到冰块里面，实现的方法就是营造视觉偏移。在前面的文章中，我们利用视角（相机）方向和片元法线找到边缘，在体积感的场景中，我们则利用了视角在片元面上的反射。有点抽象，其实就是下图所示的情况：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220119182121.png" width=350/>

蓝色的向量就是我们所需要的冰层内层的偏移方向。把这个向量加到uv里之后，uv就会随之偏移。

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220119201150.png"/>

可以看到，我们将发现信息用一张法线纹理来展示，这样可以更生动的展现冰面的细节。由于<span style="color:red">**法线**</span>贴图是切线空间下的法线贴图，所以需要将<span style="color:green">**视角方向**</span>从世界空间转换到切线空间。得到反射的结果之后，利用mask节点取得<span style="color:blue">**反射向量**</span>的x、y坐标。
实际上，在这一步完成时，我们已经得到了一种镜面的效果（只不过并没有反射站在对面的人的形象）。我们还需要把这一移动壳层往物体内部移动，才能塑造出体积感：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220119202319.png"/>

我们对内壳的z值动了手脚：
- 纹理：用于改变内壳壳层不同凹凸程度的贴图
- 求反：为了实现正确的凹凸，我希望白色的部分在上，这样更符合冰层的实际效果
- /250：即最大凹陷程度（当纹理的颜色为黑色时），这个值是多少可以自行改变
- 除法：将z值缩放施加最终输出流上

# 调整分辨率
我们把上述过程完成的值输出，得到了非常奇怪的不正常的效果。可以看到，当对图像进行放大之后，我们会看到很多小的贴图：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220119211629.png" width=300/>

这是因为没有调整贴图的分辨率。因为我使用的两张贴图都是1024*1024的，所以加上这个节点组对分辨率进行调整：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220119211514.png"/>

现在得到的效果就是正确的了。
