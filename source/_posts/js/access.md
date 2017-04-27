---
title: 万军丛中取敌将首级－如何优雅安全地在深层数据结构中取值
categories: js
tags: js
date: 2017-04-04
author: 侯策
---

古有赵子龙面对“冲锋之势，有进无退，陷阵之志，有死无生”的局面，能万军丛中取敌将首级。
在我们的Javascript中，往往用对象（Object）来存储一个数据结构。如果这个结构非常复杂，那么想要安全优雅地取出一个值，也并非简单。

这篇文章将会详细阐述在一个嵌套较深的场景中，如何安全的完成读写操作。先后会尝试多种方法，希望对读者有所启发。

本文示例借鉴A.Sharif的最新文章：[Safely Accessing Deeply Nested Values In JavaScript](https://medium.com/javascript-inside/safely-accessing-deeply-nested-values-in-javascript-99bf72a0855a)，喜欢看英文原版的同学可以直接戳链接。

## 场景介绍
在React开发中，我们根据数据来渲染视图。经常会出现类似下面这种情况：

    const props = {
        user: {
            posts: [
                { title: 'Foo', comments: [ 'Good one!', 'Interesting...' ] },
                { title: 'Bar', comments: [ 'Ok' ] },
                { title: 'Baz', comments: []}
            ],
            comments: [...]
        }
    }

这是一个典型的获取用户评论信息并加以展示的场景。其实，这还嵌套的不够深，试想一个回复存在多层：回复的回复，回复的回复的回复。。。

姑且先看我们的示例吧，此时我们想获取第一个post的评论信息。用传统的javascript方法应该这么做：

    props.user &&
    props.user.posts &&
    props.user.posts[0] &&
    props.user.posts[0].comments

也许经验丰富的javascript开发者会明白使用这么多&&的意义。这是为了在对象中相关取值的过程，需要验证每一个key和index的存在性。否则会有报错，这将会是致命性的。并且props这个数据结构必然是动态生成的，存在有时valid有时invalid的情况。在测试过程中，很难复现。

同样的尴尬场景比比皆是，想象一下，如果我们需要获取一名用户最后一个评论博客的题目，就需要：

    props.user &&
    props.user.comments &&
    props.user.comments[0] &&
    props.user.comments[0].blog.title

这些例子夸张吗？其实不然。我们明白了，想要获取一个数据值，需要一层一层遍历属性的存在性。这无疑是繁琐的。



## 解决方案
现在明白了我们面临的困扰，接下来我会用纯JavaScript方法，以及最具有函数式代表的JavaScript库－Ramda，辅以柯粒化（currying）等思想和方案解决问题。

### JavaScript方案
先直接上代码：

    const get = (p, o) => p.reduce((xs, x) => (xs && xs[x]) ? xs[x] : null, o)

    console.log(get(['user', 'posts', 0, 'comments'], props)) // [ 'Good one!', 'Interesting...' ]
    console.log(get(['user', 'post', 0, 'comments'], props)) // null

注意这里我使用了一个ES5中，比较偏向函数式思想的reduce方法。关于这个方法，我想很多人其实还并不理解，建议先去进行学习，或者参考我之前的[一篇文章。](http://www.jianshu.com/p/5b4c2f4c7a52)
同时，我尝试获取：user－>posts[0]－>comments，
并配以一个反例：user－>post[0]－>comments；
当然，在反例中，post数组并不存在。

我们来分析一下代码。

    const get = (p, o) =>
        p.reduce((xs, x) =>
            (xs && xs[x]) ? xs[x] : null, o)

我们实现的get方法中，接收两个参数，第一个p表示获取值的路径（path）；另外一个参数表示目标对象。

同样，为了设计上的更加灵活和抽象。我们可以柯粒化我们的方法：

    const get = p => o =>
        p.reduce((xs, x) =>
            (xs && xs[x]) ? xs[x] : null, o)

这样的话，就可以这个姿势调用：

    const getUserComments = get(['user', 'posts', 0, 'comments'])
    console.log(getUserComments(props))
    // [ 'Good one!', 'Interesting...' ]
    console.log(getUserComments({user:{posts: []}}))
    // null

如果关于get方法中reduce的使用还不清楚，那就再看一个简单的例子：

    ['id'].reduce((xs, x) => (xs && xs[x]) ? xs[x] : null, {id: 10})
    // 返回10


### Ramda方案
如果不自己手动设计上述方法的话，我们可以使用Ramda函数式类库完成：

    const getUserComments = R.path(['user', 'posts', 0, 'comments'])

接下来调用需要这个姿势：
    
    getUserComments(props) // [ 'Good one!', 'Interesting...' ]
    getUserComments({}) // null

如果我们想在指定路径下未找到一个值时，不返回null，而是返回自定义的内容呢？我们可以使用pathOr方法，第一个参数用来设置默认输出。

    const getUserComments = R.pathOr([], ['user', 'posts', 0, 'comments'])
    getUserComments(props) // [ 'Good one!', 'Interesting...' ]
    getUserComments({}) // []


## 总结

这篇文章翻译自A.Sharif的最新文章：[Safely Accessing Deeply Nested Values In JavaScript](https://medium.com/javascript-inside/safely-accessing-deeply-nested-values-in-javascript-99bf72a0855a)，其中后半部分未做翻译。
后半部分其实分析了 Ramda＋[Folktale](http://folktalejs.org/)的实现，以及Ramda＋[Lenses](https://medium.com/@dtipson/functional-lenses-d1aba9e52254)的实现。

Folktale和Lenses是非常函数式Functional Programming的思想，理解起来相对晦涩且比较小众。有兴趣的读者可以点击原文去自行了解。

Happy Coding!

如果你对函数式编程并不感冒，大可只学习第一部分的实现。对于函数式编程有兴趣的同学，希望这篇文章能够抛砖引玉，欢迎与我交流。



PS: 作者[Github仓库](https://github.com/HOUCe)，欢迎通过代码各种形式交流。














