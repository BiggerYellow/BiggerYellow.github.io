Freelancer Jekyll theme  [![Build Status](https://api.travis-ci.org/jeromelachaud/freelancer-theme.svg?branch=master)](https://travis-ci.org/jeromelachaud/freelancer-theme/) 
=========================

Jekyll theme based on [Freelancer bootstrap theme ](http://startbootstrap.com/template-overviews/freelancer/)

## How to use
 - Place a image in `/img/portfolio/`
 - Replace `your-email@domain.com` in `_config.yml` with your email address. Refer to [formspree](http://formspree.io/) for more information.
 - Create posts to display your projects. Use the follow as an example:
```txt
---
layout: default
modal-id: 1
date: 2020-01-18
img: cabin.png
alt: image-alt
project-date: January 2020
client: The Client
category: Web Development
description: The description of the project

---
```

## Demo
View this jekyll theme in action [here](https://jeromelachaud.com/freelancer-theme)

## Screenshot
![screenshot](https://raw.githubusercontent.com/jeromelachaud/freelancer-theme/master/screenshot.png)

---------
For more details, read the [documentation](http://jekyllrb.com/)
```

#### jekyll官网
https://jekyllrb.com/docs/structure/

#### 图片素材
https://www.pexels.com/zh-cn/
图片大小: 900*650

#### 数学公式网站
https://latex.codecogs.com/
https://demo.wiris.com/mathtype/en/developers.php#mathml-latex

### 创建分类
1. 在_includes中新建一个分类.html, 以spring.html为例
2. 在_post中新建一个spring文件夹用来存放spring相关的markdown文章,同时新建的文章需要加上spring标记 spring: true, 同时modal-id属性必须不一样
3. 在spring.html中填充内容,遍历所有spring为true的文章并设置标签<section id="spring">(注意此id最为重要需要与使用的地方保持一致),注意同时需要在_includes.css.main.css中设置标签为spring的样式
4. 在_includes.nav.html中添加sping滚动轴
    <li class="page-scroll">
        <a href="#spring">Spring</a>
    </li>
5. 在_layouts.default.html中添加新增的spring.html,注意这里如果想删除某一个栏目,不是使用注释掉,需要直接从文件中删除
    {% include spring.html %}

