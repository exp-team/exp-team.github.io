---
title: 深入新版BS4源码 探索flex和工程化sass奥秘
categories: program
tags: program
date: 2017-01-12
author: 侯策
---

你可能已经听说了一个“大新闻”：Bootstrap4 合并了代号为#21389的PR，宣布[放弃支持IE9，并默认使用flexbox弹性盒模型](https://github.com/twbs/bootstrap/pull/21389)。
这标志着前端开发全面步入“现代浏览器”的时代进一步来临；样式处理也再一次面向未来，拥抱更加灵活的弹性盒模型－Flex布局。

这篇文章会带你深入Bootstrap最新版源码，窥探其架构组织奥秘，并解析最具亮点的栅格化系统。
你也会了解到sass的高阶用法和flex最新语法奥秘。

## BS4的新特性
在开启我们的探索之前，有必要先梳理一下BS4添加的新特性：
1）从Less迁移到Sass： 
现在，Bootstrap已加入Sass的大家庭中。得益于Libsass（Sass 解析器），Bootstrap的编译速度比以前更快；
2）改进网格系统：
新增一个网格层适配移动设备，并整顿语义混合。
3）默认弹性盒模型（flexbox）：
这是项划时代的变动，利用flexbox的优势快速布局。
4）废弃了wells、thumbnails和panels，使用cards代替。
5）新的自定义选项：
不再像上个版本一样，将渐变、淡入淡出、阴影等效果分放在单独的样式表中。而是将所有选项都移到一个Sass变量中。
想要给全局或考虑不到的角落定义一个默认效果？很简单，只要更新变量值，然后重新编译就可以了。
6）使用rem和em单位。
7）重构所有JavaScript插件：
Bootstrap 4用ES6重写了所有插件。现在提供UMD支持、泛型拆解方法、选项类型检查等特性。
8）改进工具提示和popovers自动定位：
这部分要感谢Tether（A positioning engine to make overlays, tooltips and dropdowns better）工具的帮助，
如果你还不知道Tether是什么，可以去他家[Github地址](https://github.com/HubSpot/tether)。

## BS4栅格系统揭秘
了解了以上新特性，我们主要研究BS从诞生以来最大的“卖点”－栅格系统。

## 一个栅格实例
我们选取代表性的BS4官网范例，可以[在线参考](http://v4.bootcss.com/examples/dashboard/#), 或者参看以下截图，
在大屏幕下，我们看到：

当屏幕宽度小于576px时候，我们有：

对应代码：

    <div class="col-6 col-sm-3">
        ...
    </div>
    <div class="col-6 col-sm-3">
        ...
    </div>
    <div class="col-6 col-sm-3">
        ...
    </div>
    <div class="col-6 col-sm-3">
        ...
    </div>

.col-6 class样式在源码里面可以简单归纳（不完全）为：

    .col-6 {
        -webkit-box-flex: 0;
        -webkit-flex: 0 0 50%;
        -ms-flex: 0 0 50%;
        flex: 0 0 50%;
        max-width: 50%;
    }

.col-sm-3 class在源码里面可以归纳为：

    .col-sm-3{
        -webkit-box-flex: 0;
        -webkit-flex: 0 0 25%;
            -ms-flex: 0 0 25%;
                flex: 0 0 25%;
        max-width: 25%;
    }

### 两种类的共存和交替作用
我们看到，代码里设置了这两个class进行样式声明，很明显他们的样式属性是有冲突的，那么他们是如何做到“和平共处”交替发挥作用的呢？

1）在屏幕宽度小于576px时候，我们发现.col-sm-3并没有起作用，这时候起作用的是.col-6。
我们在源码里发现.col-sm-*的样式声明全部在@media (min-width: 576px) {...}的媒体查询中，
这就保证了在576px以下屏幕，只有在媒体查询之外的.col-*样式声明发挥了作用。
2）在屏幕宽度大于576px时候，命中.col-sm-3的样式声明，所有他的优先级一定大于.col-6，这时候就保证了一行四栏的样式“占上风”。

### flex讲解
我们从样式代码里看到类似flex: 0 0 25%的声明，为了理解它，我们从flex属性入手：
flex属性是flex-grow, flex-shrink 和 flex-basis的简写（类似backgroud是很多背景属性的简写一样），
它的默认值为0 1 auto。后两个属性可选。语法格式如下：

    .item {
        flex: none | [ <'flex-grow'> <'flex-shrink'>? || <'flex-basis'> ]
    }

1）flex-grow：属性定义项目的放大比例，默认为0。我们看到BS代码里这个值一直为0，即如果存在剩余空间，也不放大。

2）flex-shrink：属性定义了项目的缩小比例，默认为1，即如果空间不足，该项目将缩小。

3）flex-basis：属性定义了在分配多余空间之前，项目占据的主轴空间（main size）。
浏览器根据这个属性，计算主轴是否有多余空间。它可以设为跟width或height属性一样的值（比如350px），则项目将占据固定空间。
当然BS4这里设置为比例值，这也是响应式自然而然实现的基础。


## SASS在BS4工程化中的伟大作用
看到此，不难明白BS4栅格的实现，但是这并不是此文的最终目的。我们可以深入更多，比如，BS4的栅格系统里，一行一共是12栏。他的媒体查询断点又包括：xs: 0, sm: 576px, md: 768px, lg: 992px, xl: 1200px。
参考其源码dist/css目录下样式代码，我们会想组织生成如此大量的CSS样式，不用预处理器简直是反人类的。而BS4却是把sass用到了极致。

### .col-sm-*是如何生成的
我们深入其scss目录下，scss/mixins/_grid.scss文件：

    @if $enable-grid-classes {
        @include make-grid-columns();
    }

在enable-grid-classes变量为true的情况下（默认为true），调用make-grid-columns()

make-grid-columns()这个mixin定义在scss/mixins/_grid-reamework.scss文件中（找的我好心累。。。）：

    @mixin make-grid-columns($columns: $grid-columns, $gutters: $grid-gutter-widths, $breakpoints: $grid-breakpoints) {
        ...
    }

这个mixin接受三个参数：
1）$columns表示栅格数目默认为12
2）$gutters默认为30
3）$breakpoints表示断点设置，这是一个全局变量，为map类型。
这些你可以在scss/mixins/_breakpoints.scss文件中和scss/_variables.scss文件中找到。

认识了这些参数，我们看.col-sm-*具体实现，下面代码我已经进行过大范围精简，只保留col-sm-*相关部分，并且加了注释：

    @each $breakpoint in map-keys($breakpoints) {
        // Returns a blank string if smallest breakpoint, otherwise returns the name with a dash infront.
        $infix: breakpoint-infix($breakpoint, $breakpoints);
        // Media of at least the minimum breakpoint width. No query for the smallest breakpoint.
        // Makes the @content apply to the given breakpoint and wider.
        @include media-breakpoint-up($breakpoint, $breakpoints) {
            @for $i from 1 through $columns {
                .col#{$infix}-#{$i} {
                    @include make-col($i, $columns);
                }
            }
        }
    }

我们一步一步来分析：
1）@each $breakpoint in map-keys($breakpoints)，对每一个断点进行遍历；
2）breakpoint-infix是一个函数，它定义在css/mixins/_breakpoints.scss文件当中， 返回一个带破折号的断点标识字符串，比如这里我们关系的就是“-sm”；
3）media-breakpoint-up是一个mixin：

    @mixin media-breakpoint-up($name, $breakpoints: $grid-breakpoints) {
        $min: breakpoint-min($name, $breakpoints);
        @if $min {
            @media (min-width: $min) {
            @content;
        }
        } @else {
            @content;
        }
    }

4）breakpoint-min又是一个函数，它返回了断点的具体数值。这里是用来拼媒体查询条件的。
5）最后最关键样式的生成又使用了另外一个定义在css/mixins/_grid.scss文件当中的mixin:

    @mixin make-col($size, $columns: $grid-columns) {
        flex: 0 0 percentage($size / $columns);
        max-width: percentage($size / $columns);
    }

到此为止，我们深入了Bootstrap V4的scss/目录下的源码，研究涉及了：
css/mixins/_grid-framework.scss文件;
css/mixins/_breakpoints.scss文件;
css/mixins/_grid.scss文件;
css/_variables.scss文件;
css/bootstrap-gris.scss文件;
......

如果你理解了这些，那么再去读bootstrap新版源码就不会存在任何难度。相信你也能够在全局上，以“上帝视角”了解sass所起的作用，大型样式框架的架构组织。

## 总结
通过阅读源码的栅格系统部分，我们应该认识到sass对于这样大型样式框架系统的重要意义：
1）css模块化在管理组织上的灵活性；
2）复用的意义，我们使用了大量的mixin,function,全局变量；
3）像JS一样神奇的语法，包括条件、遍历等等等等。
我们也应该对flex这一神器有了更加深刻的认识。

