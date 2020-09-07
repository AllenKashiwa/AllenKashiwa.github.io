---
layout:     post
title:      "【GDC精讲】GDC2016_Eiserloh_Squirrel_JuicingYourCameras"
date:       2020-09-05 00:14:28
---

# **GDC精讲**
这个系列会精讲一些我关注的GDC演讲，主题的选取全凭兴趣。如果你有特别感兴趣的主题，欢迎私信我。

# **资源**
[原演讲地址](https://www.youtube.com/watch?v=tu-Qe66AvtY&t)

[PDF讲义](http://www.mathforgameprogrammers.com/gdc2016/GDC2016_Eiserloh_Squirrel_JuicingYourCameras.pdf)

[PPT（推荐）](http://www.mathforgameprogrammers.com/gdc2016/GDC2016_Eiserloh_Squirrel_JuicingYourCameras.pptx)

# **大纲**

- Camera Shake（相机抖动）
  - Translational vs. Rotational （位移与旋转）
  - Noise vs Random （噪声与随机）

- Smoothed motion（平滑运动）
  - Parametric motion*（参数运动）
  - Asymptotic Averaging（渐进平均）
  - Asymmetric Asymptotic Averaging(不对称渐进平均)

- Framing（取景构图）
  - Points of focus（焦点）
  - Points of interest（兴趣点）
  - Feathering

- Voronoi split-screen（Voronoi分屏）
  - N-way Voronoi split-screen cameras（多路Voronoi分屏相机）

# 前言

正式开始之前，演讲者介绍了另外两个关于提升游戏表现效果的演讲：
- [The art of screenshake（震屏的艺术）](https://www.youtube.com/watch?v=AJdEqssNZ-U)

- [Juice It or Lose It（带感为王）](https://www.youtube.com/watch?v=Fy0aCDmgnxg)

B站up主谜之声曾对这两个演讲做过中文介绍，推荐大家参考：https://www.bilibili.com/video/BV1Ds411v7Ux

# Camera Shake（相机抖动）

总的来说，相机抖动就像是盐，完全不放盐，菜就没味道，放多了呢又齁得慌。

- 没内味
![没内味](http://baizihan.me/assets/images/in-post/juicing_your_camera/boring.gif)

- 太过了
![太过了](http://baizihan.me/assets/images/in-post/juicing_your_camera/crazy.gif)

因此，我们需要相机抖动，并且能够控制抖动强度。

我们用一个0~1的变量trauma来决定相机抖动的程度。在战斗时受伤或者放技能等情况下可以增加trauma值。trauma值随着时间一直**线性衰减**。相机抖动的程度就由trauma来决定，但不是直接用，而是trauma的平方值或立方值，这样相机抖动的程度变化就是一条曲线。例如当前是0.9的trauma，那相机抖动的程度就是0.9^3=0.729。

- ![y=x^3](http://baizihan.me/assets/images/in-post/juicing_your_camera/y_cubed_x.png)

这样取值更接近现实生活中可能会发生的视角震动，例如行驶崎岖道路的车辆由于弹簧和减震器造成的抖动。另一个好处是能让玩家感受到震动的强度是逐步升级的。

## Translational vs. Rotational （位移与旋转）

相机抖动有两种形式，一是位移，二是旋转。这两种方式在2D游戏和3D游戏中给玩家的感受是不同的。

在2D中位移和旋转结合会更带感。而3D中由于景深的关系用旋转抖动效果已经很明显也不用担心位移抖动带来的穿模问题。

## Noise vs Random （噪声与随机）

在具体实现抖动时，2D与3D的代码是类似的：

- 2D

![2D](http://baizihan.me/assets/images/in-post/juicing_your_camera/2dimplementation.png)

- 3D
![3D](http://baizihan.me/assets/images/in-post/juicing_your_camera/3dimplementation.png)

而相比起随机，使用噪声是一种更好的方式：

![使用噪声](http://baizihan.me/assets/images/in-post/juicing_your_camera/use_noise.png)

使用噪声的好处是：
- 感觉抖动效果更连贯，而不是像随机那样很跳脱。
- 天然支持暂停和类似子弹时间的减速效果。
- 可以调节频率。（比如减慢之后可以搞出类似手持拍摄的效果）
- 只要使用同样的seed，可以很方便的复现抖动效果与回放。

总结一下相机抖动：

![使用噪声](http://baizihan.me/assets/images/in-post/juicing_your_camera/takeaways.png)


# Smoothed motion（平滑运动）

## Parametric motion*（参数运动）

通常在游戏中，我们让相机跟随主角的移动来探索丰富多彩的游戏世界。而主角的移动一般没啥规律，可能会很生硬。因此常常采用渐进平滑的方式让相机跟随主角。

想要平滑运动，最好的做法是使用三次厄尔密曲线(cubic Hermite curves)。参见[另一场GDC演讲](https://www.gdcvault.com/play/1018178/Math-for-Game)。

这里介绍一种更简单的方式：渐进平均。

## Asymptotic Averaging（渐进平均）

想象你面前有一块蛋糕，你刀法一流，气沉丹田一刀下去蛋糕被你均匀的一份为二。你调整姿势对其中一半再一刀下去，这次又是均匀的一份为二。你什么时候才能将蛋糕切完呢？答案是你永远也切不完，每次蛋糕都剩下上一次结果的一半大小。这就是渐进平均。

我们将之运用于相机的运动，让相机每帧都按一个权重，更接近我们期望的位置。如此一来，相机就会缓慢接近我们的目标位置而显得平滑优雅。而权重的大小决定了我们渐进的快慢。

![渐进平均](http://baizihan.me/assets/images/in-post/juicing_your_camera/asymptotic_average.png)


## Asymmetric Asymptotic Averaging(不对称渐进平均)

我们可以在水平方向和垂直方向上使用不同的渐进权重。水平方向可以更快的跟随，垂直方向上可以稍微慢些。而垂直方向上，向上和向下也可以使用不同的渐进权重。向上比较缓慢，主角跳跃等操作相机不会马上跟随可以更好的观察场景；而向下可以更快的跟随以免在主角掉落时让主角出画。

我们也可以改变渐进权重获得更丰富的效果。

# Framing（取景构图）

除了相机的位置，我们还需要关注最后呈现给玩家的构图。

## Points of focus（焦点）

焦点是那些需要一直吸引玩家注意力的要素。

  - 主要焦点，例如主角，除了过程动画时基本不会出画。
  - 次要焦点，例如锁定的敌人，也要尽量不出画。

## Points of interest（兴趣点）

场景中的兴趣点是一些游戏过程中玩家可能会想关注的点，但不需要一直保持玩家的注意力，例如关卡的宝箱。兴趣点会通过改变相机焦距或位移的方式让玩家转移注意力。

利用兴趣点可以巧妙的突出许多东西：
  - 敌人
  - 搜刮物
  - 按钮和操作杆
  - 密室之门
  - 陷阱
  - 关卡设计的标签

## Soft and fuzzy 柔和与模糊

为了避免兴趣点突然改变，我们要设计一套算法找到当前最重要的兴趣点。

通常，计算每个兴趣点的“接近度”:
- 阈值之外的那些具有接近度0
- 内部阈值内的那些具有接近度1
- 那些在内部和外部阈值之内的值约为[0,1]
- 每个兴趣点的权重=接近度*重要性

## multiple primary focus points(多焦点)

真正的难题是多焦点的情况。典型的例子比如四人同屏游戏。

如何处理这种多焦点的情况呢？可能的方案是：
- 如果一人落后则屏幕无法推进关卡
- 玩家可以出画
- 玩家出画就挂掉
- 玩家被拉回来
- 玩家可以“拖动”屏幕（和其他玩家）
- 缩小以涵盖所有人
- 分屏

除了分屏之外，其他方案都会影响游戏玩法。如果不是特殊设计，通常使用分屏来解决多焦点的情况。

但是分屏让人难受的是要放弃50%（双人同屏）或75%（四人同屏）的屏幕资源。而且如果是合作（而非对抗）的游戏，玩家大部分时间是待在一起的，这样就浪费了宝贵的屏幕资源。

# Voronoi split-screen

[Voronoi](https://zh.wikipedia.org/wiki/沃罗诺伊图)分屏相机仅在需要分屏时才分屏，可以有效解决浪费屏幕资源的问题。

![Voronoi分屏相机](http://baizihan.me/assets/images/in-post/juicing_your_camera/voronoi_split_screen_cameras.gif)

可以看到A和B两个玩家，在足够近的时候是共享屏幕的，只有在距离较远的时候才会分屏，并且分屏的方向是两个玩家位置决定的。

整个过程是这样

- 首先判断玩家是否可以只显示在一个屏幕上

![共享屏幕](http://baizihan.me/assets/images/in-post/juicing_your_camera/shared.png)

- 如果不能共享屏幕，则计算屏幕空间Voronoi边界

![计算边界](http://baizihan.me/assets/images/in-post/juicing_your_camera/cant_share.png)

- 根据距离平衡各自的屏幕空间
![平衡各自屏幕空间](http://baizihan.me/assets/images/in-post/juicing_your_camera/balance_private_space.png)

- 合并结果
  - 算法1:单独渲染，然后缝制
    - 这种方法分割和相对位置都是真实的。玩家沿分割线对称。好处是合屏时是朝着另外的玩家移动也是朝着分割线移动。坏处是玩家不在各自子屏幕的中心。
![先渲染再缝制](http://baizihan.me/assets/images/in-post/juicing_your_camera/stitch.png)

  - 算法2:以玩家为中心
    - 这种方法分割是真实的，但相对位置不是。玩家沿屏幕中心对称。好处是玩家在各自子屏幕的中心。坏处是合屏时得朝着分割线移动，而不是朝着另外的玩家移动。
![以玩家为中心](http://baizihan.me/assets/images/in-post/juicing_your_camera/recenter.png)


需要注意的是，平衡合并和分离之间的过渡至关重要
  - 超出outer distance，视图完全分开
  - 在inner distance内，视图将完全合并
  - 在outer与inner之间混合（淡入淡出）每个视图以收敛到合并的视图。

## N-way Voronoi split-screen cameras（N路Voronoi分屏相机）

我们已经知道双人分屏用voronoi分屏相机是怎样的了。那多人又如何分屏？

- 假设有N个玩家
![N个玩家](http://baizihan.me/assets/images/in-post/juicing_your_camera/n_players.png)

- 获取他们的相对位置
![相对位置](http://baizihan.me/assets/images/in-post/juicing_your_camera/relative_displacement.png)

- 找到两两之间的中点和法线
![两两分割](http://baizihan.me/assets/images/in-post/juicing_your_camera/midway_point.png)

- 找到每对玩家划分区域的边界
![区域边界](http://baizihan.me/assets/images/in-post/juicing_your_camera/boundary_edges.png)

- 创建世界空间边界的凸包
![创建凸包](http://baizihan.me/assets/images/in-post/juicing_your_camera/convex_hulls.png)

- 得到世界空间的voronoi区域
![世界空间的voronoi区域](http://baizihan.me/assets/images/in-post/juicing_your_camera/world_space_voronoi_regions.png)

- 用世界空间的voronoi区域决定各子屏幕上的voronoi区域的形状
![屏幕形状由voronoi区域决定](http://baizihan.me/assets/images/in-post/juicing_your_camera/private_shape.png)

- 利用stencil分开渲染
![分开渲染](http://baizihan.me/assets/images/in-post/juicing_your_camera/render_separately.png)

- 合并进视图
![合并视图](http://baizihan.me/assets/images/in-post/juicing_your_camera/composite.png)


最后，演讲者还列出了一些相关内容供大家参考：
- [Scroll Back: The Theory and Practice of Cameras in Side-Scrollers](https://www.gdcvault.com/play/1022243/Scroll-Back-The-Theory-and)
- [Fast and Funky 1D Nonlinear Transforms (GDC 2015)](https://www.gdcvault.com/play/1022142/Math-for-Game-Programmers-Fast)
- [Random Numbers (GDC 2014)](https://www.gdcvault.com/play/1020648/Math-for-Game-Programmers-Random)
- [Interpolations and Splines (GDC 2012)](http://www.essentialmath.com/GDC2012/GDC12_Eiserloh_Squirrel_Interpolation-and-Splines.ppt)

如果你喜欢这个系列可以扫描下面的二维码关注我的公众号：

![Unity与图形学](http://baizihan.me/assets/images/qrcode.jpg)
