---
layout:     post                    # 使用的布局（不需要改）
title:      用sticky和伪元素实现table的“冻结窗格”效果  # 标题 
subtitle:   难点在border的固定上，但以后有望纳入升级版的sticky #副标题
date:       2019-03-08              # 时间
author:     Terry Wang                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - css
    - front-end
---
# 用sticky和伪元素实现table的“冻结窗格”效果

## 先说一些题外话

Hi, 已经三个月没有更新blog了，这期间我做的事有：开了题、写了一篇综述性的小小论文（代码克隆检测相关）、在做第二篇论文的实验（和javascript混淆与恶意代码检测有关哦）、在为校招做准备、面了一次ByteDance。最重要的是，来到了现在的公司实习（导师安排的），经过几番周折，终于做上了非常喜欢的前端开发工作。

实习后，我每天的工作基本分为两类：根据需求开发新功能（包括与产品、后端的沟通）、修复bug（包括与测试的沟通）。
我发现在公司里开发前端应用和自己个人开发有几个明显差异：
1. 多人分工协作与版本控制
> 自己一个人开发的时候很少用版本控制，git基本只被我用来找别人的工具、备份自己的代码和博客而已。但在一个团队中开发时，版本控制（尤其是分支）的重要性就凸显出来了。git可以帮我们轻松地合并多人的工作。
2. 新功能开发和修复bug都不是从零开始的，我需要读懂已有的代码的逻辑，并且尽量保持我的新代码和旧代码在逻辑和设计上的一致。
>自己一个人开发时就如同在荒岛上开垦，如果我想实现一个功能基本是google一下，在stackoverflow上找个能用的就用了，不需要考虑太多。
>而在公司开发，我遇到“这个功能怎么实现”这类问题时，第一个动作是从已有的代码中找类似的功能的实现，然后“依葫芦画瓢”地实现出新功能。

>我觉得，在公司开发时保持风格一致可以使得新人很快熟悉代码，从而降低人员流动的成本。但如果旧的代码风格有些不好的设计，新人不经思考地继承过来的话，可能会为项目带来越来越多”bad smell”（可能带来问题的部分）；另外，新人如果想引入新工具（我其实很想在项目中引入antd，但限于angular版本以及样式，被mentor回绝了），通常也比较困难，使得项目难以享受到新工具带来的便利（我觉得自己现在写的很多代码都可以抽象为一些工具，而这类工具应该已经有不少可用的实现了。

## Motivating Example

前一阵同事读了我的博客说，我可以试着把重点放在“把问题描述清楚”上，所以这次我就试试~

最近需要实现一些表格，由于表格中的数据量比较大，所以会要有水平和垂直的滚动。而滚动时，我们有希望可以像[Excel中的“冻结窗格”](https://support.office.com/zh-cn/article/%E5%86%BB%E7%BB%93%E7%AA%97%E6%A0%BC%E4%BB%A5%E9%94%81%E5%AE%9A%E8%A1%8C%E5%92%8C%E5%88%97-dab2ffc9-020d-4026-8121-67dd25f2508f)功能一样，把想要固定的行和列（通常是首行和首列，因为包含了字段和行的名称）固定住不滚动。

得益于[CSS的position中的新成员“sticky”](https://developer.mozilla.org/en-US/docs/Web/CSS/position)，我们可以很方便地让某个元素固定在滚动时停留在视野范围内，然而，测试组的同学发现了一个bug:

> 当表格是有border的，在滚动时,所有border都会随着滚动条滚动走，包括我们想要固定的单元格的border。而“走掉的border”会在原有位置上留下很多“缝隙”，然后就出现了测试组同学提的搞笑的现象了，底层滚动的单元格会透过这些“缝隙”显现出来。

我开始时觉得这个bug应该不会在一些比较厉害的组件库（比如antd）中出现，于是找来antd的相应组件，给它加了个border，结果，居然也出现了这个bug。我把bug用stackblitz复现了（另外，我发现这个工具是个做demo和复现bug的神器，在线IDE的体验已经接近原生vs code了，国内的cloud studio要有信心~），链接：[https://angular-8mdzb7.stackblitz.io](https://angular-8mdzb7.stackblitz.io)。并且[给antd的angular版（ng-zorro）提了issue](https://github.com/NG-ZORRO/ng-zorro-antd/issues/3032)，目前已经被ng-zorro-bot（这个是机器人么）指派啦，期待着他们提出解决的方案。


小插曲，一开始我对开源组件库（尤其又是这么庞大复杂的antd）只抱着“拿来就用”的态度，觉得读读文档，把它当做黑盒来用就ok了。在mentor说“你看下它是怎么实现的”时，我觉得这已经超出我的能力范围，这么复杂我要怎么看（事实证明也确实没全看懂）。但其实用chrome的开发者工具，看看特定元素加了哪些样式就ok了（所以样式这个东西也没办法加密的吧），可以再追溯下源码。所以，有些问题是没有自己想象得那么复杂的，可以敢于尝试。

## 我的解决方案

我在stackoverflow上搜到了一些相关问题，比如[https://stackoverflow.com/questions/50361698/border-style-do-not-work-with-sticky-position-element](https://stackoverflow.com/questions/50361698/border-style-do-not-work-with-sticky-position-element)。border随着元素滚动现在是sticky默认的，无法改变。所以解决方案基本都是用其他元素补充上border滚动走后留下的“缝隙”，从而在视觉上表现出border“固定了”的效果。替代的元素通常有两种，第一种是用box-shadow，第二种是用伪元素实现视觉上和border一样的小竖线和小横线。我最开始用box-shadow实现过一版，mentor用伪元素实现过一版，但已经过去一段时间了（一个多月吧）。我最近又遇到这个bug，发现我的box-shadow实现没有被采用（。。。我得追溯下为什么），而是用伪元素实现的。为了保持代码风格的一致，所以我也用伪元素来实现了。

我用stackblitz做了一个[demo](https://angular-67hwum.stackblitz.io)，CSS部分是用SASS写的，所以把border样式、行列数都写成了变量。如果看不出效果，大家可以根据自己的显示器调整这些变量的值。（用gitpages搭的博客要是能嵌入iframe就好了，这样就能在这个页面内调试了，看来我的blog配置还有升级的需要）。

实现方法就是在需要固定的行和列的外侧用伪元素加上小竖线和小横线，然后别忘把左上角固定单元格的z-index调成比较大的值，使得它能浮动在上面。不过我的实现还有一个小bug，就是左上角单元格和横向、纵向滚动都有关，所以两种样式会有冲突。它需要在四个边框上都加上伪元素。但是貌似是后加的水平滚动的样式覆盖了先加的垂直滚动的样式。我应该需要单独为这种两个方向滚动都需要的表格设置左上角的样式了，这个留待以后尝试。

## 结束语

虽然个人写前端是从2017年开始的，但感觉进步比较密集的时间还是来到公司以后。多谢mentor的耐心指导。目前感觉自己还处在入门阶段，很多工具的使用和基本原理还不是很熟悉，除了在项目中见到一个学一个外，我也有系统地读一些书和文档（最近觉得《你所不知道的JavaScript》系列不错，读这本是因为自己在上次的面试中发现对原生js了解得太少了）。

”零散地查找“和”系统地读书“，这两种方式学习相对来讲都是成本比较低的，而成本较高的就是现在这样，对于问题做专门的研究并写成blog。经过上次的面试我发现，面试中凡是我曾经花比较多的功夫去自己研究并且写成blog或自己实现过的内容，回想起来都要比那些只是从书本上或网上看到、最多照着敲了一遍的要印象深刻得多。所以，我觉得这样专门研究并写blog的方法，对于我来说才是真的能把知识嫁接到自己的知识体系上的方法。想起有一次听[蚂蚁金服体验技术部的知乎live](https://www.zhihu.com/lives/952228227582857216)时有工程师说他们比较看重”自己的思考和体系化的沉淀“，之前一直觉得这个描述比较虚，现在如果让我来定义的话，像这篇博客这样的就是啦，当然我这篇讲得还比较浅，我觉得我可以学习下”得到“和geektime专栏中的吴军老师、winter等作者的写作方式，还有一些比较好的团队或个人博客。

## 参考资料

[1] https://developer.mozilla.org/zh-CN/docs/Web/CSS/Pseudo-elements  
[2] https://developer.mozilla.org/en-US/docs/Web/CSS/position  
[3] https://stackoverflow.com/questions/50361698/border-style-do-not-work-with-sticky-position-element  
[4] https://github.com/NG-ZORRO/ng-zorro-antd 
