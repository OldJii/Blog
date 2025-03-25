---
title: "Hugo 多图排版 even 主题适配"
date: 2025-03-24T17:49:06+08:00
draft: false
tags: ["hugo", "mathJax"]
categories: ["工作"]
toc: false
---
在使用 Hugo 搭建个人博客时，常常会遇到图片排版的问题。尤其是竖图和横图混合时，Hugo 默认的排版方式会让竖图左右留出大量空白，显得不美观。此外，Markdown 本身也不支持横向排列图片（除非使用内嵌 HTML，但这种方式存在诸多问题，我尝试后已放弃）。

之前我参考了一篇 [Hugo 多图排版的教程](https://immmmm.com/about-images-gird/)，但按照教程操作后并没有达到预期效果。评论区里也有其他用户遇到了类似问题，但原作者并未给出解答。

![image-20250325152613877](https://imgoldjii.oss-cn-beijing.aliyuncs.com/202503251526936.png)

经过一番检查和尝试，我发现问题出在主题生成静态页面时，会将 Markdown 中的图片用 `<a>` 标签包裹，以便实现点击预览大图的效果。这与原教程的 CSS 选择器不匹配，导致样式无法生效。下面是适配方案。

```css
/* 多图分栏样式 */
.post-content p:has(> a:nth-child(2)) {
  column-count: 2;
  column-gap: 8px;
  margin: 6px 0;
}
.post-content p:has(> a:nth-child(3)) { column-count: 3; }
.post-content p:has(> a:nth-child(4)) { column-count: 4; }
.post-content p:has(> a:nth-child(5)) { column-count: 5; }
.post-content p:has(> a:nth-child(6)) { column-count: 4; }

/* 处理图片显示 */
.post-content p:has(> a:nth-child(2)) a {
  display: block;
  margin-bottom: 8px;
  break-inside: avoid;
}

/* 针对6图模式调整间距 */
.post-content p:has(> a:nth-child(6)) a {
  margin-bottom: 8px;
}

/* 确保图片填充容器 */
.post-content p a img {
  width: 100%;
  height: auto;
  display: block;
}
```

对于 Even 主题，需要将上述代码追加到 `Blog/themes/even/assets/sass/_partial/_post/_content.scss` 文件中。请注意，不同 Hugo 主题的 `content` 文件目录可能会有所不同，具体路径需要根据你所使用的主题进行调整。

### 使用规则

使用规则与原教程一致。图片之间可以换行，但不能空行，这样 CSS 会根据 `<p>` 元素内的 `<img>` 标签数量自动调整分栏数量。例如：

```markdown
#### 更多图片横竖随机

![](https://cn.bing.com/th?id=OHR.Breckenridge_EN-US4460042968_768x1280.jpg)
![](https://cn.bing.com/th?id=OHR.BisonWindCave_EN-US4537340482_1280x768.jpg)
![](https://cn.bing.com/th?id=OHR.Umschreibung_EN-US4693850900_768x1280.jpg)
![](https://cn.bing.com/th?id=OHR.DonkeyFeast_EN-US1153850805_768x1280.jpg)
![](https://cn.bing.com/th?id=OHR.SessileOaks_EN-US1487454928_1280x768.jpg)
![](https://cn.bing.com/th?id=OHR.DonkeyFeast_EN-US1153850805_1280x768.jpg)
![](https://cn.bing.com/th?id=OHR.RumeliHisari_EN-US4800002879_1280x768.jpg)
![](https://cn.bing.com/th?id=OHR.InscriptionWall_EN-US1392173431_1280x768.jpg)
![](https://cn.bing.com/th?id=OHR.HummockIce_EN-US4606231645_768x1280.jpg)
```

### 效果如下

#### 两张

![](https://cn.bing.com/th?id=OHR.SessileOaks_EN-US1487454928_1280x768.jpg)
![](https://cn.bing.com/th?id=OHR.InscriptionWall_EN-US1392173431_1280x768.jpg)

#### 三张

![](https://cn.bing.com/th?id=OHR.DonkeyFeast_EN-US1153850805_1280x768.jpg)
![](https://cn.bing.com/th?id=OHR.RumeliHisari_EN-US4800002879_1280x768.jpg)
![](https://cn.bing.com/th?id=OHR.BisonWindCave_EN-US4537340482_1280x768.jpg)

#### 四张

![](https://cn.bing.com/th?id=OHR.Umschreibung_EN-US4693850900_768x1280.jpg)
![](https://cn.bing.com/th?id=OHR.HummockIce_EN-US4606231645_768x1280.jpg)
![](https://cn.bing.com/th?id=OHR.DonkeyFeast_EN-US1153850805_768x1280.jpg)
![](https://cn.bing.com/th?id=OHR.Breckenridge_EN-US4460042968_768x1280.jpg)

#### 更多图片横竖随机

同样会出现各列高度不一致的情况，但可以接受。

![](https://cn.bing.com/th?id=OHR.Breckenridge_EN-US4460042968_768x1280.jpg)
![](https://cn.bing.com/th?id=OHR.BisonWindCave_EN-US4537340482_1280x768.jpg)
![](https://cn.bing.com/th?id=OHR.Umschreibung_EN-US4693850900_768x1280.jpg)
![](https://cn.bing.com/th?id=OHR.DonkeyFeast_EN-US1153850805_768x1280.jpg)
![](https://cn.bing.com/th?id=OHR.SessileOaks_EN-US1487454928_1280x768.jpg)
![](https://cn.bing.com/th?id=OHR.DonkeyFeast_EN-US1153850805_1280x768.jpg)
![](https://cn.bing.com/th?id=OHR.RumeliHisari_EN-US4800002879_1280x768.jpg)
![](https://cn.bing.com/th?id=OHR.InscriptionWall_EN-US1392173431_1280x768.jpg)
![](https://cn.bing.com/th?id=OHR.HummockIce_EN-US4606231645_768x1280.jpg)