---
title: 手把手教你撸一个跑男动画 顺便抽丝剥茧CSS3动画奥秘
categories: web
tags: web
date: 2017-03-14
author: 侯策
---

作为一名真正的前端开发者，我们不能只关注前端的逻辑部分，毕竟“水银泄地”般的页面实现和“炫酷逼真”的动画效果是我们区别于其他程序员所特有的优势。尽量百分之百的还原视觉稿，连接UE设计灵感和用户视觉享受，正所谓“晋帝时祭北郊，更祝版，工人削之，笔入木三分。”借古书法形容我们的代码，当真是恰当准确又自恋无比。

之前的一些文章大多都是分享JS相关，今天轻松一下，我来谈谈前端页面的动画部分。最要通过剖析一个上线的“跑男”动画实例，来把CSS3中动画相关的知识点一网打尽。如果读者有自己的感想或者不一样的见解，欢迎一起讨论。


## 项目简介
这是一个运营活动页面——春季马拉松大比拼：用户以闯关形式参加进行角色扮演。在满足一定条件下，自己扮演的马拉松选手会绕着跑道前进，向终点发起冲击。

部分页面动画效果如下：

![部分动画场景截图](/bimg/run3.gif)

当然，这个动画并不完美。考虑到时间性价比，我只用了两帧重复循环。但也达到了运营和产品小妹的满意。如果在没有上线压力的情况下，我们完全可以把他打磨的更流畅顺滑。

首先，我们来看一下它的具体实现方式吧。如果您觉得很简单，也可以往下读，相信你也会有不一样的收获。

 
## 动画方案
这一系列的动画设计，出于性能和简单的考虑，我采用了纯CSS3来实现。CSS3实现动画，主要有两种方式：transition属性和animation属性。前者用张鑫旭大大的话来说就是用来“平滑的改变CSS的值”。一般对于需要特定帧处理的动画，这显然是苍白无力的。我就不过多介绍。重点介绍一下animation属性。

### animation
animation属性其实是一个简写属性，就像我们更加熟悉的“background”属性一样。它用于设置六个动画属性：

1）animation-name
2）animation-duration
3）animation-timing-function
4）animation-delay
5）animation-iteration-count
6）animation-direction

最重要的就是animation-name，它规定需要绑定到选择器的keyframe名称，它要和keyframes对应。keyframes，就是我们用来定义几个关键节点帧。

具体我就不再进行科普了。如果初学者不了解，社区上关于这些的资料可是一大把。

### 跑男开跑
回到我们具体的业务场景，我们进行分析。跑男的动画其实可以拆分为两种，一个是交替摆腿，另一个是位置移动。这两个动作要严丝合缝的结合。能把这个想清楚，那就实现了一大半。

那么如何让这两种动画一起施加在“静止的”跑男身上呢？

我采用了增加一个div标签的方式：

    <div class="man-wrapper">
        <div class="man" id="man"></div>
    </div>

'man-wrapper'这个div与'man'这个div大小完全一致。父节点处理位移，子节点负责交替摆腿：

    .man-wrapper {
        display: inline-block;
        width: 46px;
        height: 75px;
        position: absolute;
    }
    .man {
        display: inline-block;
        width: 46px;
        height: 75px;
        background: url(img/sprite.png);
        position: absolute;
        top: 0;
        left: 0;
    }

当需要触发位移，开启跑步状态时，父节点添"start-run"类：

    $('.man-wrapper').addClass('start-run');

子节点添加：

    $('#man').addClass('running');

### 动画实现
关于“start-run”的动画设计，在跑道上直道部分相对简单，我们思路是使用transform->translate3d。但是视觉稿上存在不少于5处不规则弯道，在不改变原图的基础上，我们可以使用transform->rotate3d。具体设计看下图：

![动画部分分解](/bimg/run.gif)


