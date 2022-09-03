---
tags: ['Unreal','shader','demo','技术美术','TA']
title: UE4 Materials 101：shader逐帧动画
created: '2022-01-15'
categories: UE4-Materials101
cover: https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220116180243.png
---

# UE4 Materials 101：shader逐帧动画
shader逐帧动画可以用在各类被分割成帧拼接在一张图里的贴图。先丢个成品图：
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/%E7%88%86%E7%82%B8.gif"/>

## FlipBook实现法
这是UE4内置的一个节点，可以很方便（懒）的实现效果：
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220116015712.png"/>

需要输入贴图的长和宽，一组纹理映射以及时间（时间根据需要调整成需要的帧数）。输出的uv输入到纹理节点中，再输出到根节点即可完成flipbook效果。

## 节点实现法
我们将探寻flipbook节点内部的具体实现，这有助于对flipbook进行魔改。
### 缩放UV
首先，我们需要对贴图的uv进行缩放。因为这里使用的贴图是由6*6个小贴图拼合而成的，使用需要在uv方向同时缩放1/6，以使得每一张小贴图能够完全映射：
> **<span style="color:green">Frac函数</span>**
Frac函数的作用是将值的范围限定在（0，1）之间。当值超出1时，Frac函数会舍弃掉小数点左边的所有内容。

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220116170609.png"/>

### 跳转帧
接下来，我们需要uv在一定时间之后从第一帧往下跳。
在输入时间后，我们把时间参数调整为以1.5s为一个周期，其目的是方便之后的帧率设置。在这里我们先设定帧率（constant）为4，与时间相乘后经过Floor函数的处理与纹理映射相加，再经过上一步的缩放步骤得到固定在画面上的动画效果。
虽然现在实现了动画，但是动画仅仅沿着大贴图的对角线往下，效果很差。我们将在之后继续改进。
> **<span style="color:green">Floor函数</span>**
Floor函数的作用是对数进行四舍五入。这样可以让原本线性增长的时间呈现出阶梯式的变化，满足“跳转”的效果

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220116171336.png"/>

### 有序递增
我们还需要在现有的基础上，让画面从大贴图的第一帧开始一帧一帧的往下跳转。步骤是：
- 对时间除以6（因为6个小贴图一行，如果是其他情况再改），再用floor函数进行阶梯化处理
- 与阶梯化的时间相组合（横着跳6次，然后竖着1次，完成一行的变换）
- 加上纹理映射

之后，把帧速率从4改成24帧，就能还原Flipbook节点实现的效果。

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220116172753.png"/>












