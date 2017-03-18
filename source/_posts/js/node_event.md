---
title: 探索nodeJS事件机制源码 打造属于自己的事件发布订阅系统
categories: js
tags: js
date: 2017-03-22
author: 侯策
---


毫无疑问，nodeJS改变了整个前端开发生态。本文通过分析nodeJS当中events模块源码，由浅入深实现了属于自己的ES6事件观察者系统。千万不要被nodeJS的外表吓到，不管你是写nodeJS已经轻车熟路的老司机，还是初入前端的小菜鸟，都不妨碍对这篇文章的阅读和理解。

##内有乾坤
nodeJS[官方介绍](https://github.com/nodejs/node)中，第二句话便是："Node.js uses an event-driven, non-blocking I/O model that makes it lightweight and efficient"。由此，“事件驱动（event-driven）”对nodeJS设计理念的重要性可见一斑。比如，我们对于文件的读取，任务队列的执行情况都需要这样一个观察者模式来保障。

##那个最熟悉的陌生人
同时，作为前端开发人员，我们对于所谓的“事件驱动”理念——即事件发布订阅模式（Pub/Sub模式）一定再熟悉不过了。这种模式在js里面有与生俱来的基因。我们可以认为JS本身就是事件驱动型语言：比如，页面上有一个button, 点击一下就会触发上面的click事件。这是因为此时有特定程序正在监听这个事件，随之触发了相关的处理程序。

这个模式最大的一个好处在于能够解耦，实现“高内聚、低耦合”的理念。那么这样的一个“熟悉的”模式应该怎么实现呢？

其实社区上已经有不少前辈的实现方式了，但是都不能算特别完美，或者不能完全符合特定的场景需求。

本文通过解析nodeJS源码中的events模块，提取其精华，一步步打造了一个基于ES6的eventEmitter系统。

读者有任何想法，欢迎与我交流。同时希望各路大神给予斧正。

## 背景简介
为了方便大家理解，我从一个很简单的页面实例说起。

该页面中，存在两处不同的收藏组件。一处在页面顶部，一处在页面详情侧栏。第一次点击一个收藏组件按钮，发送异步请求，进行收藏，同时请求成功的回调函数里将页面中所有“收藏”按钮转换状态为“已收藏”。以达到“当前文章”收藏状态的全局同步。


![页面实例](http://upload-images.jianshu.io/upload_images/4363003-a04e1b0f57f8a63d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


完成这样的设计很简单，我们大可在业务代码中进行混乱的操作处理，比如初学者常见的做法是：点击第一处收藏，回调逻辑修改页面当中所有收藏按钮。
这样做的问题在于耦合混乱，不仅仅是一个收藏组件，试想当代码中全都是这样的“随意”操作，后期维护成本便一发不可收。

我的Github仓库中，也有对于这么一个页面实例的分析，读者若想自己玩一下，可以访问[这里。](https://github.com/HOUCe/eventEmitter)

当然，更优雅的做法就是使用事件订阅发布系统。
如何设计一个事件订阅发布系统？我们先来看看nodeJS是怎么做的吧？

 
## nodeJS方案
读者可以自己去nodeJS仓库查找源码，不过更推荐参考我的[Github-事件发布订阅研究项目](https://github.com/HOUCe/eventEmitter)，里面不仅有自己实现的多套基于ES6的事件发布订阅系统，也附赠了nodeJS实现源码。同时我对源码加上了汉语注释，方便大家理解。

在nodeJS中，引入eventEmitter的方式和使用方法如下：

    // 引入 events 模块
    var events = require('events');
    // 创建 eventEmitter 对象
    var eventEmitter = new events.EventEmitter();

我们要研究的，当然就是这个eventEmitter实例。先不急于深入源码，我们需要在使用层面先有一个清晰的理解和认知。不然盲目阅读源码，便极易成为一只“无头苍蝇”。

一个eventEmitter实例，自身包含有四个属性：
1）_events：
这是一个object，其实相当于一个哈希map。他用来保存一个eventEmitter实例中所有的注册事件和事件所对应的处理函数。以键值对方式存储，key为事件名；value分为两种情况，当当前注册事件只有一个注册的监听函数时，value为这个监听函数；如果此事件有多个注册的监听函数时，value值为一个数组，数组每一项顺序存储了对应此事件的注册函数。
需要说明的是，理解value值的这两种情况，对于后面的源码分析非常重要。我认为nodeJS之所以有这样的设计，是出于性能上的考虑。因为很多情况（单一监听函数情况）并不需要在内存上新建一个额外数组。

2）_eventsCount：整型，表示此eventEmitter实例中注册的事件个数。

3）_maxListeners：整型，表示此eventEmitter实例中，一个事件最多所能承载的监听函数个数。

4）domain：在node v0.8+版本的时候，发布了一个模块domain。这个模块做的是捕捉异步回调中出现的异常。这里与主题无关，不做展开。

同样，eventEmitter实例的构造函数原型上，包含了一些更为重要的属性和方法，包括但不限于：
1）addListener(event, listener)：
为指定事件添加一个注册函数（以下称监听器）到监听器数组的尾部。他存在一个别名：on。
2）once(event, listener)：
为指定事件注册一个单次监听器，即监听器最多只会触发一次，触发后立刻解除该监听器。
3）removeListener(event, listener)：
移除指定事件的某个监听器，监听器必须是该事件已经注册过的监听器。
4）removeAllListeners([event])：
移除所有事件的所有监听器。如果指定事件，则移除指定事件的所有监听器。
5）setMaxListeners(n)：
默认情况下，如果你添加的监听器超过10个就会输出警告信息。setMaxListeners 函数用于提高监听器的默认限制的数量。
6）listeners(event)：返回指定事件的监听器数组。
7）emit(event, [arg1], [arg2], [...])：
按参数的顺序执行每个监听器，如果事件有注册监听器返回true，否则返回false。


## nodeJS设计之美
上一段其实简要介绍了nodeJS中eventEmitter的使用方法。下面，我们要做的就是深入nodeJS events模块源码，了解并学习他的设计之美。

### 如何创建空对象？
我们已经了解到，_events是要来储存监听事件(key)、监听器数组(value)的map。那么，他的初始值一定是一个空对象。直观上，我们可以这样创建一个空对象：

    this._events = {};

但是nodeJS源码中的实现方式却是这样：

    function EventHandlers() {};
    EventHandlers.prototype = Object.create(null);
    this._events = new EventHandlers();

这么做的原因是出于性能上的考虑，经过jsperf比较，在v8 v4.9版本中，后者性能有超出2倍的表现。
对此，作为一个“吹毛求疵”有态度的程序员，我写了一个benchmark，对一个对象进行一千次取值操作，求平均时间进行验证：

    _events = {};
    _events.test='test'
    for (let i = 0; i < 1000; i++) {
        window.performance.mark('test empty object start');
        console.log(_events.test);
        window.performance.mark('test empty object end');
        window.performance.measure('test empty object','test empty object start','test empty object end');
    } 
    let sum1 = 0
    for (let k = 0; k < 1000; k++) {
        sum1 +=window.performance.getEntriesByName('test empty object')[k].duration
    }
    let averge1 = sum1/1000;
    console.log(averge1*1000);

    function EventHandlers() {};
    EventHandlers.prototype = Object.create(null);
    _events = new EventHandlers();
    for (let i = 0; i < 1000; i++) {
        window.performance.mark('test empty object start');
        console.log(_events.test);
        window.performance.mark('test empty object end');
        window.performance.measure('test empty object','test empty object start','test empty object end');
    } 
    let sum1 = 0
    for (let k = 0; k < 1000; k++) {
        sum1 +=window.performance.getEntriesByName('test empty object')[k].duration
    }
    let averge1 = sum1/1000;
    console.log(averge1*1000);


我自己的想法是，使用nodeJS源码中这样创建空对象的方式，在对对象属性的读取上能够节省原型链查找的时间。但是，如果一个属性直接在该对象上，即hasOwnProperty()为true，是否还有节省查找时间，性能优化的空间呢？

另外，不同浏览器引擎的处理可能也存在差别，即使是流行的V8引擎，处理机制也“深不可测”。同时，benchmark中都是对同一属性的读取，一般来讲浏览器引擎对同样的操作行为应该会有一个“cache”机制，据我了解JIT(just-in-time)实时汇编，会将重复执行的"hot code"编译为本地机器码，极大增加效率。所以benchmark实现的purity也有被一定程度的干扰。不过好在测试实例都是在相同环境下执行。

所以源码中，此处性能优化上的2倍数值，我持一定的保留态度。


### addListener实现
[经过整理，适当删减后的源码点击这里查看](https://github.com/HOUCe/eventEmitter/blob/master/src/common/event/node-eventEmitter.js)，保留了我的注释。我们来一步一步解读下源码。

判断添加的监听器是否为函数类型，使用了typeof进行验证：

    if (typeof listener !== 'function') {
        throw new TypeError('"listener" argument must be a function');
    }

接下来，要分为几种情况。
case1: 
判断_events表是否已经存在，如果不存在，则说明是第一次为eventEmitter实例添加事件和监听器，需要新创建_events：

    if (!events) {
        events = target._events = new EventHandlers();
        target._eventsCount = 0;
    } 

还记得EventHandlers是什么吗？忘记了把屏幕往上滚动再看一下吧。

同时，添加指定的事件和此事件对应的监听器：

    existing = events[type] = listener;
    ++target._eventsCount;

注意第一次创建时，为了节省内存，提高性能，events[type]值是一个监听器函数。如果再次为相同的events[type]添加监听器时（下面case2），events[type]对应的值需要变成一个数组来存储。

case2:
又啰嗦一遍：如果_events已存在，在为相关事件添加监听器时，需要判断events[type]是函数类型（只存在一个监听函数）还是已经成为了一个数组类型（已经存在一个以上监听函数）。
并且根据相关参数prepend，分为监听器数组头部插入和尾部插入两种情况，以保证监听器的顺序执行：
    
    if (typeof existing === 'function') {
        existing = events[type] = prepend ? [listener, existing] :
                                          [existing, listener];
    } 
    else {
        if (prepend) {
            existing.unshift(listener);
        } 
        else {
            existing.push(listener);
        }
    }

case3:
在阅读源码时，我还发现了一个很“诡异”的逻辑：

     if (events.newListener) {
        target.emit('newListener', type,
                  listener.listener ? listener.listener : listener);
        events = target._events;
    }
    existing = events[type];

仔细分析，他的目的是因为nodeJS默认：当所有的eventEmitter对象在添加新的监听函数时，都会发出newListener事件。这其实也并不奇怪，我个人认为这么设计还是非常合理的。

cae4:
之前介绍了我们可以设置一个事件对应的最大监听器个数，nodeJS源码中通过这样的代码来实现：

    EventEmitter.prototype.setMaxListeners = function setMaxListeners(n) {
        if (typeof n !== 'number' || n < 0 || isNaN(n)) {
            throw new TypeError('"n" argument must be a positive number');
        }
        this._maxListeners = n;
        return this;
    };

当对这个值进行了设置之后，如果超过此阈值，将会进行报警：

    if (!existing.warned) {
        m = $getMaxListeners(target);
        if (m && m > 0 && existing.length > m) {
            existing.warned = true;
            const w = new Error('Possible EventEmitter memory leak detected. ' +
                                `${existing.length} ${String(type)} listeners ` +
                                'added. Use emitter.setMaxListeners() to ' +
                                'increase limit');
            w.name = 'MaxListenersExceededWarning';
            w.emitter = target;
            w.type = type;
            w.count = existing.length;
            process.emitWarning(w);
        }
    }

### emit发射器实现
有了之前的注册监听器过程，那么我们再来看看监听器是如何被触发的。其实触发过程直观上并不难理解，核心思想就是将监听器数组中的每一项，即监听函数逐个执行就好了。

[经过整理，适当删减后的源码](https://github.com/HOUCe/eventEmitter/blob/master/src/common/event/node-eventEmitter.js)同样可以这里找到。源码中，包含了较多的错误信息处理内容，忽略不表。下面我挑出一些“出神入化”的细节来分析。

首先，有了上面的分析，我们现在可以清晰的意识到某个事件的监听处理可能是一个函数类型，表示该事件只有一个事件处理程序；也可能是个数组，表示该事件有多个事件处理程序，存储在监听器数组中。（我又啰嗦了一遍，因为理解这个太重要了，不然你会看晕的）

同时，emit方法可以接受多个参数。第一个参数为事件类型：type，下面两行代码用于获取某个事件的监听处理类型。用isFn布尔值来表示。

    handler = events[type];
    var isFn = typeof handler === 'function';

isFn为true，表示该事件只有一个监听函数。否则，存在多个，储存在数组中。

源码中对于emit参数个数有判断，并进行了switch分支处理：

    switch (len) {
        case 1:
            emitNone(handler, isFn, this);
            break;
        case 2:
            emitOne(handler, isFn, this, arguments[1]);
            break;
        case 3:
            emitTwo(handler, isFn, this, arguments[1], arguments[2]);
            break;
        case 4:
            emitThree(handler, isFn, this, arguments[1], arguments[2], arguments[3]);
            break;
        // slower
        default:
            args = new Array(len - 1);
            for (i = 1; i < len; i++) {
                args[i - 1] = arguments[i];
            }
            emitMany(handler, isFn, this, args);
    }

我们挑一个相对最复杂的看一下——默认模式调用的emitMany：

    function emitMany(handler, isFn, self, args) {
        if (isFn) {
            handler.apply(self, args);
        }
        else {
            var len = handler.length;
            var listeners = arrayClone(handler, len);
            for (var i = 0; i < len; ++i) {
                listeners[i].apply(self, args);
            }
        }
    }

对于只有一个事件处理程序的情况（isFn为true），直接执行：

    handler.apply(self, args);

否则，便使用for循环，逐个调用：

    listeners[i].apply(self, args);

非常有意思的一个细节在于：

    var listeners = arrayClone(handler, len);

这里需要读者细心体会。
源码读到这里，我不禁要感叹设计的严谨精妙之处。上面代码处理的意义在于：防止在一个事件监听器中监听同一个事件，从而导致死循环的出现。
如果您不理解，且看我这个例子：

    let emitter = new eventEmitter;
    emitter.on('message1', function test () {
        // some codes here
        // ...
        emitter.on('message1', test}
    });
    emit('message1');

讲道理，正常来讲，不经过任何处理，上述代码在事件处理程序内部又添加了对于同一个事件的监听，这必然会带来死循环问题。
因为在emit执行处理程序的时候，我们又向监听器队列添加了一项。这一项执行时，又会“子子孙孙无穷匮也”的向后添加。

源码中对于这个问题的解决方案是：在执行emit方法时，使用arrayClone方法拷贝出另一个一模一样的数组，进而执行它。这样一来，当我们在监听器内监听同一个事件时，的确给原监听器数组添加了新的函数，但并没有影响到当前这个被拷贝出来的副本数组。在循环中，我们执行的也是这个副本函数。


### 单次监听器once实现
once(event, listener)是为指定事件注册一个单次事件处理程序，即监听器最多只会触发一次，触发后立刻解除该监听器。

实现方式主要是在进行监听器绑定时，对于监听函数进行一层包装。该包装方式在原有函数上添加一个flag标识位，并在触发监听函数前就调用removeListener()方法，除掉此监听函数。

代码里，我们可以抽丝剥茧（已进行删减）学习一下：

     EventEmitter.prototype.once = function once(type, listener) {
        this.on(type, _onceWrap(this, type, listener));
        return this;
    };

once方法调用on方法（即addListener方法，on为别名），第二个参数即监听程序进行_onceWrap化包装，包装过程为：

    this.target.removeListener(this.type, this.wrapFn);
    if (!this.fired) {
        this.fired = true;
        this.listener.apply(this.target, arguments);
    }

_onceWrap化的主要思想是将once第二个参数listener的执行：

     this.listener.apply(this.target, arguments);

包上了一次判断，并在执行前进行removeListener删除该监听程序。



### removeListener的惊鸿一瞥
removeListener(type, listener)移除指定事件的某个监听器。其实这个实现思路也比较容易理解，我们已经知道events[type]可能是函数类型，也可能是数组类型。如果是数组类型，只需要进行遍历，找到相关的监听器进行删除就可以了。

不过关键问题就在于对数组项的删除。

平时开发，我们常用splice进行数组中某一项的删除，99％的case都会想到这个方法。可是nodeJS相关源码中，对于删除进行了优化。自己封装了一个spliceOne方法，用于删除数组中指定角标。并且号称这个方法比使用splice要快1.5倍。我们就来看一下他是如何实现的：

    function spliceOne(list, index) {
        for (var i = index, k = i + 1, n = list.length; k < n; i += 1, k += 1) {
            list[i] = list[k];
            list.pop();
        }
    }

传统删除方法：

    list.splice(index, 1);

究竟是否计算更快，我也实现了一个benchmark，产生长度为1000的数组，删除其第52项。反复执行1000次求平均耗时:

    let arr = Array.from(Array(100).keys());
    for (let i = 0; i < 1000; i++) {
        window.performance.mark('test splice start');
        arr.splice(52, 1);
        window.performance.mark('test splice end');
        window.performance.measure('test splice','test splice start','test splice end');
    }
    let sum1 = 0
    for (let k = 0; k < 1000; k++) {
        sum1 +=window.performance.getEntriesByName('test splice')[k].duration
    }
    let averge1 = sum1/1000;
    console.log(averge1*1000); // 1.7749999999869034


    let arr = Array.from(Array(100).keys());
    for (let i = 0; i < 1000; i++) {
        window.performance.mark('test splice start');
        spliceOne(arr, 52);
        window.performance.mark('test splice end');
        window.performance.measure('test splice','test splice start','test splice end');
    }
    let sum1 = 0
    for (let k = 0; k < 1000; k++) {
        sum1 +=window.performance.getEntriesByName('test splice')[k].duration
    }
    let averge1 = sum1/1000;
    console.log(averge1*1000); // 1.5350000000089494

明显使用spliceOne方法更快，时间上缩短了13.5%，不过依然没有达到官方的1.5，需要说明的是我采用最新版本的Chrome进行测试，和官方环境存在偏差。



## 自己造轮子
前文我们感受了nodeJS中的eventEmitter实现方式。我也对于其中的核心方法，在源码层面进行了剖析。学习到了“精华”之后，更重要的要学以致用，自己实现一个基于ES6的事件发布订阅系统。

我的实现版本中充分利用了ES6语法特性，并且相对于nodeJS实现减少了一些“不必要的”优化和判断。

因为nodeJS的实现其实里，很多api在前端浏览器环境开发中并用不到。所以我对对外暴露的方法进行了精简。最终实现上，除去注释部分，只用了不到40行代码。如果您有兴趣，可以去[代码仓库]()访问，整个逻辑还是很简单的。

里面同时附赠了我同事@颜海镜大神基于zepto实现版本，以及nodeJS events模块源码，方便读者进行对比。
整个过程编写时间仓促，其中必然不乏疏漏之处，还请您斧正并与我讨论。


## 总结
对于nodeJS源码events模块的阅读，令我受益匪浅。设计层面上，优秀的包装和抽象对我一定的启发；实现层面上，很多“意想不到”的case处理，让我“叹为观止”。
虽然业务上暂时使用不到nodeJS，但是对于每一个前端开发人员来说，这样的学习我认为是有必要的。今后，我会整理出文章，总结对nodeJS源码更多模块的分析，希望读者能够保持交流和探讨。
整篇文章里面出的benchmark，我认为并不完美。同时，对于浏览器引擎处理上，我存在知识盲点和漏斗，希望有大神给与斧正。


PS：百度知识搜索部大前端继续招兵买马，有意向者火速联系。。。

