---
title: 通过一个场景实例 了解前端处理大数据的无限可能
categories: js
tags: js
date: 2017-04-04
author: 侯策
---

随着前端的飞速发展，在浏览器端完成复杂的计算，支配并处理大量数据已经屡见不鲜。那么，如何在**最小化内存消耗的前提**下，高效优雅地完成复杂场景的处理，越来越考验开发者功力，也直接决定了程序的性能。

本文展现了一个完全在控制台就能模拟体验的实例，通过一步步优化结构，实现了生产并操控100000个对象的场景。

内容借鉴了Owen Densmore最新文章：[Two Headed ES6 Classes!](https://medium.com/@backspaces/two-headed-es6-classes-fe369c50b24)，喜欢看英文原版的同学可以直接戳链接。中文翻译版并非直译，进行了较大幅度的讲解和增删。

最后，文章所有用到的代码我都已经整理托管到[Github](https://github.com/HOUCe/Two-Headed-Classes)当中，欢迎大家围观讨论。

## 场景和初级感知
具体来说，我们需要一个构造函数或者类似“生产工厂”factory模式，实例化100000个以上实例。

先来感知一下具体实现，打开你的控制台，仔细观察并复制粘贴以下代码执行。
我们创建一个长度为100000的数组，数组的每一项元素都为0:

    a = new Array(1e6).fill(0);

在a的基础上，再生产一个长度为100000的数组，数组的每一项元素都为js object，拥有id属性，并且其属性值为其在元素中的index值；

    b = a.map((val, ix) => ({id: ix}))

在b的基础上，生产一个长度为100000的数组，类似于b，同时我们增加其它属性：

    c = a.map((val, ix) => ({id: ix, shape: 'square', size: 10.5, color: 'green'}))

**语义上，我们可以更直观的理解：c就是包含了100000个绿色的、size为10.5的小方块。**

如果你按照指示做了下来，控制台上会有以下内容：

![控制台截图](http://upload-images.jianshu.io/upload_images/4363003-58e9f38eb0790b9d.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**你也许会想，这么大的数据量，内存占用会是什么样的情况呢？**

好，我来带你看看，点击控制台Profiles，选择Take Shapshot。在Window->Window目录下，选择占据内存最大的进行筛选，你会得到：


![内存占用情况](http://upload-images.jianshu.io/upload_images/4363003-2fd5dd675bf80fc3.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



很明显，

- a数组：8MB
- b数组：40MB
- c数组：64MB

也许在实际场景中，除了100000个绿色的、size为10.5的小方块，我们还需要很多不同的颜色，不同的size的形状。之前，这样“变态”的需求常见于游戏应用中。但是现在，复杂项目中类似场景，也许距离你并不遥远。



## ES6 Classes处理需求
简单“热身”之后，我们了解了实际需求。接下来，我们考察一下 ES6 Classes处理的情况。请重新刷新浏览器tab，复制执行以下代码。
我们使用了ES6 Classes概念，并扩充了每个形状的坐标信息：

    class Shape {
        constructor (id, shape = 'square', size = 10.5, color = 'green') {
            this.x = x; //  坐标x轴
            this.y = y; //  坐标y轴
            Object.assign(this, {id, shape, size, color})
        }
    }

    a = new Array(1e6).fill(0);
    b = a.map((val, ix) => new Shape(ix));

此时，我们再来看一下内存占用情况：

![内存占用截图](http://upload-images.jianshu.io/upload_images/4363003-10615f58c727ea91.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



很明显，此时b数组由100000个形状组成，占据：80MB；也许这并不出乎意料，此时的b数组毕竟又多了两个属性。


## 优化设计：Two-Headed Classes
我们先来分析一下上面的实现，熟悉原型链、原型概念的同学也许会明白，之前的方案产生的实例，顺着原型链上溯，具有三层：

> **第一层**：[id, shape, size, color, x, y]; 这一层的hasOwnproperty为true;
**第二次**：[Shape]; 这一层 instance.__proto__ === Constructor.prototype;
**第三层**：[Object]; 这一层的再顶层，就为null了

这样的情况下，数据层只有一层，即为第一层。
但是，如果有大量的不同颜色，不同size，不同形状的情况下。单一数据层，是难以满足我们需求的。
我们需要，再添加一层数据层，构成Two-Headed Classes！同时，还需要对于者默认的属性，实现共享，以节省内存的占用。

**那应该如何实现呢？**
我们可以使用Object.create方法，这样使得生产得到的实例的__proto__指向b数组的元素，然后在最顶层设计一个id属性。
也许这样说过于晦涩，那就直接参考代码吧，请注意，这是本篇文章最难以理解的地方，请务必仔细揣摩：

    two = Object.create(b[0]); 
    // two.__proto__ === b[0]
    two.id = 1;

这样子的话，对于每一个实例，我们有如下关系：

> 第一层：[id]； 这一层的hasOwnproperty为true;
第二层：[id, shape, size, color, x, y];  这一层 instance.__proto__ === Constructor.prototype;
第三层：[Shape];
第四层：[Object];这一层的再顶层，就为null了

我们将Shape的一个实例作为一个新的object的原型，并复写了id属性，原有的id属性将作为默认id。

当然，上边的代码只是“个案”，我们进行“生产化”：

    proto = new Shape(0);
    function newTwoHeaded (ix) {
        const obj = Object.create(proto);
        obj.id = ix;
        return obj
    }
    c = a.map((val, ix) => newTwoHeaded(ix));

这么做多加入了一个数据层，那么有什么“收获”呢？我们来看一下b和c的内存占用情况吧：


![内存占用截图](http://upload-images.jianshu.io/upload_images/4363003-0402466c934dcc5b.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这表明：我们从80MB的b，优化得到了64MB的c! 
**原因当然就在于虽然多加了一层，但是第二层变成了“共享”。 **

当然，如果你基础知识雄厚的话，可能要问：那第二层诸如：shape, size, color这些属性变成共享的之后，存在互相干扰怎么破解呢？
好问题，我先不解答，先给大家看一下最后的final product：

    
    class ShapeMaker {
        constructor () {
            Object.assign(this, ShapeMaker.defaults())
        }
        static defaults () {
            return {
                id: null,
                x: 0,
                y: 0,
                shape: 'square',
                size: 0.5,
                color: 'red',
                strokeColor: 'yellow',
                hidden: false,
                label: null,
                labelOffset: [0, 0],
                labelFont: '10px sans-serif',
                labelColor: 'black'
            }
        }
        newShape (id, x, y) {
            const obj = Object.create(this);
            return Object.assign(obj, {id, x, y})
        }
        setDefault (name, value) {
            this[name] = value;
        }
        getDefault (name) {
            return this[name]
        }
    }


在实例化的时候，我们便可以这样使用：

    shapeProto = new ShapreMaker();
    d = a.map((val, ix) => shapeProto.newShape(ix, ix/10, -ix/10))

就像上面所说的，初始化实例时，我们初始化了id, x, y这么三个参数。作为该实例的数据层。这个实例的原型上，也有类似的参数，来保证默认值。这些原型上的属性，对于实例数组中的每个实例，都是共享的。

为了更好的对比，如果设计是这样子：

    function fatShape (id, x, y) {
        const a = new shapeMaker();
        return Object.assign(a, {id, x, y})
    }
    e = a.map((val, ix) => fatShape(ix, ix/10, -ix/10))

那么所有属性无法共享，而是各自拷贝了一份。在内存的占用上，将是我们给出方案的三倍之多！

![内存占用截图](http://upload-images.jianshu.io/upload_images/4363003-a2e2a7d060df822d.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## 阿喀琉斯之踵
阿喀琉斯，是凡人珀琉斯和美貌仙女忒提斯的宝贝儿子。忒提斯为了让儿子炼成“金钟罩”，在他刚出生时就将其倒提着浸进冥河，遗憾的是，乖儿被母亲捏住的脚后跟却不慎露在水外，全身留下了惟一一处“死穴”。后来，阿喀琉斯被帕里斯一箭射中了脚踝而死去。
**后人常以“阿喀琉斯之踵”譬喻这样一个道理：即使是再强大的英雄，他也有致命的死穴或软肋。**

就像我们刚才提的到解决方案一样，也有一些“不足”。问题其实在之前我也已经抛出：“第二层诸如：shape, size, color这些属性变成共享的之后，存在互相干扰怎么破解呢？”

这个问题的答案其实也隐藏在上面的代码中，很简单，就是我们在实例的自身属性上，进行复写，而避免更改原型上的属性造成污染。

如果你看的云里雾里，不要紧，马上看一下我下面的代码说明：

    d.every((item) => item.shape === 'square') // true

打印为true，是因为d数组中的每个实例的shape属性，都在原型上，且初始值都为'square';

现在我们调用setDefault方法，实现对默认shape的改写。

    shapeProto.setDefault('shape', 'circle');
    d.every((item) => item.shape === 'square'); // false

因为此时所有实例的shape都在原型上，并共享这个原型。更改之后，我们有：

    d.every((item) => item.shape === 'circle'); // true


但是，我只想把第一个实例的shape设置为triangle，其他的不变，该怎么办呢？只需要在第一个实例上，增加一个shape属性，进行重写：

    d[0].shape = 'triangle';
    d.every((item) => item.shape === 'circle'); // false

好吧，尝试完毕之后，我们在变回来。

    d[0].shape = 'circle';

这时候，自然有：

    d.every((item) => item.shape === 'circle'); // true

同时，再折腾一下：

    d[0].shape = 'triangle';
    d.every((item) => item.shape === 'triangle'); // false

相信下面的也不难理解了：

    shapeProto.setDefault('shape', 'triangle');
    d.every((item) => item.shape === 'triangle'); // true


**这种模式其实比单纯使用ES6 Classes要灵活的多，同时也节省了内存。所有的静态属性都是共享的，但是共享的静态属性又都是可变的，可复写的。**



## 总结
这篇文章，我们在开头部分了解到了在大量数据的情况下，内存的占用是如何一步一步变的沉重。同时，我们提供了一种，在传统的Classes之上增加一个数据层的方法，有效地解决了这个问题。解决方案充分利用了Object.create等手段。

当然，理解这些内容并不简单，需要读者有比较扎实的Javascript基础。在您阅读过程当中，有任何问题，欢迎与我讨论。


Happy Coding!

PS: 作者[Github仓库](https://github.com/HOUCe)，欢迎通过代码各种形式交流。