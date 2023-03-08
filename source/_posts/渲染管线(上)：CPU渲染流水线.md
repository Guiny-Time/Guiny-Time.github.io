---
tags: ['TA','Unity Shader','shader','渲染管线']
title: 渲染管线(上)：CPU渲染流水线
date: '2022-11-01T06:30:17.165Z'
cover: https://yanxuan.nosdn.127.net/c20bb329b08268650721dda0c2ef0b71.jpg
---

我们总在说渲染管线，几乎每一个与TA有关的课程第一课都和渲染管线相关，这是一切图像的基础。就拿最近在用的Unity内置渲染管线为例吧，其中包含了CPU和GPU两个部分，总的流程图如下所示：

<img src="https://tvax2.sinaimg.cn/large/006UcwnJly1h7b0cqkq9hj31gd0r4k7m.jpg" alt="image" width="1885" data-width="1885" data-height="976">

<img src="https://tvax3.sinaimg.cn/large/006UcwnJly1h7oi3ilns2j30pz0j7tep.jpg" alt="image" width="635" data-width="935" data-height="691">

应用阶段的处理场所在CPU。这一阶段主要处理以下几件事：
- 准备好场景数据
- 剔除不需要渲染的部分
- 设置渲染顺序
- 加载数据到显存
- 调用DrawCall
- **输出渲染图元**(Rendering Primitives)作为几何阶段的输入。通俗来讲，渲染图元可以是点、线、三角面等，这些信息会传递给GPU渲染管线处理。

## **剔除(Culling)**
CPU阶段的剔除是**粗粒度**的，和GPU阶段的裁剪不同。粗粒度剔除发生在物体层面，剔除是以物体1为单位进行的。这一步包含了三种剔除：
1. 视锥体剔除
视锥体剔除指的是剔除掉相机视锥体之外的元素，视锥体即相机的可视范围，全部在外的物体(比如下图的鸟)不需要渲染，部分在外的物体会在GPU渲染部分被裁剪。
<img src="https://tvax3.sinaimg.cn/large/006UcwnJly1h7ojvby2awj309a06u0tl.jpg" alt="image" width="334" data-width="334" data-height="246">

2. 层级剔除
层级剔除指物体所处的层(Layer)是否被相机所渲染。如果相机的渲染层不包含该物体的layer，则该物体不被渲染。譬如在上文的例子中，如果不渲染树木层，就会是下面的画面：
<img src="https://tvax1.sinaimg.cn/large/006UcwnJly1h7oi84qvyaj30gh0aitd2.jpg" alt="image" width="403" data-width="593" data-height="378">

3. 遮挡剔除
遮挡剔除指当一个物体被其他物体遮挡而不在摄像机的可视范围内时，不对其进行渲染。对这些物体进行渲染是浪费时间和性能的，所以要先剔除掉这部分元素以节省接下来的时间。

## 设置渲染顺序
渲染顺序主要由**渲染队列(Render Queue**)决定的，序号越小的越先渲染。Unity默认渲染队列序号如下：
- Background（1000）
  最早被渲染的物体的队列。
- Geometry（2000）
  不透明物体的渲染队列。大多数物体都应该使用该队列进行渲染，也是Unity Shader中默认的渲染队列。
- AlphaTest（2450）
  有透明通道，需要进行Alpha Test的物体的队列，比在Geomerty中更有效。
- Transparent（3000）
 半透物体的渲染队列。一般是不写深度的物体，Alpha Blend等的在该队列渲染。
- Overlay（4000）
 最后被渲染的物体的队列，一般是覆盖效果，比如镜头光晕，屏幕贴片之类的

 当渲染队列相同时，**不透明队列**(RenderQueue < 2500)，根据摄像机距离**从前往后排序**，这样先渲染离摄像机近的物体，远处的物体被遮挡剔除；**半透明队列**(RenderQueue > 2500)，根据摄像机距离**从后往前排序**，这是为了保证渲染正确性，例如半透明黄色和蓝色物体，不同的渲染顺序会出现不一样的颜色。

## 打包数据到显存
在CPU阶段，数据会先从硬盘加载到RAM中，随后网格、纹理等数据又被加载到显存上。加载到显存上之后，数据就会被移除。
之所以这么做是因为GPU访问显存的速度比访问前两者更快，而且多数显卡可能无法直接访问RAM。打包的数据主要包括：
1. **模型信息**，包括顶点坐标、法线、uv、切线、顶点颜色等等
2. **变换矩阵**V和P，这是由相机的位置和FOV决定的
3. **灯光信息**
4. 模型的**材质参数**，设置对应的渲染状态

## 调用SetPass Call和Draw Call
- SetPass Call
Shader脚本中一个Pass语义块就是一个完整的渲染流程，一个着色器可以包含多个Pass语义块，**每当GPU运行一个Pass之前，就会产生一个SetPassCall**，所以可以理解为调用一个完整渲染流程。
- DrawCall
CPU每次调用图像编程接口命令GPU渲染的操作称为一次Draw Call。Draw Call就是一次渲染命令的调用，它指向一个需要被渲染的图元（primitive）列表，不包含任何材质信息。GPU收到指令就会根据渲染状态（例如材质、纹理、着色器等）和所有输入的顶点数据来进行计算，最终输出成屏幕上显示的那些漂亮的像素。
