---
title: "在 Colab 搭建 Stable Diffusion"
date: 2023-02-22T15:49:06+08:00
draft: false
tags: ["Ai 绘画", "Stable Diffusion"]
categories: ["工作"]
toc: false
---

<div align="center">
<img src=https://imgoldjii.oss-cn-beijing.aliyuncs.com/sdsm.png width=600 />
</div>

Form [Weird Wonderful Ai Art](https://weirdwonderfulai.art/resources/disco-diffusion-70-plus-artist-studies)

<!--more-->

> 2023 年 9 月 23 日更新：Colab 官方禁用 Stable Diffusion WebUI，无法解决断网问题，本文方法作废。详情见 https://decrypt.co/197428/google-colab-stable-diffusion-web-ui-ban

## 🎨 Ai 绘画
Stable Diffusion 是 stability.ai 于 2022 年发布的深度学习文生图模型，可以在大多数配备有适度 GPU 的电脑硬件上运行。

Colaboratory 简称 Colab，是 Google Research 团队发布的一种托管式 Jupyter 笔记本服务，可以免费使用 GPU / TPU 计算资源。

这是一份保姆级的在 Colab 搭建 Stable Diffusion 服务的教程，按照以下步骤操作，即可浅尝 Ai 绘画的魅力。

## 分配资源
进入 [Stable_Diffusion_WebUi_Altryne](https://colab.research.google.com/github/altryne/sd-webui-colab/blob/main/Stable_Diffusion_WebUi_Altryne.ipynb) Colab 工程。

点击右上角的**连接**获取云主机资源并连接运行时。

![1-image-20230222142539642](https://imgoldjii.oss-cn-beijing.aliyuncs.com/1-image-20230222142539642.png)

运行 1.0 单元格中的命令可以获主机信息。

![1-image-20230222142936172](https://imgoldjii.oss-cn-beijing.aliyuncs.com/1-image-20230222142936172.png)

一般分配到的都是 Tesla T4 机型，如果人品欧分配到 V100，那可以好好的吹一波了。

## 配置模型
首先注册 [Hugging Face](https://huggingface.co/) 账号，并生成 WRITE 类型的 [Access Tokens](https://huggingface.co/settings/tokens)。

![1-image-20230222144043327](https://imgoldjii.oss-cn-beijing.aliyuncs.com/1-image-20230222144043327.png)

返回到 Colab 界面，展开 *1.4 Connect to Google Drive* 单元格，勾选 download_if_missing 并输入 Hugging Face 的 Accsee Token。

![1-image-20230222144232989](https://imgoldjii.oss-cn-beijing.aliyuncs.com/1-image-20230222144232989.png)

## 设置密码
在 *2.1 Optional - Set webUI settings and configs before running* 中配置密码。

![1-image-20230222144551905](https://imgoldjii.oss-cn-beijing.aliyuncs.com/1-image-20230222144551905.png)

## 运行服务
将所有单元块收起，依次运行所有单元块部署服务即可。

运行耗时
- Setup stage：5min
- Run the Stable Diffusion webui：1s
- Launch WebUI for stable diffusion：1min

![1-image-20230222144857944](https://imgoldjii.oss-cn-beijing.aliyuncs.com/1-image-20230222144857944.png)

运行过程中会弹窗请求登录 Google Drive，正常登录并允许即可。

![1-image-20230222145720360](https://imgoldjii.oss-cn-beijing.aliyuncs.com/1-image-20230222145720360.png)

运行完成点击 Public URL 进入 Stable Diffusion WebUI 界面。账号为 wubei，密码为上面自己配置的。

![1-image-20230222150541478](https://imgoldjii.oss-cn-beijing.aliyuncs.com/1-image-20230222150541478.png)

然后就可以尽情的享用了。

输入 prompt，即可生成对应的图像，比如输入 “ an astronaut in the water ”。

![1-image-20230222151132661](https://imgoldjii.oss-cn-beijing.aliyuncs.com/1-image-20230222151132661.png)

当然，你可以输入更多的 prompt 信息，来生成更加具体准确的图像。

> (((masterpiece))),((best quality)), flat chest,((loli)),((one girl)),very long light white hair, beautiful detailed red eyes,aqua eyes,white robe, cat ears,(flower hairpin),sunlight, light smile,blue necklace,see-through, 
>
> 杰作，最佳品质，贫乳，萝莉，1个女孩，很长的头发，淡白色头发，红色眼睛，浅绿色眼睛，白色长裙，猫耳，发夹，阳光下，淡淡的微笑，蓝色项链，透明

![1-image-20230222154010936](https://imgoldjii.oss-cn-beijing.aliyuncs.com/1-image-20230222154010936.png)
