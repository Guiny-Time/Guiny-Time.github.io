---
tags: ['TA','Unreal','技术美术','shader','demo']
title: UE4 Materials 101：雨水系列4-雨水涟漪
created: '2022-01-20'
categories: UE4-Materials101
cover: https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220128001537.png
---

&emsp;&emsp;这几天摸了，周六周天去乡下卡溜，周一和同学上街卡溜，周二去跑步，周三网上买了一把TTC红轴的机械，打算做为入坑键盘玩一玩。刚到家就被家里的史前积灰问候了，虽然我也不知道家里一堆灰从哪来的。。。大概可以算作我家的七大灵异事件之一了，另外六大我也不知道是哪些就是了。因为没有去实体店试过其他轴体我也不好说怎么样，个人用起来还可以吧，不过少了一些打击感，不够爽（可能我更适合青轴之类的吧，没试过都不好说，不过福州哪里有线下键盘体验店啊（话说红轴是还没按到低就会触发，行程比较短吧，对打字还是比较友好的2333
&emsp;&emsp;那今天不摸了，我们要来实现的效果是咕了很久的雨水涟漪（以一种同心圆的形式向外扩散的水波）：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/%E6%B6%9F%E6%BC%AA.gif"/>

&emsp;&emsp;其实我经常在想壁纸引擎的一些效果是不是用shader做出来的，因为有很多相似的地方，比如灵梦壁纸的涟漪（左）、一些背景的扭曲效果（右）：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/%E7%81%B5%E6%A2%A6.gif" style="display:inline" width=269/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/%E5%A3%81%E7%BA%B8.gif" style="display:inline" width=270/>

&emsp;&emsp;首先，照样是从Ben神那里获取素材，这次有三个素材，第一个是用于蒙版作用的圈圈，第二个是包含了第一张图的蒙版（R通道）的法线，第三个是包含了众多图二的无规则的图：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220127152106.png" style="display:inline" width=100/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220127211918.png" style="display:inline" width=100/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220127222902.png" style="display:inline" width=100/>

# 实现扩张的同心圆
&emsp;&emsp;实现水波动画的第一步：我们需要实现同心圆的效果：
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220127154223.png" width=400/>

&emsp;&emsp;Frac节点将时间限制在了0~1的循环之中，实际上我们直接用这个“循环时间”和蒙版相乘就能得到近似扩张的效果，但是效果不够真实，从数学意义上来说其实是纹理整体慢慢**变亮**而不是扩散。
&emsp;&emsp;那要怎么做呢？我们可以先把循环时间减去1得到范围在[-1,0]的新的循环时间，再加上蒙版的值。这样的结果是慢慢加起来的，由此可以获得向外**扩张**的效果。通过这种方式（左）得到的结果与直接将循环时间与纹理相乘得到的（右）相比较如下所示：
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/%E6%B6%9F%E6%BC%AA1.gif" style="display:inline" width=230/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/%E6%B6%9F%E6%BC%AA2.gif" style="display:inline" width=230/>

&emsp;&emsp;接下来我们要把统一的圆进行分割，成为“波纹”。我们可以引入sin函数：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220127163057.png"/>

> **clamp节点**
&emsp;&emsp;clamp节点的作用是把一个值限制在一个上限和下限之间。当值溢出最小值和最大值的范围时，在最小值和最大值之间选择一个值使用。默认情况下，往上越界时使用max值，往下越界时使用min值。

&emsp;&emsp;好！难点来了。这里的三个节点的作用分别是：控制环的粗细、控制环的数量、成环。如果没有sin节点，前面的clamp节点将失去视觉上的作用（还是有的，但不明显），但mul节点依然体现在边缘的锐度上（边缘的值乘完变得很大甚至溢出，这就体现在锐化边缘上了）。
这时候整个白色圆圈的情况其实是：内部的值全是3，边缘呈现0~3的过渡，sin节点对这些过渡进行处理，有多少个波峰就形成多少个白色圈。
- mul节点控制的是过渡范围的大小，mul的乘数越大，过渡的范围越窄，对应的白圈宽度越窄。
- clamp节点控制同心圆的圈数。clamp的max值越大，其中的波峰就越多。
> **小插曲：关于UE4中的sine**
&emsp;&emsp;我遇到了一个史前难题，我在试图理解为什么会形成相近的环时被sin节点整的快吐血，因为我发现输入为PI时节点的输出竟是白色。正当我质疑这个世界的三角函数法则是不是出了什么问题的时候我发现：
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220127174836.png" width=350/>
&emsp;&emsp;焯。这也是为什么当我们用saturate节点将范围钳制在[0, 1]的时候还能获得完整的周期的原因，针对上图的“为什么这样设计”的答案我想也是这个。在sine节点的详细页面中也有一个周期反映了sine节点的周期，我们可以根据需要去手动的修改它。惭愧，居然之前都没发现sin是这样的sin。

# 添加法线
&emsp;&emsp;解压法线的方法和之前的做法一样，将存储在G和B通道中的信息取出解压、映射、与范围相乘即可。完成之后去除掉base color，就能得到下图左的效果：
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220127212836.png" style="display:inline" width=220/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220127221658.png" style="display:inline" width=250/>
&emsp;&emsp;然后，我们可以在这个基础上加上粗糙度（就像之前一样！），如上图右所示。

# 加上更多的涟漪
&emsp;&emsp;现在我们要让第三张图发挥作用了。将textureSample节点的贴图进行替换，我们就可以得到众多大小不同的涟漪：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220127225002.png"/>

&emsp;&emsp;但现在所有的涟漪是同时产生同时消失的，效果很糟。现在，我们就需要用到第三章贴图的A通道来错开时间了（就像我们之前对雨滴和水流所做的那样）：

<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220127225923.png"/>

&emsp;&emsp;但这还不够。如果一个玩家观察表面一段时间，他就会发现所有的涟漪都没有交集，并且互不影响，还是有序出现的（比较周期摆在那里）。我们可以把整个过程复制一遍，每一个独立的过程加上不一样的偏移。为了界面美观，我们可以把过程封装在函数里（同时也方便之后调用）。这样，我们就完成了整个效果。

