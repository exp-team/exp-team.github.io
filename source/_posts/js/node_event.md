---
title: 由nodeJS源码激发的灵感：“打造属于自己的事件发布订阅系统”
categories: js
tags: js
date: 2017-03-17
author: 侯策
---

毫无疑问，nodeJS改变了整个前端开发生态。不管你是写nodeJS已经轻车熟路的老司机，还是初入前端的小菜鸟，都不妨碍对这篇文章的阅读和理解。nodeJS[官方介绍](https://github.com/nodejs/node)中，第二句话就是："Node.js uses an event-driven, non-blocking I/O model that makes it lightweight and efficient."由此“事件驱动（event-driven）”对nodeJS设计理念的重要性可见一斑。没错，我们对于文件的读取，任务队列的执行情况都需要这样一个观察者模式来保障。

同样，作为前端开发人员，我们对于所谓的“事件驱动”——事件发布订阅模式（Pub/Sub模式）一定再熟悉不过了。这种模式在js里面有与生俱来的基因优势。因为js本身就是事件驱动型语言：比如，我们页面上有一个button, 点击一下就会触发上面的click事件。
因为此时有特定程序正在监听这个事件，随之触发相关的处理程序。

这个模式最大的一个好处在于，能够解耦，实现“高内聚、低耦合”的理念。那么这样的一个模式应该怎么实现呢？
本文通过解析nodeJS源码中events模块，提取其精华，一步步打造了一个基于ES6的eventEmitter系统。

读者有任何想法，欢迎与我交流。同时希望各路大神给予斧正。

## 背景简介
为了方便大家理解，我从一个很简单的[页面实例](http://jingyan.baidu.com/article/ab69b270a678272ca6189f6a.html)说起。
该页面中，从在两处不同的收藏组件。一处在页面顶部，一处在页面详情侧栏。第一次点击组件，发送异步请求，进行收藏，同时请求成功的回调函数里应该将页面中所有“收藏”按钮转换状态为“已收藏”。
![场景截图](/bimg/fave.png)

完成这样的设计很简单，我们大可在业务代码中进行混乱的操作处理。当然，更优雅的做法就是使用事件订阅发布系统。
如何设计一个事件订阅发布系统？我们先来看看nodeJS是怎么做的吧？

 
## nodeJS方案
读者可以自己去nodeJS仓库查找源码，也可以参考我的[Github事件发布订阅研究项目]()，里面不仅有自己实现的多套基于ES6的事件发布订阅系统，也附赠了nodeJS实现源码。

在nodeJS中，引入eventEmitter的方法和使用方法如下：

    // 引入 events 模块
    var events = require('events');
    // 创建 eventEmitter 对象
    var eventEmitter = new events.EventEmitter();

我们要研究的，当然就是这个eventEmitter实例。

一个eventEmitter实例，自身包含有四个属性：
1）_events：是一个object，他用来保存一个eventEmitter实例中所有的注册事件和事件所对应的处理函数。以键值对方式存储，key为事件名，value分为两种情况，当此事件只有一个注册函数时，value为这个注册函数；如果此事件有多个注册函数时，value值为一个数组，数组每一项存储了对应此事件的注册函数。
2）_eventsCount：整型，表示此eventEmitter实例中注册的事件个数。
3）_maxListeners：整型，表示此eventEmitter实例中最多所能承载的事件个数。
4）domain：在node v0.8+版本的时候，发布了一个模块domain。这个模块做的是t捕捉异步回调中出现的异常。这里与主题无关，不做展开。

同样，eventEmitter实例的构造函数原型上，包含了一些更为重要的属性和方法，包括但不限于：
1）addListener(event, listener)：为指定事件添加一个注册函数（以下称监听器）到监听器数组的尾部。
2）once(event, listener)：为指定事件注册一个单次监听器，即监听器最多只会触发一次，触发后立刻解除该监听器。
3）removeListener(event, listener)：移除指定事件的某个监听器，监听器必须是该事件已经注册过的监听器。
4）removeAllListeners([event])：移除所有事件的所有监听器，如果指定事件，则移除指定事件的所有监听器。
5）setMaxListeners(n)：默认情况下，如果你添加的监听器超过10个就会输出警告信息。setMaxListeners 函数用于提高监听器的默认限制的数量。
6）listeners(event)：返回指定事件的监听器数组。
7）emit(event, [arg1], [arg2], [...])：按参数的顺序执行每个监听器，如果事件有注册监听返回true，否则返回false。


## nodeJS eventEmitter设计之美
上一段其实简要介绍了nodeJS中eventEmitter的使用方法，读者可以在社区上找到不少资料。下面，我们要做的就是深入nodeJS事件模块源码，了解他的设计之美。

### 如何创建空对象？
我们已经了解到，_events是要来储存监听器的map。那么，他的初始值一定是一个空对象。我们可以：

    this._events = {};

非常简单直观。但是nodeJS源码中的实现方式：

    function EventHandlers() {};
    EventHandlers.prototype = Object.create(null);
    this._events = new EventHandlers();

这么做的原因是出于定能上的考虑，经过jsperf比较，在v8 v4.9版本中，后者性能有超出2倍的表现。


### addListener实现
[经过整理，适当删减后的源码]()，并保留了我的注释。我们来一步一步解读一下。

判断添加的监听器是否为函数类型，使用了typeof进行验证：

    if (typeof listener !== 'function') {
        throw new TypeError('"listener" argument must be a function');
    }

case1: 
判断_events是否已经存在，如果不存在，则说明是第一次为eventEmitter实例添加监听器，需要新创建_events：

    if (!events) {
        events = target._events = new EventHandlers();
        target._eventsCount = 0;
    } 

同时，添加指定的事件和此事件对应的监听器：

    existing = events[type] = listener;
    ++target._eventsCount;

注意第一次创建时，为了节省内存，提高性能，events[type]值是一个监听器函数。如果再次为相同的events[type]添加监听器时（下面case2），events[type]对应的值需要变成一个数组。

case2:
如果_events已存在，在为相关事件添加监听器时，需要判断events[type]是函数类型（第一次添加）还是已经成为了一个数组类型（非第一次添加）。
并且还要根据参数分为监听器数组头部插入和尾部插入两种情况，以保证监听器的顺序执行：
    
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
在阅读源码时，我发现了一个很“诡异”的逻辑：

     if (events.newListener) {
        target.emit('newListener', type,
                  listener.listener ? listener.listener : listener);
        events = target._events;
    }
    existing = events[type];

仔细分析，他的目的在于：nodeJS默认当所有的eventEmitter对象在添加新的监听函数时都会发出newListener事件。这其实也并不奇怪，我个人认为这么设计还是非常合理的。

cae4:
之前介绍了我们可以设置最大监听事件个数，nodeJS源码中通过这样的代码来实现：

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

### emitter发射器实现
有了之前的注册监听器，那么我们再来看看监听器是如何被触发的。其实触发过程直观上并不难理解，核心思想就是将监听器数组中的每一项，即监听函数逐个执行就好了。
[经过整理，适当删减后的源码]()同样可以这里找到。源码中，包含了较多的错误信息处理内容，忽略不表。下面我挑出一些“出神入化”的细节来分析。

首先，有了上面的分析，我们现在可以清晰的意识到某个事件的监听器可能是一个函数类型，表示该事件只有一个事件处理程序；也可能是个数组，表示该事件有多个事件处理程序，存储在监听器数组中。
同时，emit方法可以接受多个参数。第一个参数为事件类型：type

    handler = events[type];
    var isFn = typeof handler === 'function';

源码中对于参数个数有个判断，并进行了switch分支处理：

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

我们就来看一默认模式调用的emitMany：

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

对于只有一个事件处理程序的情况（isFn为true），直接执行：handler.apply(self, args);
否则，便使用for循环，逐个调用：listeners[i].apply(self, args);

非常有意思的一个细节在于：

    var listeners = arrayClone(handler, len);

读到这里，我不禁要感叹设计的严谨精妙之处，他的意义在于防止在一个事件监听器中监听同一个事件，接而导致死循环。如果您不理解，且看我这个例子：

    let emitter = new eventEmitter;
    emitter.on('message1', function test () {
        // some codes here
        // ...
        emitter.on('message1', test}
    });
    emit('message1');

正常来讲，不经过任何处理，上述代码在事件处理程序内部又添加了对于同一事件的监听，这必然会带来死循环问题。源码中对于这个问题的解决方案是：在执行emit方法时，使用arrayClone方法，拷贝出另一个一模一样的数组，来执行它。这样一来，当我们在监听器内监听同一个事件时，的确给原监听器数组添加了新的函数，但并没有影响到当前这个被拷贝出来的副本数组，在循环中，我们执行的也是这个副本函数。


### 单次监听器once实现
once(event, listener)是为指定事件注册一个单次监听器，即监听器最多只会触发一次，触发后立刻解除该监听器。实现方式主要是在进行监听器绑定时，对于监听函数进行一层包装。该包装方式在原有函数上添加一个flag标识位。代码里，我们可以抽丝剥茧（已进行删减）：

     EventEmitter.prototype.once = function once(type, listener) {
        this.on(type, _onceWrap(this, type, listener));
        return this;
    };

once方法调用on方法（即addListener方法，on为别名alias），第二个参数即监听程序进行_onceWrap化。

    this.target.removeListener(this.type, this.wrapFn);
    if (!this.fired) {
        this.fired = true;
        this.listener.apply(this.target, arguments);
    }

_onceWrap化的主要思想是给once第二个参数listener的执行：this.listener.apply(this.target, arguments);包上了一次判断并在执行前进行removeListener删除该监听程序。


## 自己造轮子
此前我们感受了nodeJS中的eventEmitter实现方式，我也对于其中的核心方法，在源码层面进行了剖析。学习到了精华之后，我们要学以致用，自己实现一个基于ES6的事件发布订阅系统。

因为nodeJS的实现其实很多api在前端浏览器环境开发中并用不到。所以我对暴露的方法进行了精简。目前只开放了：on,once,emit,off四个方法。最终实现除去注释部分，只用了不到40行代码。如果您有兴趣，可以去[代码仓库]()访问，整个逻辑还是很简单的。里面同时附赠了我同事@颜海镜大大基于zepto实现版本。整个过程编写时间仓促，其中必然不乏疏漏之处，还请您斧正并与我讨论。


## 总结



PS：百度知识搜索部大前端继续招兵买马，有意向者火速联系。。。





















