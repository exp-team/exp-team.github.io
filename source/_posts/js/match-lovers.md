---
title: 从一个拖拽类小游戏开发 谈一谈ES6中面向对象思路设计组件 
categories: js
tags: js
date: 2017-04-02
author: 侯策
---

无意中在Github看到了一个拖拽类小游戏，翻看了一下代码，发现拖拽的实现使用了jQuery UI当中现成的组件。
同样在不经意间，看到了@波同学一系列的[进阶教程](http://www.jianshu.com/u/10ae59f49b13)，每一章节内容很丰富，质量也很高，尤其有一节讲到了“面向对象实战之封装拖拽对象”，并且最后实现了封装成jquery插件。

这里，我将这两个“无意中”进行了整合。使用ES6中的面向对象，实现了“拖拽”类，并且使用并开发成了一个新的小游戏－lovers match.

这篇文章谈一谈设计“拖拽”类的实现思路和整个游戏开发的设计。

## 游戏一瞥

## ES6中的class和原型链系列知识
JavaScript语言的传统方法是通过构造函数，定义并生成新对象。
ES6中提供了更接近传统语言的写法，引入了Class（类）这个概念。通过class关键字，可以定义类。
当然，ES6的class其实只是一个语法糖，它的绝大部分功能，ES5都可以做到。我曾经也写过[分析文章](http://www.jianshu.com/p/d36fb31f9cff)，来讨论Babel是如何对ES6中的Class进行编译并实现继承的。

如果你对传统模式很感兴趣，或者想修炼基础知识，[不妨一读](http://www.jianshu.com/p/d36fb31f9cff)。
同样，也很建议对比一下@波同学的传统代码实现，和我的ES6版本实现。相信对于原型、原型链、构造函数的知识会有一个更加清晰的认识。

## ES6封装的拖拽类
先来看一下实现的Drag类实现结构：

    class Drag {
        constructor(opt) {
            ...
        }
        init() {
            ...
        }
        getPosition() {
            ...
        }
        setPostion(pos) {
            ...
        }
        getTransform() {
            ...
        }
    }

上面代码定义了一个Drag类，可以看到里面有一个constructor方法，这就是构造方法，其实相当于传统的构造函数。我们在这里进行一些初始化操作。

另外，需要注意的是，除了init方法，我们还可以看到getPosition，setPostion，getTransform这三个方法。其实这三个只是工具函数，不需要对外暴露。所以写在上面那样的位置，并不是很合理的。在每一个Drag的实例中，并不需要这样的方法。

更好的做法当然是加上类似Java中的private关键字，使之成为私有方法。那么在ES6中，是否有这样一个关键字呢？
在阮一峰大大的《ECMAScript6 标准》一书中，这么写到：
>目前，有一个提案，为class加了私有属性。方法是在属性名之前，使用#表示。很有意思的一点是，之所以要引入一个新的前缀#表示私有属性，而没有采用private关键字，是因为JavaScript是一门动态语言，使用独立的符号似乎是唯一的可靠方法，能够准确地区分一种属性是私有属性。另外，Ruby语言使用@表示私有属性，ES6没有用这个符号而使用#，是因为@已经被留给了 Decorator。

另外，可以接受的方法是使用static关键字：加上static关键字，就表示该方法不会被实例继承，而是直接通过类来调用，这就称为“静态方法”。

其实，这些关键字要么就是在草案阶段，要么就是在提议阶段，不可能直接使用。为此，我们应该把这三个工具方法放在class声明之外，作用域之内。


## Transform属性的嗅探
如果你理解拖拽这种交互细节，你就会想到拖拽过程中需要改变目标元素的位置信息。位置信息的改变，大体上使用两种方式：
1）left/top/right/bottom属性
当然，这要配合relative或者absolute属性使用。
2）CSS3 transform:translate()属性

我们推荐第二种方法，毕竟如果我们使用translate3d()赋值，会开启GPU加速，在拖拽性能上会更有优势。
但是问题在于浏览器兼容性上。有的浏览器不支持，有的浏览器需要加浏览器前缀。
因此更合理的做法是做一个嗅探，在不支持的情况下，采用left/top/right/bottom属性的Plan B:

    getTransform() {
        let transform = '';
        let divStyle = document.createElement('div').style;
        let transformArr = ['transform', 'webkitTransform', 'MozTransform', 'msTransform', 'OTransform'];
        var i = 0;
        let len = transformArr.length;
        for(; i < len; i++)  {
            if(transformArr[i] in divStyle) {
                return transform = transformArr[i];
            }
        }
        return transform;
    }

上面这个工具方法的目的是对transform进行统一范化。主要是路是创建一个div标签，之后使用for...in遍历div的style属性。
如果浏览器不兼容，transform值为空，否则为相应支持的加上浏览器前缀的transform属性。

其实上面程序实现不太完美的一点是，我们每次嗅探，都会生成一个无意义的div标签。这样当然会造成浪费。



## 拖拽过程设计
在拖拽过程设计上，应该很容易想到需要监听touchstart/mousedown，touchmove/mousemove，touchend/mouseup这三对事件。
当mousedown触发时，我们执行：

    me.startX = e.pageX;
    me.startY = e.pageY;

    const pos = me.getPosition();
    me.sourceX = pos.x;
    me.sourceY = pos.y;

    me.dragStart && me.dragStart({me.startX, me.startY, ...pos});

    function moveHandler(e) {
        ...
    }
    function endHandler() {
        ...
    }

    $(document).on(eMove, moveHandler);
    $(document).on(eEnd, endHandler);

dragStart是我设计暴漏给用户在mousedown触发时的回调函数，在获取到事件各种信息（事件坐标，事件对象坐标）之后执行。
之后在document上绑定mousemove触发时的处理函数：moveHandler，和mouseup触发时的处理函数：endHandler。

细心的读者可能要问了，为什么要把这些事件绑定在document上，而不是目标元素上呢？其实在亲自动手实现前，我也不是很理解。
后来发现这么做的目的是为了防止在拖拽过程中，因为移动过快，鼠标已经移出目标元素，而造成拖拽效果实效的问题。如果您不理解，相信动手实践下就会发现。


在mousemove处理函数即moveHandler中，当然要实现的就是根据鼠标移动距离，算出目标元素的坐标信息了。唯一要注意的细节是，对于left/top/right/bottom和transform:translate()的分情况处理。另外，如果你是用原生JS实现，那就需要在注意取目标元素样式值的数字转换问题以及浏览器兼容性问题。
计算和处理过程这里我就不贴出来了。


在mouseup处理函数即endHandler中，需要做的就是对document上已经绑定事件的解绑，并执行定制的回调函数：

    $(document).off(eMove);
    $(document).off(eEnd);
    me.dragEnd && me.dragEnd({target:me.element});


## 总结



Happy Coding!


PS: 作者[Github仓库](https://github.com/HOUCe)，欢迎通过代码各种形式交流。