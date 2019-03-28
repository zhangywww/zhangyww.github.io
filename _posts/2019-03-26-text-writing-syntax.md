---
title: jekyll的text主题写作语法
tags: reference
mathjax: true
lightbox: true
article_header:
  type: cover
  image:
    src: /screenshot.jpg
---

本文对jekyll主题中写作的一些语法格式进行介绍。
<!--more-->

## YAML Front Matter
YAML Front Matter是写在每个markdown文件头部的配置信息。配置信息说明如下：
```
# layout: [article]
# mode: [normal(default), immersive] 
# type: [webpage(default), article]
# key: !!str 关键字
# lang: [en(default), zh, zh-Hans, zh-Hant]
# author: [authors.yml中的值]
# show_title: [true(default), false]
# show_edit_on_github: [true, false(default)] 需要设置_config.yml中的repository和repository_tree
# show_date: [true(default), false]
# show_tags: [true(default), false]
# full_width: [true, false(default)]
# comment: [true(default), false]

# 数学引擎、图表引擎等
# mathjax: [true, false(default)]
# mathjax_autoNumber: [true, false(default)]
# mermaid: [true, false(default)]
# chart: [true, false(default)]

# cover: !!str coverImage的url
# header: [false, !!map] 设置成false为隐藏header
# article_header: !!map
# aside: !!map
# sidebar: !!map
# footer: [false] 设置成false隐藏footer
# lightbox: [true, false] 设置成true时可以单击浏览图片
```

通常使用如下格式
```
pageview: true
title: 文章标题
tags: 分类标签
# mathjax: true
# lightbox: true
article_header:
  type: cover
  image:
    src: /images/toolpath.png
```

## 数学公式
$$ P_i = a_i + b_i $$