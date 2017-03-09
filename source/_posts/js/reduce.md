---
title: 面试题目别有洞天 -> 从es6优雅解法，到降级polyfill，再到redux reducer迷之命名
categories: js
tags: js
date: 2017-03-09
author: 侯策
---

之前的[一篇文章：从一道面试题，到“我可能看了假源码”](http://www.jianshu.com/p/6958f99db769)讨论了bind方法的各种进阶Pollyfill，今天再分享一个有意思的题目。

从解这道题目出发，我会谈到数组的Reduce方法，ES6特性和Redux数据流框架中Reducer的命名等等。一道典型的题目，却如唐代诗人章碣《对月》诗中所云：“别有洞天三十六，水晶台殿冷层层。”

## 题目背景
完成一个'flatten'的函数，实现“拍平”一个多维数组为一维。示例如下：

    var testArr1 = [[0, 1], [2, 3], [4, 5]];
    var testArr2 = [0, [1, [2, [3, [4, [5]]]]]];
    flatten(testArr1) // [0, 1, 2, 3, 4, 5]
    flatten(testArr2) // [0, 1, 2, 3, 4, 5]

## 解法先睹为快
先看一眼比较优雅的ES6解法：

    const flatten = arr => arr.reduce((pre, val) => pre.concat(Array.isArray(val) ? flatten(val) : val), []);

如果你看不明白，不要放弃。我会用ES5的思路“翻译”一下，相信你很快就能看懂。

如果你一眼能看明白，也建议继续往下读。因为会有“不一样”的知识点。

## 深入解读
第一个想到的念头肯定是递归，递归自然就想到递归的“尽头”，那就要判断数组某项元素是否还是数组类型。
好吧，我们开始动手实现一个方案，其实是上面解法的ES5版本：

    var flatten = function(array) {
        return array.reduce(function(previous, val) {
            if (Object.prototype.toString.call(val) !== '[object Array]') {
                return (previous.push(val), previous);
            }
            return (Array.prototype.push.apply(previous, flatten(val)), previous);
        }, []);
    };

可能这样写，对于很多人来说，并不能完全理解。因为我们使用了较多JS高级用法。关键核心还用到了类似“函数式”思想的reduce方法。
千万不要灰心，继续往下看。

### return的到底是什么？
我们注意到上面的写法return使用了（）表达式。括号内容前半句是为了执行。这样写也许稍微晦涩难懂一些。请看下面的代码示例，你就会明白：

    function t() {
        var a = 1;
        return (a++, a);
    }
    t(); // 2


### Object.prototype.toString.call是什么？
Object.prototype.toString.call可以暂且认为是“功能最强大”的类型判断语句。在对数组类型进行判断时，需要格外小心，比如这样几个“陷阱”：

    var a = [];
    typeof a; // "object"
    a instanceof Array; // true;
    Object.prototype.toString.call(a); // "[object Array]"


### reduce方法到底做了什么？
现在到了最关键的地方。reduce方法是ES5引入，很多人使用它的场景并不多。但是了解他的特性却是必须的。遗憾的是，社区上对于它的内容似乎都不是“太重视”。“函数式“思想也让一些初学者望而却步。这里我简要进行“科普”，因为下面我要围绕它进行延伸：

reduce在英文中译为“减少; 缩小; 使还原; 使变弱”，MDN对方法直述为：“The reduce method applies a function against an accumulator and each value of the array (from left-to-right) to reduce it to a single value.”
我并不打算对他直接翻译，因为这样会变的更加晦涩难懂。

我们看他的使用语法：

    array1.reduce(callbackfn[, initialValue])


参数分析：

1）array1：必需。
一个数组对象。即调用reduce方法的必须是一个数组类型。

2）callbackfn：必需。
一个接受最多四个参数的函数。对于数组中的每个元素，reduce方法都会调用 callbackfn 函数一次。
这个callback的4个参数为：

    accumulator // 上一次调用回调返回的值，或者是提供的初始值（initialValue）
    currentValue // 数组中正在处理的元素
    currentIndex // 数据中正在处理的元素索引，如果提供了initialValue ，从0开始；否则从1开始
    array // 调用reduce的数组

3)initialValue可选项。
其值用于第一次调用callback的第一个参数。如果此参数为空，则拿数组第一项来作为第一次调用callback的第一个参数。

比如，我们分析一个常用用法：

    [0,1,2,3,4].reduce(function(previous, item, currentIndex, array){
      return previous + item;
    });
    // 10

这里并未提供reduce的第二个参数initialValue，所以从数组第一项开始进行回调函数的执行。并且每次回调函数执行完之后的结果，作为下一次的previous执行回调。

所以，上述代码便是一个累加器的实现。


## ES6写法
现在理解了Reduce函数，再结合ES6特性，使解法更加优雅：

    const flatten = arr => arr.reduce((pre, val) => pre.concat(Array.isArray(val) ? flatten(val) : val), []);

这样写是不是太“函数式”了，但是思路跟之前解法完全一样。我只不过充分使用了箭头函数带来的便利。并且使用了更便捷的isArray对数组类型进行判断。这是开篇提到的解法，也是MDN最新版的实现。


## 如何实现一个reduce的pollyfill
现在明白了reduce的秘密，接下来我们需要充分发挥对JS的理解，来手动实现一个reduce函数。毕竟，reduce是ES5带来的数组新特性，在不使用ES5-shim的情况下，需要手动兼容。另外，其实reduce方法可以实现的逻辑，大多都能够使用循环来实现。但是了解这样一个优雅的方法，不管是在程序的可读性上，还是在设计理解层面上，还是很有必要的。

同样，在MDN上也有实现，但是我觉得下面的代码实现更加优雅和清晰：

    var reduce = function(arr, func, initialValue) {
        var base = typeof initialValue === 'undefined' ? arr[0] : initialValue;
        var startPoint = typeof initialValue === 'undefined' ? 1 : 0;
        arr.slice(startPoint)
            .forEach(function(val, index) {
                base = func(base, val, index + startPoint, arr);
            });
        return base;
    };

如果读者有不同实现思路，也欢迎与我讨论。

## ES5-shim的pollyfill
我也同样看了下ES5-shim里的pollyfill，跟我的思路基本完全一致。唯一有一点区别的地方在于我用了forEach迭代而ES5-shim使用的是简单for循环。

当然，数组的forEach方法也是ES5新增的。但我这里是为了用简单明了的思路，实现reduce方法，根本目的还是希望对reduce有一个全面透彻的了解。

如果您还不明白，我认为还是对于reduce方法没有掌握透彻。建议再梳理一遍。


## Redux中的reducer
明白了reduce函数，我们再来看一下Redux中的reducer和这个reduce有什么命名上的关联。

熟悉Redux数据流架构的同学理解reducer做了什么，关于这个纯函数的命名，在redux源码github仓库上也有一个[官方解释](https://github.com/reactjs/redux/blob/master/docs/basics/Reducers.md)：“It's called a reducer because it's the type of function you would pass to Array.prototype.reduce(reducer, ?initialValue)”，虽然是一笔带过，但是总结的恰到好处。

我详细说一下：Redux数据流里，reducers其实是根据之前的状态（previous state）和现有的action（current action）更新state（这个state可以理解为上文累加器的结果（accumulation））。每次redux reducer被执行时，state和action被传入，这个state根据action进行累加或者是“自身消减”（reduce，英文原意），进而返回最新的state。这符合一个典型reduce函数的用法：state -> action -> state.


## 总结
这篇文章对于如何优雅地“扁平化”一个多维数组进行了解法分析。并且对于秉承函数式编程思想的reduce方法进行了深入讨论，我们还实现了reduce的pollyfill。在充分理解的基础上，又简要延伸到redux数据架构里面reducer的命名。熟悉Redux的同学一定会有所感触。

最后希望对读者有所启发，也欢迎同我讨论。

PS：百度知识搜索部大前端继续招兵买马，高级工程师、实习生职位均有，有意向者火速联系。。。