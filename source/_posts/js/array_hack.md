---
title: 三个叹为观止的ES6 Array hack
categories: js
tags: js
date: 2017-04-04
author: 侯策
---

在JavaScript中，数组随处可见。在最新版本的ECMAScript 6背景下，借助新的展开符、解构等特性，我们可以对数组做很多“四两拨千斤”的事情。
这片文章我会分享几个超级有用的hack技巧。


## 遍历空数组
JavaScript数组其实是天生“稀疏”的。稀疏数组其实是一个很重要的概念：

>A sparse array is one in which the elements do not have contiguous indexes starting at 0.

从定义来看，稀疏数组就是没有从0开始的连续的index。

那么什么样会有稀疏数组？
有没有被赋值的项；
有被删除（delete）的项

我们从下面这个例子来看：

    const arr = new Array(4);

我们新建了一个长度为4的数组。你会发现，遍历这个数组我们只会得到：

    arr.map((elem, index) => index);
    // [undefined, undefined, undefined, undefined]

为了解决这个问题，比如，我想得到一个每一项值为其index的数组，长度为4，可以这样做：

    const arr = Array.apply(null, new Array(4));
    arr.map((elem, index) => index);
    // [0, 1, 2, 3]

当然，我们有一个更好的方法，就是使用ES6的展开符特性：

    const arr = [...new Array(4)];
    arr.map((elem, index) => index);
    // [0, 1, 2, 3]

## 给方法传递一个空参数
如果你想调用某个方法，但是忽略这个方法的某个参数，那么正常情况下，这样做是会报错的：

    function method (a1, a2, a3) { console.log('ok'); }
    method('parameter1', , 'parameter3');
    Uncaught SyntaxError: Unexpected token ,

在实际开放中，这样的场景其实屡见不鲜，我们通常的做法是，将这个函数参数传递为null或者undefined:

    method('parameter1', null, 'parameter3') // or
    method('parameter1', undefined, 'parameter3');

我个人其实并不喜欢用null来代替，因为在JavaScript中，null会被当作一个object来处理，这其实是很奇怪的。但是在ES6中，借助展开符和数组特性，我们能更好地实现上述做法。

上文提到JavaScript中数组其实是天生稀疏的，所以，给一个数组传递一个空值是没有问题的。因此，我们这样做：

    method(...['parameter1', , 'parameter3']);
    // ok


3. Unique array values
I always wonder why the Array constructor doesn’t have a designated method to facilitate the use of unique array values. Spread operators are here for the rescue. Use spread operators with the Set constructor to generate unique array values.
> const arr = [...new Set([1, 2, 3, 3])]
[1, 2, 3]

## 数组去重
数组去重，是一个老生常谈的话题。实现方式真的已经很多了。但是我其实一直以来不明白Array构造函数的原型上，为什么没有一个“官方”的方法，来产生一个不重复的数组或者完成数组去重的功能。ES6展开符的出现，成为了一种“官方”解决方案。

我们使用展开符，结合Set构造函数，便可以产生一个不含重复项的数组：

    const arr = [...new Set([1, 2, 3, 3])]
    // [1, 2, 3]
    
其实， NaN != NaN 对数组去重的不同方法会产生不同影响。在上述方法当中，我们会得到：

    const arr = [...new Set([1, 2, 3, 3, NaN, NaN])]
    // [1, 2, 3, NaN]   


## 总结
今天介绍了几个利用ES6新特性对数组实现的一些hack方法，简单有效且优雅得体。在ES6引领前端开发的今天，希望对大家能有所启发。也欢迎留言，与我讨论。

Happy Coding!

PS: 作者[Github仓库](https://github.com/HOUCe)，欢迎通过代码各种形式交流。
