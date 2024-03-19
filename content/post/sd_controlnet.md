---
title: "Stable Diffusion 的光影艺术"
date: 2023-09-26T15:49:06+08:00
draft: false
tags: ["Stable Diffusion", "AICG"]
categories: ["工作"]
toc: false
---

最近在网上经常有看到类似上面这种图片，图片里的建筑物或者是光影形成了一些文字，给图片附加了更多的信息。评论有很多人很好奇，所以记录下制作方式。

<div align="center">
<img src=https://imgoldjii.oss-cn-beijing.aliyuncs.com/picgo/202310192146502.png width=100%/>
</div>
<!--more-->


其实制作的方法很简单，只是在基本 text2img 的基础上使用 ControlNet 进行垫图并附加一些光影性强的 Model 即可。

## Checkpoint 配置
- Checkpoint Model：推荐使用写实类的模型，比如 majicMIX realistic、dalcefoPainting 3rd。
- Sampler method：推荐使用 DMP++ 2S a Karras、DMP++ 2M Karras。
- Sampling Steps：推荐 25 左右，不要超过 30，因为要实现光影文字效果，Step 数太多会渲染掉文-字效果。


<div align="center">
<img src=https://imgoldjii.oss-cn-beijing.aliyuncs.com/picgo/202310192143384.png width=100%/>
</div>

## 底图制作
需要用到一张黑底白字的图片，用什么做随意，用你熟悉工具即可，尺寸需要跟你上面配置的 Size 信息保持一致，以保证文字的位置是准确的，比如下图。


<div align="center">
<img src=https://imgoldjii.oss-cn-beijing.aliyuncs.com/picgo/202310192150418.png width=100%/>
</div>


另外，如果你能让文字的边缘实现模糊效果就更好了，生成的图片光影文字效果会更贴合。

## ControlNet 配置
- ControlNet Model：对出图效果的影响比较关键，推荐使用 control_v1p_sd15_brightness、lightingBasedPicture_v10，都是比较优秀的光影 Model。
- ControlNet Type：全部
- 预处理器：推荐 none，也可以使用 tile_colorfix，但是要搭配 control_v11f1e_sd15_tile 模型。
- Control Weight：0.4 ～ 0.55
- Starting Control Step：0 ~ 0.1
- Ending Control Step：0.45 ～ 0.6


<div align="center">
<img src=https://imgoldjii.oss-cn-beijing.aliyuncs.com/picgo/202310192144711.png width=100%/>
</div>


最后就抽卡吧，慢慢抽，另外附上一张网友测试的 Control Step 与出图效果的对比图。


<div align="center">
<img src=https://imgoldjii.oss-cn-beijing.aliyuncs.com/picgo/202310192154764.webp width=100%/>
</div>
