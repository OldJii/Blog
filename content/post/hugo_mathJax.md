---
title: "Hugo中支持LaTeX的数学表达式"
date: 2021-12-18T01:19:00+08:00
draft: false
tags: ["hugo", "mathJax"]
categories: ["工作"]
toc: false
---

Hugo 默认是不支持显示 LaTeX 风格的数学表达式的，Markdown 中的数学表达式语法在 Hugo 中默认并不会被识别。

在 Hugo 中引入 MathJax 即可解决该问题，[MathJax](https://www.mathjax.org) 是一个适用于所有浏览器的 JavaScript 数学显示引擎。

引入方法很简单，在文章的模版的 HTML 的中添加以下内容即可。

```javascript
<script type="text/javascript"
        async
        src="https://cdn.bootcss.com/mathjax/2.7.3/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
MathJax.Hub.Config({
  tex2jax: {
    inlineMath: [['$','$'], ['\\(','\\)']],
    displayMath: [['$$','$$'], ['\[\[','\]\]']],
    processEscapes: true,
    processEnvironments: true,
    skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
    TeX: { equationNumbers: { autoNumber: "AMS" },
         extensions: ["AMSmath.js", "AMSsymbols.js"] }
  }
});

MathJax.Hub.Queue(function() {
    // Fix <code> tags after MathJax finishes running. This is a
    // hack to overcome a shortcoming of Markdown. Discussion at
    // https://github.com/mojombo/jekyll/issues/199
    var all = MathJax.Hub.getAllJax(), i;
    for(i = 0; i < all.length; i += 1) {
        all[i].SourceElement().parentNode.className += ' has-jax';
    }
});
</script>

<style>
code.has-jax {
    font: inherit;
    font-size: 100%;
    background: inherit;
    border: inherit;
    color: #515151;
}
</style>
```

要注意的是需要确保该 HTML 必定被 Hugo 框架加载，我这里是添加到了 `layouts/partials/` 目录下的 `footer.html`文件中，因为我确定该文件必定包含在网站的每个页面中。

实际上单纯引入 MathJax 并不需要如此多行代码，多余的部分是为了解决 Markdown 和 LaTeX 中对于划线 _ 的不同定义带来的问题，详情可以参考 [这篇文章](https://www.gohugo.org/doc/tutorials/mathjax/) 。


$$
D(x) = \begin{cases}
\lim\limits_{x \to 0} \frac{a^x}{b+c}, & x<3 \\
\pi, & x=3 \\
\int_a^{3b}x_{ij}+e^2 \mathrm{d}x,& x>3 \\
\end{cases}
$$


$$
\lim_{x \to \infty} x^2_{22} - \int_{1}^{5}x\mathrm{d}x + \sum_{n=1}^{20} n^{2} = \prod_{j=1}^{3} y_{j}  + \lim_{x \to -2} \frac{x-2}{x}
$$


好了，开心的在 Hugo 中使用 LaTeX 吧。