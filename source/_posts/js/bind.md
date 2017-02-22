---
title: 从一道面试题的进阶，到“我可能看了假源码”
categories: js
tags: js
date: 2017-02-20
author: 侯策
---

今天想谈谈一道前端面试题，我做面试官的时候经常喜欢用它来考察面试者的基础是否扎实，以及逻辑、思维能力和临场表现，题目是：“模拟实现ES5中原生bind函数”。
也许这道题目已经不再新鲜，部分读者也会有思路来解答。社区上关于原生bind的研究也很多，比如用它来实现函数“颗粒化（currying）”，
或者“反颗粒化（uncurrying）”。
但是，我确信有很多细节是您注意不到的，也是社区上关于这个话题普遍缺失的。
这篇文章面向有较牢固JS基础的读者，会从最基本的理解入手，一直到分析ES5-shim实现bind源码，相信不同程度的读者都能有所收获。
也欢迎大家与我讨论。

## bind函数究竟是什么?
在开启我们的探索之前，有必要先明确一下bind到底实现了什么：
1）简单粗暴地来说，bind是用于绑定this指向的。（如果你还不了解JS中this的指向问题，以及执行环境上下文的奥秘，这篇文章暂时就不太适合阅读）。

2）bind使用语法：

    fun.bind(thisArg[, arg1[, arg2[, ...]]])

bind方法会创建一个新函数。当这个新函数被调用时，bind的第一个参数将作为它运行时的this，之后的一序列参数将会在传递的实参前传入作为它的参数。本文不打算科普基础，如果您还不清楚，请[参考MDN内容](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Function/bind)。

3)bind返回的绑定函数也能使用new操作符创建对象：这种行为就像把原函数当成构造器。提供的this值被忽略，同时调用时的参数被提供给模拟函数。

## 初级实现
了解了以上内容，我们来实现一个初级的bind函数Polyfill:
    
    Function.prototype.bind = function (context) {
        var me = this;
        var argsArray = Array.prototype.slice.call(arguments);
        return function () {
            return me.apply(context, argsArray.slice(1))
        }
    }

这是一般“表现良好”的面试者所能给我提供的答案，如果面试者能写到这里，我会给他60分。
我们先简要解读一下：
基本原理是使用apply进行模拟。函数体内的this，就是需要绑定this的实例函数，或者说是原函数。最后我们使用apply来进行参数（context）绑定，并返回。
同时，将第一个参数（context）以外的其他参数，作为提供给原函数的预设参数，这也是基本的“颗粒化（curring）”基础。

## 初级实现的加分项
上面的实现（包括后面的实现），其实是一个典型的[“Monkey patching(猴子补丁)”](https://en.wikipedia.org/wiki/Monkey_patch)，即“给内置对象扩展方法”。所以，如果面试者能进行一下“嗅探”，进行兼容处理，就是锦上添花了，我会给10分的附加分。

    Function.prototype.bind = Function.prototype.bind || function (context) {
        ...
    }

## 颗粒化（curring）实现
上述的实现方式中，我们返回的参数列表里包含：atgsArray.slice(1)，他的问题在于存在预置参数功能丢失的现象。
想象我们返回的绑定函数中，如果想实现预设传参（就像bind所实现的那样），就面临尴尬的局面。真正实现颗粒化的“完美方式”是：

    Function.prototype.bind = Function.prototype.bind || function (context) {
        var me = this;
        var args = Array.prototype.slice.call(arguments, 1);
        return function () {
            var innerArgs = Array.prototype.slice.call(arguments);
            var finalArgs = args.concat(innerArgs);
            return me.apply(context, finalArgs);
        }
    }

如果面试者能够给出这样的答案，我内心独白会是“不错啊，貌似你就是我要找的那个TA～”。但是，我们注意在上边bind方法介绍的第三条提到：bind返回的函数如果作为构造函数，搭配new关键字出现的话，我们的绑定this就需要“被忽略”。

## 构造函数场景下的兼容
有了上边的讲解，不难理解需要兼容构造函数场景的实现：

    Function.prototype.bind = Function.prototype.bind || function (context) {
        var me = this;
        var args = Array.prototype.slice.call(arguments, 1);
        var F = function () {};
        F.prototype = this.prototype;
        var bound = function () {
            var innerArgs = Array.prototype.slice.call(arguments);
            var finalArgs = args.concat(innerArgs);
            return me.apply(this instanceof F ? this : context || this, finalArgs);
        }
        bound.prototype = new F();
        return bound;
    }

如果面试者能够写成这样，我几乎要给满分，会帮忙联系HR谈薪酬了。当然，还可以做的更加严谨。

## 更严谨的做法
我们需要调用bind方法的一定要是一个函数，所以可以在函数体内做一个判断：

    if (typeof this !== "function") {
      throw new TypeError("Function.prototype.bind - what is trying to be bound is not callable");
    }

做到所有这一切，我会很开心的给满分。其实MDN上有个自己实现的polyfill，就是如此实现的。
另外，《JavaScript Web Application》一书中对bind()的实现，也是如此。

故事貌似要画上休止符了——

## 一切还没完，高潮即将上演
如果你认为这样就完了，其实我会告诉你说，高潮才刚要上演。曾经的我也认为上述方法已经比较完美了，直到我看了es5-shim源码（已适当删减）：

    bind: function bind(that) {
        var target = this;
        if (!isCallable(target)) {
            throw new TypeError('Function.prototype.bind called on incompatible ' + target);
        }
        var args = array_slice.call(arguments, 1);
        var bound;
        var binder = function () {
            if (this instanceof bound) {
                var result = target.apply(
                    this,
                    array_concat.call(args, array_slice.call(arguments))
                );
                if ($Object(result) === result) {
                    return result;
                }
                return this;
            } else {
                return target.apply(
                    that,
                    array_concat.call(args, array_slice.call(arguments))
                );
            }
        };
        var boundLength = max(0, target.length - args.length);
        var boundArgs = [];
        for (var i = 0; i < boundLength; i++) {
            array_push.call(boundArgs, '$' + i);
        }
        bound = Function('binder', 'return function (' + boundArgs.join(',') + '){ return binder.apply(this, arguments); }')(binder);

        if (target.prototype) {
            Empty.prototype = target.prototype;
            bound.prototype = new Empty();
            Empty.prototype = null;
        }
        return bound;
    }

看到了这样的实现，心中的困惑太多，不禁觉得我看了“假源码”。但是仔细分析一下，剩下就是一个大写的 。。。服！
这里先留一个悬念，不进行源码分析。读者可以自己先研究一下。如果想看源码分析，点击[这篇文章的后续－源码解读](https://exp-team.github.io/blog/2017/02/20/js/es5-shim-bind/)。


## 总结
通过比对几版的polyfill实现，对于bind应该有了比较深刻的认识。作为这道面试题的考察点，肯定不是让面试者实现低版本浏览器的向下兼容，因为我们有了es5-shim,es5-sham处理兼容性问题，并且无脑兼容我也认为是历史的倒退。
回到这道题考查点上，他有效的考察了很重要的知识点：比如this的指向，JS的闭包，原型原型链功力，设计程序上的兼容考虑等等硬素质。
在前端技术快速发展迭代的今天，在“前端市场是否饱和”“前端求职火爆异常”“前端入门简单，钱多人傻”的浮躁环境下，对基础内功的修炼就显得尤为重要，这也是你在前端路上能走多远、走多久的关键。

PS：百度知识搜索部大前端继续招兵买马，有意向者火速联系。。。




