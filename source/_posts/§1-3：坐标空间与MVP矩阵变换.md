---
tags: ['Unity Shader', '坐标空间', 'Unity', '线性代数']
title: §1-3坐标空间与MVP矩阵变换
date: 2021-07-17
categories: UnityShader
mathjax: true
cover: https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/90572417_p0.jpg
---

<center><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20210924154412.png" width=800px/>
<p>从观察者的观察空间投影到屏幕空间</p></center>

&emsp;&emsp;什么是MVP矩阵？MVP矩阵的三个字母指的是模型矩阵(Model)、观察矩阵(View)和投影矩阵(Projection)。回想一个模型从导出到最终呈现在屏幕上的过程，我们发现模型的顶点的坐标经历了这么些事：
- 在建模软件中，模型顶点的坐标是以模型中心为原点计算的
- 在把模型拖入场景之后，模型顶点的坐标是以世界中心为原点计算的
- 从摄像机出发观察模型
- 将模型投影到摄像机上，进行渲染，最终呈现在玩家眼前

&emsp;&emsp;实际上，这个变换就是使用了MVP矩阵。从模型空间的顶点，我们最终得到了投影在相机上的结果。再想想写shader的时候，顶点着色器往往包含以下语句：
```ShaderLab
v2f o;                                  // 传输到片元着色器
o.pos = UnityObjectToClipPos(v.vertex); // 将顶点坐标从模型空间(Object)转换到裁剪空间(Clip)
```
&emsp;&emsp;我们知道，顶点着色器最基本的功能就是将顶点坐标从模型空间转换到裁剪空间。这里的UnityObjectToClipPos函数的作用即将顶点坐标乘上MVP矩阵进行变化，实现了一个物体从自己的模型空间(也成为局部空间/Local Space)经过模型矩阵(M)，转换到世界空间(World Space)；再经过观察矩阵(V)，进入到摄像机所观察到的视觉空间(View Space)；最后通过投影矩阵转换到裁剪空间(Clip Space)的过程。

# 模型空间--->世界空间
&emsp;&emsp;模型空间就是在建模的时候以模型自身中心为中心的坐标系；世界空间指的是以世界中心为中心的坐标系。
&emsp;&emsp;感觉好像对模型空间很生疏？实际上，当一个物体有子对象的时候，父物体的坐标是世界空间下的坐标，而子物体的坐标其实就是模型空间下的。举个例子，现在我们的地图中有一个电脑作为父物体，而显示器、键盘都是子物体。当我们聚焦整个父物体时，我们发现它的Transform组件下的坐标就是世界空间下的坐标:
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20210926155901.png" width=200/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20210926160002.png" width=310/>
&emsp;&emsp;而当我们聚焦电脑下的组件（比如键盘）时，Transform组件显示的坐标就是整个电脑模型空间下的坐标：
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20210926160252.png" width=200/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20210926160322.png" width=310/>

&emsp;&emsp;再比如说，我们在处理模型的时候经常会遇到上(Vector3.up)、左(Vector3.left)、前(Vector3.forward)等方向性的概念，而萌新使用这些概念的时候往往无法得到正确的结果，比如我们希望这个模型通过Vector3.up明确方向实现向上移动，但是它怎么朝着莫名其妙的方向动了呢？这是因为搞混了模型空间和世界空间。Unity脚本API中对Vector3.up的描述是：用于编写 Vector3(0, 1, 0)的简便方法。然而这里的(0, 1, 0)是对物体的模型空间而言的，而不是世界空间。举个例子：


// 请在这里插入手绘示意图


&emsp;&emsp;这就是出错的原因了。我们提到的上下左右前后几种方向被称为“自然方向”，由于Unity使用左手系的缘故，这些方向和坐标轴的对应关系为：
- +x ---> 右/right
- +y ---> 上/up
- +z ---> 前/forward
## 模型矩阵M
&emsp;&emsp;那么，我们该如何将点/向量从模型空间转换到世界空间呢？回想一下刚刚我们提到的父对象的Transform坐标！
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20210926160002.png" width=310/>
&emsp;&emsp;父对象的Transform的坐标指的就是这个模型原点在世界空间下的坐标，从(0, 0, 0)变成(-2.918, -3.026, 0.091)了呢，还有旋转变换和缩放变换。实际上，我们需要的就是一个矩阵的复合变换来实现这个过程。变换方式如下(顺序不能改)：
- 进行缩放
- 进行旋转（绕各轴旋转的顺序可以打乱，比如说我下面写的是先转z然后y然后x）
- 进行平移
$$M = 
\begin{pmatrix}
   1         & 0        & 0        & t_{x} \\
   0         & 1        & 0        & t_{y} \\
   0         & 0        & 1        & t_{z} \\
   0         & 0        & 0        & 1
\end{pmatrix}  \begin{pmatrix}
   1         & 0              & 0             & 0 \\
   0         & cos\theta_{x}  & -sin\theta_{x}& 0 \\
   0         & sin\theta_{x}  & cos\theta_{x} & 0 \\
   0         & 0              & 0             & 1
\end{pmatrix}  \begin{pmatrix}
   cos\theta_{y}  & 0        & sin\theta_{y}  & 0 \\
   0              & 1        & 0              & 0 \\
   -sin\theta_{y} & 0        & cos\theta_{y}  & 0 \\
   0              & 0        & 0              & 1
\end{pmatrix}  \begin{pmatrix}
   cos\theta_{z}  & -sin\theta_{z}  & 0       & 0 \\
   sin\theta_{z}  & cos\theta_{z}   & 0       & 0 \\
   0              & 0               & 1       & 0 \\
   0              & 0               & 0       & 1
\end{pmatrix}  \begin{pmatrix}
   k_{x}     & 0        & 0        & 0 \\
   0         & k_{y}    & 0        & 0 \\
   0         & 0        & k_{z}    & 0 \\
   0         & 0        & 0        & 1
\end{pmatrix}
$$
代入数据得：
$$M = 
\begin{pmatrix}
   1         & 0        & 0        & -2.918 \\
   0         & 1        & 0        & -3.026 \\
   0         & 0        & 1        & 0.091 \\
   0         & 0        & 0        & 1
\end{pmatrix}  \begin{pmatrix}
   1         & 0              & 0             & 0 \\
   0         & \cos{44.003}  & -\sin{44.003}& 0 \\
   0         & \sin{44.003}  & \cos{44.003} & 0 \\
   0         & 0              & 0             & 1
\end{pmatrix}  \begin{pmatrix}
   \cos{85.543}  & 0        & \sin{85.543}  & 0 \\
   0              & 1        & 0              & 0 \\
   -\sin{85.543} & 0        & \cos{85.543}  & 0 \\
   0              & 0        & 0              & 1
\end{pmatrix}  \begin{pmatrix}
   \cos{-94.927}  & -\sin{-94.927}  & 0       & 0 \\
   \sin{-94.927}  & \cos{-94.927}   & 0       & 0 \\
   0              & 0               & 1       & 0 \\
   0              & 0               & 0       & 1
\end{pmatrix}  \begin{pmatrix}
   1.8     & 0        & 0        & 0 \\
   0         & 1.8    & 0        & 0 \\
   0         & 0        & 1.8    & 0 \\
   0         & 0        & 0        & 1
\end{pmatrix}
$$

&emsp;&emsp;具体结果是什么我就不算了，这庞大的计算量。。。总之，在计算的最后我们会得到一个4*4矩阵，这就是针对该模型的模型矩阵 $M$ 。利用这个矩阵，我们现在可以求得子物体Transform坐标（模型空间坐标）的世界空间坐标了！
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20210926160322.png" width=310/>
&emsp;&emsp;将键盘的模型坐标$\begin{pmatrix}-0.005815\\0.0167358\\0.1960645\\1\end{pmatrix}$乘上刚刚算好的模型矩阵M，我们就能得到就能得到键盘的世界坐标。

# 世界空间--->观察空间
&emsp;&emsp;观察空间就是以摄像机为中心的坐标系。我们可以认为观察空间也是模型空间的一种，**想象一下所有被观察的物体都是相机的子物体**，那么我们要做的就是某种意义上的“从世界坐标重新转换成模型坐标”的过程。这不是恰好和刚刚进行的“从模型空间坐标转换成世界空间坐标”的操作正好反过来了吗？
&emsp;&emsp;没错！从模型空间到世界空间，$M$ 矩阵的操作是先缩放、旋转，最后平移。那么反过来的过程就应该是**先平移再旋转**。由于缩放对相机视野并不产生影响，所以我们不考虑缩放的选项。不过到这里还没有结束，我们还**需要对z轴进行逆变换，因为观察空间是右手系**（也是几个坐标空间中唯一的右手系），而世界空间是左手系。为什么呢？因为我们希望相机的前方是z轴的负方向，这样一来观察空间中每个能够进入摄像机视野的物体的z值都是正数，并且表示了该物体和摄像机的距离。
&emsp;&emsp;为什么非要搞这一出呢？在接下来的学习中我们会接触到**深度值**的概念。在对每个物体进行渲染之前，我们会将物体片元的深度值写入深度缓冲中（关闭深度写入另谈）。当同一个位置上出现多个片元时，我们会只渲染深度值最小的那个片元。当z值（深度）都是正值时，我们能够对数据更直观的分析。
> **另一种思考模式**
&emsp;&emsp;我们可以想象将摄像机移动到世界中心，再将相机的坐标轴与世界空间的坐标轴对齐。这样处理之后得到的观察矩阵和前文提到的相同。
&emsp;&emsp;相机从世界中心原点移动到它所在的位置，是先平移，再旋转的。那么移动回来，就需要先旋转，再平移。

## 观察矩阵V
&emsp;&emsp;以这个摄像机为例：
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20210926171746.png" width=190/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20210926171828.png" width=441/>

$$V = 
\begin{pmatrix}
   1         & 0      & 0     & 0 \\
   0         & 1      & 0     & 0 \\
   0         & 0      & -1    & 0 \\
   0         & 0      & 0     & 1
\end{pmatrix}
\begin{pmatrix}
   1         & 0              & 0             & 0 \\
   0         & cos\theta_{x}  & -sin\theta_{x}& 0 \\
   0         & sin\theta_{x}  & cos\theta_{x} & 0 \\
   0         & 0              & 0             & 1
\end{pmatrix}  \begin{pmatrix}
   cos\theta_{y}  & 0        & sin\theta_{y}  & 0 \\
   0              & 1        & 0              & 0 \\
   -sin\theta_{y} & 0        & cos\theta_{y}  & 0 \\
   0              & 0        & 0              & 1
\end{pmatrix}  \begin{pmatrix}
   cos\theta_{z}  & -sin\theta_{z}  & 0       & 0 \\
   sin\theta_{z}  & cos\theta_{z}   & 0       & 0 \\
   0              & 0               & 1       & 0 \\
   0              & 0               & 0       & 1
\end{pmatrix}  \begin{pmatrix}
   1         & 0        & 0        & t_{x} \\
   0         & 1        & 0        & t_{y} \\
   0         & 0        & 1        & t_{z} \\
   0         & 0        & 0        & 1
\end{pmatrix}
$$
代入数据得:
$$V = 
\begin{pmatrix}
   1         & 0      & 0     & 0 \\
   0         & 1      & 0     & 0 \\
   0         & 0      & -1    & 0 \\
   0         & 0      & 0     & 1
\end{pmatrix}
\begin{pmatrix}
   1         & 0              & 0             & 0 \\
   0         & \cos{-29.049}  & -\sin{-29.049}& 0 \\
   0         & \sin{-29.049}  & \cos{-29.049} & 0 \\
   0         & 0              & 0             & 1
\end{pmatrix}  \begin{pmatrix}
   \cos{-24.138}  & 0        & \sin{-24.138}  & 0 \\
   0              & 1        & 0              & 0 \\
   -\sin{-24.138} & 0        & \cos{-24.138}  & 0 \\
   0              & 0        & 0              & 1
\end{pmatrix}  \begin{pmatrix}
   \cos{23.288}  & -\sin{23.288}  & 0       & 0 \\
   \sin{23.288}  & \cos{23.288}   & 0       & 0 \\
   0              & 0               & 1       & 0 \\
   0              & 0               & 0       & 1
\end{pmatrix}  \begin{pmatrix}
   1         & 0        & 0        & 5.24 \\
   0         & 1        & 0        & -4.19 \\
   0         & 0        & 1        & -8.29 \\
   0         & 0        & 0        & 1
\end{pmatrix}
$$

&emsp;&emsp;同样，由于计算太过繁琐，我用 $V$ 来指代运算的结果，这是个4*4的矩阵。将物体的世界坐标乘以 $V$ 就可以得到观察空间下的坐标。

# 观察空间--->裁剪空间
&emsp;&emsp;这一步**并不是真正的投影(三维的游戏物体投影到屏幕的二维坐标)，只是为投影做准备**。真正的投影发生在屏幕映射的齐次除法中。裁剪空间的主要目的是判断顶点是否在可见范围内（利用w分量划定了范围，实际上就是视锥体的范围）。当一个物体部分满足范围时，它将被**裁剪**；而当一个物体完全不满足范围时，它将被**剔除**。
## 视锥体
&emsp;&emsp;什么是视锥体？使用过unity相机，或者其他建模软件中的相机的人，在gizmos模式下点击相机的时候都能看到金字塔一样的线框（下左），或者长方形一样的线框（下右）：
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20210928213302.png" width=300/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20210928213535.png" width=365/>
&emsp;&emsp;其中，左图的金字塔结构就是**透视投影**下的视锥体，而右图是**正交投影**下的视锥体。二者的区别在于是否产生透视的效果，前者能够带来近大远小的物理上的真实感，而后者则没有这种效果，不论远近所有物体看起来都没有形变。啊哈，这不是某种程度上意味着透视适合3D，而正交适合2D吗？确实，这也是为什么我们创建一个新项目时，2D项目的相机默认是正交投影而3D项目默认是透视投影的原因。
&emsp;&emsp;首先，我们来了解一下视锥体的几个基本参数！

// 插入手绘示意图

&emsp;&emsp;除了上面标注出来的视场(FoV)、近裁平面高度(nearClipPlaneHeight)、远裁平面高度(farClipPlaneHeight)、近裁面距离(Near)、远裁面距离(Far)之外，我们要怎么知道宽度信息呢？比如说近裁平面宽度什么的，但是我们的横截面只能告诉我们高度信息。为了解决这个问题，我们引入了一个新的量叫做横纵比(Aspect，顾名思义是横和纵的比值)：
$$Aspect = \frac{nearClipPlaneWidth}{nearClipPlaneHeight}$$
$$Aspect = \frac{farClipPlaneWidth}{farClipPlaneHeight}$$

## 投影矩阵
### 透视投影
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20210928215534.png" width=400/>

&emsp;&emsp;通过上述的参数（我们可以直接在Unity中设置，如上图）以及Camera.aspect更改/获取的横纵比，我们可以推导出以下投影矩阵 $P$
$$P = 
\begin{pmatrix}
   \frac{\cot_{\frac{FOV}{2}}}{Ascept}    & 0                       & 0                                 & 0 \\
   0                                      & \cot_{\frac{FOV}{2}}    & 0                                 & 0 \\
   0                                      & 0                       & -\frac{Far + Near}{Far - Near}    & -\frac{2 \cdot Near \cdot Far}{Far - Near} \\
   0                                      & 0                       & -1                                & 0
\end{pmatrix}
$$
&emsp;&emsp;这是一个恒定的公式，推导过程挺复杂的，这里不细说（之后有空感兴趣研究的话再补充）。将投影矩阵乘上点/向量的观察空间坐标之后我们得到了裁剪空间坐标，现在我们要开始判断这个东西应不应该被剔除了。
&emsp;&emsp;计算完成后，我们得到了结果$\begin{pmatrix}x\\y\\z\\w\end{pmatrix}$。当三个分量x、y、z的范围都在±w范围内时，我们认为该点位于裁剪空间内，即：
$$-w \leqslant x \leqslant w$$
$$-w \leqslant y \leqslant w$$
$$-w \leqslant z \leqslant w$$

### 正交投影
&emsp;&emsp;我们知道，正交投影常用于2D游戏，因为不论远近同一个物体看起来都是一样大的。同样的，我们可以在检查器中调整视锥体的相关参数。
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20210928221640.png" width=400/>

&emsp;&emsp;因为正交投影的近裁面和远裁面大小一致，所以我们不再有近裁平面高度这样的参数，而是用Size代替。Size的大小为原本近裁平面高度的一半。正交投影的 $P$ 矩阵如下：
$$P = 
\begin{pmatrix}
   \frac{1}{Ascept \cdot Size}    & 0                 & 0                        & 0 \\
   0                              & \frac{1}{Size}    & 0                        & 0 \\
   0                              & 0                 & -\frac{2}{Far - Near}    & -\frac{Far + Near}{Far - Near} \\
   0                              & 0                 & 0                        & 1
\end{pmatrix}
$$
&emsp;&emsp;计算完成后，我们得到了结果$\begin{pmatrix}x\\y\\z\\w\end{pmatrix}$。同样的，当三个分量x、y、z的范围都在±w范围内时，我们认为该点位于裁剪空间内，即：
$$-w \leqslant x \leqslant w$$
$$-w \leqslant y \leqslant w$$
$$-w \leqslant z \leqslant w$$

# 裁剪空间--->屏幕空间
&emsp;&emsp;已经经过MVP矩阵的变换了，现在离屏幕已经非常近了。处于裁剪空间中的物体依然是三维的（位于视锥体内），现在我们需要使用**齐次除法**(Homogeneous Devision)将视锥体变成一个正方体。
&emsp;&emsp;什么是齐次除法？这个名字听起来仿佛深渊沼泽里的骇人美人鱼。实际上这个降维投影的操作非常简单，只需要对物体的x、y、z分量分别除以w，即投影后物体的坐标为：
$$\begin{pmatrix}\frac{x}{w}  \\
                 \frac{y}{w}  \\
                 \frac{z}{w}
\end{pmatrix}$$
&emsp;&emsp;最后，进行屏幕映射（pixelWidth即像素宽度，指的是屏幕分辨率中的宽。我们也可以通过Unity设置分辨率来实现不同分辨率下的屏幕映射结果）：
$$screen_{x} = \frac{clip_{x} \cdot pixelWidth}{2 \cdot clip_{x}} + \frac{pixelWidth}{2}$$
$$screen_{y} = \frac{clip_{y} \cdot pixelHeight}{2 \cdot clip_{y}} + \frac{pixelHeight}{2}$$
&emsp;&emsp;这个坐标$(screen_{x}, screen_{y})$就是最终投影在屏幕上的像素坐标，z坐标不被处理，它将被用于深度缓冲等模块。

