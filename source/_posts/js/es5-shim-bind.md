---
title: 从一道面试题的进阶，到“我可能看了假源码”（2）
categories: js
tags: js
date: 2017-01-20
author: 侯策
---

上一篇[从一道面试题的进阶，到“我可能看了假源码”]()由浅入深介绍了关于一篇经典面试题的解法。
最后在皆大欢喜的结尾中，突生变化，悬念又起。这一篇，就是为了解开这个悬念。

如果你还没有看过[前传]()，可以参看前情回顾：
1. 问题是模拟实现ES5中原生bind函数；
2. 我们通过4中递进实现达到了完美状态；
3. 可是ES5-shim中的实现，又让我们大跌眼镜...

## ES5-shim的悬念
ES5-shim实现方式源码贴在了最后，我们看看他做了什么奇怪的事情：
1）从结果上看，返回了bound函数。
2）bound函数是这样子声明的：

    bound = Function('binder', 'return function (' + boundArgs.join(',') + '){ return binder.apply(this, arguments); }')(binder);

3）bound使用了系统自己的构造函数Function来声明，第一个参数是binder，函数体内又binder.apply(this, arguments)。
我们知道这种动态方式，类似eval。最好不要使用它，因为用它定义函数比用传统方式要慢得多。
4）那么ES5-shim抽风了吗？

## 追根问底
答案肯定是没抽风，他这样做是有理由的。

### 神秘的函数的length属性
你可能不知道，每个函数都有length属性。对，就像数组和字符串那样。函数的length属性，用于表示函数的形参。更重要的是函数的length属性值是不可重写的。我写了个测试代码：

    function test (){}
    test.length  // 输出0
    test.hasOwnProperty('length')  // 输出true
    Object.getOwnPropertyDescriptor('test', 'length') 
    // 输出：
    // configurable: false, 
    // enumerable: false,
    // value: 4, 
    // writable: false 

### 拨云见日
说到这里，那就好解释了。
ES5-shim为了最大限度的进行兼容，所以：既然不能修改length的属性值，那么在初始化时赋值总可以吧，也就是定义函数的形参个数！
于是我们可通过eval和new Function的方式动态定义函数来。
同时，很有意思的是，源码里有这样的注释：

    // XXX Build a dynamic function with desired amount of arguments is the only
    // way to set the length property of a function.
    // In environments where Content Security Policies enabled (Chrome extensions,
    // for ex.) all use of eval or Function costructor throws an exception.
    // However in all of these environments Function.prototype.bind exists
    // and so this code will never be executed.

他解释了为什么要使用动态函数，就如同我们上边所讲的那样，是为了保证length属性的合理值。但是在一些浏览器中出于安全考虑，使用eval或者Function构造器都会被抛出异常。但是，巧合也就是这些浏览器基本上都实现了bind函数，这些异常又不会被执行。

So, What a coincidence!

### 叹为观止
我们明白了这些，再看他的进一步实现：

    if (!isCallable(target)) {
        throw new TypeError('Function.prototype.bind called on incompatible ' + target);
    }

这是为了保证调用的正确性，他使用了isCallable做判断，isCallable很好实现：

    isCallable = function isCallable(value) { 
        if (typeof value !== 'function') { 
            return false; 
        }
    }

重设绑定函数的length属性：

    var boundLength = max(0, target.length - args.length);

构造函数使用情况，在binder中也有效兼容。如果你不明白，可以参考[上一篇]()

    if (this instanceof bound) { 
        ... // 构造函数调用情况
    } else {
        ... // 正常方式调用
    }

    if (target.prototype) {
        Empty.prototype = target.prototype;
        bound.prototype = new Empty();
        // Clean up dangling references.
        Empty.prototype = null;
    }

## 无穷无尽
当然，ES5-shim里还归纳了几项todo...

    // TODO
    // 18. Set the [[Extensible]] internal property of F to true.
    // 19. Let thrower be the [[ThrowTypeError]] function Object (13.2.3).
    // 20. Call the [[DefineOwnProperty]] internal method of F with
    //   arguments "caller", PropertyDescriptor {[[Get]]: thrower, [[Set]]:
    //   thrower, [[Enumerable]]: false, [[Configurable]]: false}, and
    //   false.
    // 21. Call the [[DefineOwnProperty]] internal method of F with
    //   arguments "arguments", PropertyDescriptor {[[Get]]: thrower,
    //   [[Set]]: thrower, [[Enumerable]]: false, [[Configurable]]: false},
    //   and false.
    // 22. Return F.

比较简单，我就不再翻译了。

## 总结
通过学习ES5-shim的源码实现bind方法，结合前一篇，希望能对bind和JS包括闭包，原型原型链，this等一系列知识点能有更深刻的理解。
同时在程序设计上，尤其是逻辑的严密性上，有所积累。

PS：百度知识搜索部大前端继续招兵买马，有意向者火速联系。。。









