---
title: 解决Z-Fighting
date: '2022-01-12'
tags: ['Unity','ShaderLab']
categories: 探索发现
cover: https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220113154252.png
---

# 解决Z-Fighting
Z-Fighting是在实际项目中困扰到场景布局的一大因素。从理论上来说，在相同深度值的地方有多个片元，这导致颜色缓冲区不知道该呈现哪个颜色比较合适，而当视角运动时就会出现糟糕的闪烁：
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220113143251.png" width=315/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220113143402.png" height=300/>

通过调整渲染队列并不能避免这种问题，就比如上图的半透明物体的渲染队列是在不透明物体之后的，但他们重叠时依然会发生z-fighting现象。

## 解决方法
Shaderlab语法中的Offset命令
```ShaderLab
Properties{
        _OffsetFactor("深度斜率",Float) = 0
        _OffsetUnit("深度偏移",Float) = 0
    }
SubShader{
    Pass{
        Tags{ "LightMode" = "ForwardBase" }
        Offset [_OffsetFactor],[_OffsetUnit]
        }
    }
```

### 参数解读
- **_OffsetFactor**: 缩放最大 Z 斜率，也称为**深度斜率**，以生成每个多边形的可变深度偏移。
不平行于近剪裁平面和远剪裁平面的多边形具有 Z 斜率。调整此值以避免此类多边形上出现视觉瑕疵。
- **_OffsetUnit**: 缩放最小可分辨深度缓冲区值，以产生恒定的**深度偏移**。最小可分辨深度缓冲区值（一个 _unit_）因设备而异。
负值意味着 GPU 将多边形绘制得更靠近摄像机。正值意味着 GPU 将多边形绘制得更远离摄像机。

### 效果
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220113152924.png" width=328/><img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220113153033.png" width=315/>
