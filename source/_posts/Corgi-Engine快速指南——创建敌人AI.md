---
title: Corgi Engine快速指南(2)——创建敌人AI
tags: ['游戏编程','经验分享','CorgiEngine','Unity']
categories: CorgiEngine
date: '2024-03-05 15:21:17'
copyright_author: 时光
cover: https://img11.360buyimg.com/ddimg/jfs/t1/232790/18/11682/42158/65e97ff7F73b91bdf/cf24a749a5f138be.jpg
---

{% note warning simple %}

这篇文章还没写完

{% endnote %}

经过上一篇文章的介绍，我们已经能够在引擎中实现玩家移动、跳跃、奔跑等一系列行为了。现在，我们要往场景中引入能够对玩家作出反应的敌人AI。

{% note info simple %}

本节目标：创建一个持有近战武器 (melee weapon) 的巡逻敌人“持剑战狗”，在侦察到玩家后进入警觉状态（停顿、头顶冒出感叹号），经过 1 秒开始追赶玩家，追上时使用近战攻击（具有前摇）
敌人的每次攻击会对玩家造成 1 点伤害。

{% endnote %}

# 持剑战狗
## 创建一把近战武器
我们可敬的敌人需要一把对付邪恶玩家的武器。
1. 创建一个新的空对象，将其命名为 **MeleeWeapon**，添加 *MeleeWeapon* 组件
2. 将此对象拖到项目中的文件夹中以制作预制件，然后从场景中删除它
3. 按照上一篇文章的方式创建一个敌人AI，将 MeleeWeapon 预制件拖到敌人AI的 *CharacterHandleWeapon* 脚本（如果没有，就新建一个）的 <u>InitialWeapon</u> 中
4. 现在我们可敬的敌人就拥有了一把武器！

<img src="https://img12.360buyimg.com/ddimg/jfs/t1/228590/10/11259/32720/65e6d17cF241581ef/c12a0e300086a18e.jpg" alt="image.png" title="InitialWeapon的位置" />

## 调整武器参数
在刚才添加的 *MeleeWeapon* 脚本中，设置以下参数：
- 触发模式(Trigger Mode)：半自动
- 攻击持续时间为 0.3（这是一次短攻击，基本上是一拳）。
- “造成的目标层伤害”蒙版设置为“玩家”和“敌人”
- 伤害：以及 10 点伤害。

<img src="https://img12.360buyimg.com/ddimg/jfs/t1/171866/16/41849/16852/65e6d329F64214b9e/eed45a300143b4a4.jpg" alt="image.png" title="image.png" />