---
title: UE4 Materials 101：节点数据类型
created: '2022-01-15'
tags: ['Unreal']
categories: UE4-Materials101
cover: https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220115221056.png
---

很喜欢一句话：“因为浮点数精度的不同造成了z-fighting”。在早年写shader的时候，也因为数据类型的错误得到了一片紫光：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/39DFED12AE11D5D20AC531C4E3C93BDB.jpg" width=250/>

# Constant
常量内包含了一个浮点数（即float），可以用在金属度、粗糙度、镜面反射度等地方，因为它们都接受单一浮点数为输入。
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220115194059.png"/>
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220115194309.png"/>

当把浮点数连接到base color时，它代表灰阶的所有颜色，但是最好把constant限制在[0,1]范围内，超出也不会报错，只会是白色或灰色（看哪一段超出了）：
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220115194614.png"/>

## 数学
我们可以对常量进行数学运算，将结果输出到其他节点。这很重要，因为我们经常需要对一些值进行换算处理，包括且不限于加减乘除。有以下数学节点（没截全）：
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220115200125.png" width=300/>

# Constant2Vector
包含了两个浮点数的浮点数对，很适合表示二维坐标数据，例如uv坐标。
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220115200321.png"/>

## 数学
我们也可以对浮点数对进行数学运算。相当于对两个二维向量进行操作。而当我们只需要浮点数对的其中一个值时，我们可以使用mask节点来对变量进行拦截。mask同样可以使用在其他浮点数组合（比如Vecor3, Vector4）上：
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220115202153.png"/>

除此之外，如果觉得要检查遮罩的值很麻烦，可以用split节点对组合进行拆分。但是需要注意的是，splitComponents只支持输入Vector3类型：
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220115202541.png"/>

# Constant3Vector
包含了三个浮点数对，很适合三维坐标系。截取方式和上文类似，但是需要注意的是进行加减法运算时，只能和标量或者同类型的浮点数组合进行运算（例如，Vector3不能和Vector4加减乘除）
如果想要通过一个常数和一个浮点数对合成一个Vector3，可以用append节点：
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220115211522.png"/>

如果想用constant更自由的配置（不仅是输入，还有输出上），可以用appendMany节点：
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220115212031.png"/>

如果想改变组合顺序，可以用swizzle节点，它可以改变浮点数对以及Vector3的顺序：
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220115212229.png"/>

# Constant4Vector
包含了四个浮点数对，很适合齐次坐标空间的坐标，以及带有alpha值的颜色。但值得注意的是，base color虽然可以接入Vector4节点，但对于第四个通道是忽略的。不透明度并不会影响base color。
其他部分和Vector3类似。

