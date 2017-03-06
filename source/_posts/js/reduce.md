---
title: 面试题里“别有洞天” 谈数组，ES6，Redux和Pollyfill
categories: js
tags: js
date: 2016-02-30
author: 侯策
---

之前的[一篇文章：从一道面试题，到“我可能看了假源码”](http://www.jianshu.com/p/6958f99db769)讨论了bind方法的Pollyfill，今天再分享一个有意思的题目。

从解这道题目出发，我们会谈到数组的Reduce方法，ES6特性和Redux数据流框架的核心Reducer的命名等等。就如唐代诗人章碣《对月》诗中所云：“别有洞天三十六，水晶台殿冷层层。”

## 题目背景
完成一个'flatten'的函数，实现“拍平”一个多维数组。示例如下：

```javascript
    var testArr1 = [[0, 1], [2, 3], [4, 5]];
    var testArr2 = [0, [1, [2, [3, [4, [5]]]]]];
    flatten(testArr1) // [0, 1, 2, 3, 4, 5]
    flatten(testArr2) // [0, 1, 2, 3, 4, 5]

## 解法
先看一眼最终解法：

```javascript
    const flatten = arr => arr.reduce((pre, val) => acc.concat(Array.isArray(pre) ? flatten(val) : val), []);

如果你一眼能看明白，也建议继续往下读。因为会有“不一样”的东西和知识点。
如果你看不明白，更不要畏惧。我先用ES5的思路“翻译”一下，你很快就能看懂。

因为目标数组可能是多维，所以想到第一个念头是递归，递归自然就想到递归的“尽头”，那就要判断变量是否是数组类型。
好吧，我们开始动手实现一个方案：

```javascript
    var flatten = function(array) {
        return array.reduce(function(previous, val) {
            if (Object.prototype.toString.call(val) !== '[object Array]') {
                return (previous.push(val), previous);
            }
            return (Array.prototype.push.apply(previous, flatten(val)), previous);
        }, []);
    };

可能这样写，对于很多人来说，并不能完全理解。千万不要灰心，更不能放弃。我会帮您深入分析。

### return的到底是什么？
我们注意到上面的写法return使用了（）表达式。括号内容前半句是为了执行。这样写也许稍微晦涩难懂一些。请看下面的代码示例：

```javascript
    function t() {
        var a = 1;
        return (a++, a);
    }
    t(); // 2


### Object.prototype.toString.call是什么？
Object.prototype.toString.call可以认为是功能最强大的类型判断语句。在对数组类型进行判断时，需要格外小心：

```javascript
    var a = [];
    typeof a; // "object"
    a instanceof Array; // true;
    Object.prototype.toString.call(a); // "[object Array]"


### reduce方法到底做了什么？
现在到了最关键的地方，reduce方法是ES5引入，很多人使用它的场景并不多。但是了解他的特性却是必须的。遗憾的是，社区上对于它的内容似乎都不是太重视。这里我简要进行“科普”，因为下面我要围绕它进行延伸：

reduce在英文中译为“减少; 缩小; 使还原; 使变弱”，MDN对方法直述为：“The reduce method applies a function against an accumulator and each value of the array (from left-to-right) to reduce it to a single value.”
我并不打算对他直接翻译，因为这样会变的更加晦涩难懂。

我们看他的使用语法：

```javascript
    array1.reduce(callbackfn[, initialValue])


参数分析：

1）array1必需。

一个数组对象。即调用reduce方法的必须是一个数组类型。

2）callbackfn必需。

一个接受最多四个参数的函数。对于数组中的每个元素，reduce方法都会调用 callbackfn 函数一次。
这个callback的4个参数为：

```javascript
    accumulator // 上一次调用回调返回的值，或者是提供的初始值（initialValue）
    currentValue // 数组中正在处理的元素
    currentIndex // 数据中正在处理的元素索引，如果提供了initialValue ，从0开始；否则从1开始
    array // 调用reduce的数组

3)initialValue可选项。

其值用于第一次调用callback的第一个参数。

比如，我们分析一个常用用法：

```javascript
    [0,1,2,3,4].reduce(function(previous, item, currentIndex, array){
      return previous + item;
    });
    // 10

这里并未提供reduce的第二个参数initialValue，所以从数组第一项开始进行回调函数的执行。并且每次回调函数执行完之后的结果，作为下一次的previous执行回调。

所以，上述代码便是一个累加器的实现。


## ES6写法
现在理解了Reduce函数，再结合ES6特性，使解法更加优雅：

```javascript
    const flatten = arr => arr.reduce((pre, val) => acc.concat(Array.isArray(pre) ? flatten(val) : val), []);

这样写是不是太“函数式”了，但是思路跟之前解法完全一样。我只不过充分使用了箭头函数带来的便利。并且使用了更便捷的isArray对数组类型进行判断。这是开篇提到的解法，也是MDN最新版的实现。


## 如何实现一个reduce的pollyfill
现在明白了reduce的秘密，接下来我们需要充分发挥对JS的理解，来手动实现一个reduce函数。毕竟，reduce是ES5带来的数组新特性，在不使用ES5-shim的情况下，需要手动兼容。

同样，在MDN上也有实现，但是我觉得下面的代码实现更加优雅和清晰：

```javascript
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

我也同样看了下ES5-shim里的pollyfill，跟我的思路基本完全一致。唯一有一点区别的地方在于我用了forEach迭代而ES5-shim使用的是简单for循环。

当然，数组的forEach方法也是ES5新增的。单纯从pollyfill兼容角度来看是应该使用for循环。但我这里是为了用简单明了的思路，实现reduce方法，根本目的还是希望对reduce有一个全面透彻的了解。

好了，话不多说，我们来看具体实现：

我们声明一个base并在最后返回。当initialValue未声明时，base为数组对象的第一个值，否则为initialValue。

之后使用slice方法，把数组切割为去除base之后的子数组。这个子数组可能和原数组是等值的，这是在提供initialValue情况下；未提供initialValue时，这个子数组去除了已经作为base的第一项元素。

之后对子数组使用forEach方法，调用func回调函数，并更新base的值。

如果您还不明白，我认为还是对于reduce方法没有掌握透彻。建议在梳理一遍。


## Redux中的reducer
明白了reduce函数，我们再来看一下Redux中的reducer和这个reduce有什么命名上的关联。

熟悉Redux数据流架构的同学理解reducer做了什么，关于这个纯函数的命名，也有一个[官方解释](https://github.com/reactjs/redux/blob/master/docs/basics/Reducers.md)：“It's called a reducer because it's the type of function you would pass to Array.prototype.reduce(reducer, ?initialValue)”，虽然是一笔带过，但是总结的恰到好处。

我详细说一下：Reducers其实是根据之前的状态（previous state）和现有的action（current action）更新state这个累加器（accumulation）。每次redux reducer被执行时，state和action被传入，这个state根据action进行累加或者是“自身消减”（reduce），进而返回最新的state。这符合一个典型reduce函数的用法：state -> action -> state.


## 总结
这篇文章对于如何优雅地“扁平化”一个多维数组进行了解法分析。并且对于reduce方法进行了深入讨论，我们还实现了reduce的pollyfill。在充分理解的基础上，有简要延伸到redux数据架构里面reducer的命名。希望对读者有所启发，也欢迎同我讨论。