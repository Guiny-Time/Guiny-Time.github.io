---
tags: ['TA','Unreal','技术美术','shader','demo']
title: UE4 Materials 101：雨水系列2-雨滴效果
created: '2022-01-20'
categories: UE4-Materials101
cover: https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220121010320.png
---

&emsp;&emsp;现在，我们要实现物体表面的雨滴变化效果，这种效果常见于叶子、玻璃表面的水：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/%E9%9B%A8%E6%BB%B4.gif"/>

# 使用蒙版
从Ben神那里拿来的纹理是已经打包好的蒙版，其中：
- RG通道表示法线信息
- B通道表示的是**受时间影响而产生动画**的部分雨滴，灰度的不同代表了影响的强弱
- A通道明确区分的不动的雨滴与会动的雨滴

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220120172915.png" width=300 style="display:inline"/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220120173012.png" width=302 style="display:inline"/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220120173220.png" width=300 style="display:inline"/>

## 解压法线
&emsp;&emsp;前文提到，法线纹理的两个分量被放在了RG两个通道，现在我们要解压出来并最终输入法线。当我们使用法线贴图的时候并不用经历这一步骤，为什么呢？因为纹理被识别或人为定义为法线之后，UE4就已经帮我们准备好换算的步骤了，因此直接连上就可以了。但在这里，我们是从一张遮罩贴图里解压出来的信息，因此需要手动进行转换。
转换的过程很简单：我们需要把范围为[0, 1]的颜色信息转换到范围为[-1, 1]的向量信息。实现的方法即$x * 2 - 1$。由于这样得到的结果只有xy两个分量，我们需要用append节点加上z坐标，这样就能得到正确的法线效果：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220120204546.png"/>

## 解压动画蒙版
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/%E9%9B%A8%E6%BB%B4-1.gif" width=300/>

&emsp;&emsp;我们在上文提到，我们将动画部分放在了B通道，这个通道控制动画影响的强弱。我们可以：
- 使用时间（并引入控制时间强弱的变量）节点让画面动起来
- B通道的值与时间输出相减，造成从亮变暗的效果
- 引入Frac节点形成越界循环，控制值在[0, 1)之间

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220120210114.png"/>

## 解压区域限制
&emsp;&emsp;可以看到，通过解压B通道我们得到了雨滴落下的效果，但是影响的范围确是全局的，这并不是我们想要的。这时候我们就可以用存在A通道的区域数据了。获取数据的方法也是$x * 2 - 1$，因为我们最终处理的是坐标数据。与动画部分相乘之后，我们就得到了差不多正确的效果了。

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220120211436.png"/>

## 修改法线
&emsp;&emsp;我们可以把动画部分和法线结合，这样就可以使法线和雨滴效果一起变动，产生更真实的效果。但是我们发现一个问题：现在拿到的法线只有动画部分的水滴，而固定在材质上的那部分水滴的法线被过滤掉了：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/%E9%9B%A8%E6%BB%B4-2.gif" width=300/>

&emsp;&emsp;解决的方法也很简单，我们需要获得另一部分的法线，然后加回去（*-1是为了让部分灰色和白色动画部分被滤去，同时原本黑色的固定部分变为白色。指数的作用在于将没有被完全滤去的灰色过滤）：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220120213300.png"/>

# 修改下落方向
&emsp;&emsp;我们只希望雨水出现在物体的表面（z轴方向），而不是到处都是。结合之前的环境混合shader一文，我们可以这样做：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220121003614.png"/>

# 更加物理！
&emsp;&emsp;我们可以让水滴的效果更物理一些，比如加上**粗糙度**：
- 反转我们之前的到的法线合集
- 利用1-x节点对颜色进行翻转
- 连接power节点对效果进行增强
- 连接粗糙度节点，输出结果

&emsp;&emsp;并且，你会发现其实水滴下落被吸收之后马上又落下了，这有点不合常理。我们可以Frac节点之后增加一个值来控制两滴雨滴之间的时间间隔。