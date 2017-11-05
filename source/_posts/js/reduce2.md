---
title: 面试题目别有洞天 －> reduce你对Array.reduce()的恐惧
categories: js
tags: js
date: 2017-03-09
author: 侯策
---

之前的一篇文章[面试题目别有洞天 -> 从es6优雅解法，到降级polyfill，再到redux reducer迷之命名](http://www.jianshu.com/p/5b4c2f4c7a52)中，提到了Array.reduce的巧妙用法。对于很多初学者，在理解上可能会有一些困扰。

最近恰好又在Medium上看到了Dave Lunny的[文章](https://hackernoon.com/reduce-your-fears-about-array-reduce-629b334ab945)，也讲述了Array.reduce的一些使用场景。

这篇文章对原文进行意译，并扩充更多的内容。希望对读者有所启发。

## 无处不在的场景和需求

作为JavaScript开发者，你可能了解很多Array内部方法，比如map, filter, and forEach等。但是reduce也许并不是特别熟悉，而这个方法，极具魔力，也是我非常欣赏的。

想想现在的web UI吧，array无处不在：Twitter或者Facebook的feeds流，社交聊天信息等等都是一个数组的抽象。

举个例子，我们有一个用户列表（array of users），它看上去是这样子的：

    
    const users = [
        {
            firstName: 'Bob',
            lastName: 'Doe',
            age: 37,
        }, {
            firstName: 'Rita',
            lastName: 'Smith',
            age: 21,
        }, {
            firstName: 'Rick',
            lastName: 'Fish',
            age: 28,
        }, {
            firstName: 'Betty',
            lastName: 'Bird',
            age: 44,
        }, {
            firstName: 'Joe',
            lastName: 'Grover',
            age: 22,
        }
        //  我们想象这个数组一直延伸下去...
        //  我也编不出来更多的名字了...
    ];

直到有一天，我们需要从这个数组中衍生得到一个新数组。**新数组每一项由20-30岁的用户组成，并且这些用户的全名（first name ＋ 空格 ＋ last name）需要小于10个字符。**

不要问我为什么会有这么奇怪的需求，请去问和你合作的产品经理吧！

## 传统方法
传统的方法下，我们可以这样解决问题：

    const twentySomethingsLongFullNames = users
        //  先选出来年龄符合的用户
        .filter(user => user.age >= 20 && user.age < 30)

        //  再拼接他们的名字:
        .map(user => `${user.firstName} ${user.lastName}`)

        //  最后筛选出名字长度符合要求的用户
        .filter(fullName => fullName.length >= 10);

为了达到更完美的分割，利于测试和可读，我建议把每个验证函数单独再抽象出来：

    const isInTwenties = user => user.age >= 20 && user.age < 30;
    const makeFullName = user => `${user.firstName} ${user.lastName}`;
    const isAtLeastTenChars = fullName => fullName.length >= 10;

    const twentySomethingsLongFullNames = users
                                            .filter(isInTwenties)
                                            .map(makeFullName)
                                            .filter(isAtLeastTenChars);

这种解法已经很优雅了。甚至每个函数单元都可以单独拿出来进行测试。

但是，这肯定不是唯一的方法。如果原始的user数组非常大，那么性能上也不一定能保证最优。好了，是时候印出来我们的reduce了。


## 神秘的reduce

在传统的方法中，我们遍历了三次不同的数组。在大多数情况下，这么做没什么问题。可是数据量较大时，就需要考虑性能影响了。

在使用reduce时：

    const isInTwenties = user => user.age >= 20 && user.age < 30;
    const makeFullName = user => `${user.firstName} ${user.lastName}`;
    const isAtLeastTenChars = fullName => fullName.length >= 10;

    const twentySomethingsLongFullNames = users.reduce(
        (accumulator, user) => {
            const fullName = makeFullName(user);
            if (isInTwenties(user) && isAtLeastTenChars(fullName)) {
              accumulator.push(fullName);
            }
            return accumulator
        },
        []
    );

关于reduce的具体用法可以参考我的上一篇[文章。](http://www.jianshu.com/p/5b4c2f4c7a52) 

这么做对于性能上的提升还是需要数据来说话的。我尝试产生100000条母的数组，结合console.timeing得出了**使用reduce方法，速度上3明显优于传统方法。**benchmark的脚本留在了[Github仓库中](https://github.com/HOUCe/reduce-perf-test/tree/master/reduce-perf-test)，欢迎大家来参考、指正。

你只需要clone下来后执行：cd reduce-perf-test && node test.js

当然，node环境是必须的。

我并不想引起这两种方法“谁更优雅、可读”的讨论。每个开发者都有自己的喜好，选取自己最顺手的就好。


## 另一个example

我们再来看一个例子，顺便通过这个例子可以引出reduce方法实现Array.find。
 
在一堆水果数组里，我想找出来我最不喜欢吃的banana:

    const fruits = [
        { name: 'apples', quantity: 2 },
        { name: 'bananas', quantity: 0 },
        { name: 'cherries', quantity: 5 }
    ];

 在ES6新的Array方法中，我们可以方便地使用find：

    fruits.find(function(i){return i.name === 'bananas'});

如果使用reduce方法：

    const thisShitIsBananas = fruits.reduce((accumulator, fruit) => {
        if (fruit.name === 'bananas') return fruit;
        return accumulator
    });

好了，理解了这些我们来使用reduce方法实现一个类似Array.find的arrayFind方法，当然在使用上有些差别，我们希望：

    const fruitFinder = arrayFind(fruits);
    const thisShitIsBananas = fruitFinder(fruit => fruit.name === 'bananas');

最终的实现为：

    //  arrayFind接受一个数组作为参数并返回一个函数
    //  返回的函数将接受一个筛选函数，最终返回目标结果
    const arrayFind = arr => fn => arr.reduce((acc, item, index) => {
        if (fn(item, index)) return item;
        return acc;
    });

如果想完全monkey patch一个Array.find，我们可以：

    Array.prototype.find = Array.prototype.find || function (fn) {
        let target;
        this.reverse().reduce((acc, item, index) => {
            if (fn(item, index)) return (target = item && item);
            return acc;
        })
        return target;
    }

需要注意的是，因为find()方法是永远返回第一个匹配的元素。所以在进行reduce()处理之前，我先讲数组倒置。当然也可以使用reduceRight()来达到类似的效果，以保证返回的正确顺序。


## 最后一个example

好了，现在我想将user中每个人的全名输出，并且以换行区分。现在来对比三种做法：

方法1: Bad

    let everyonesName = '';
    users.forEach(user => {
        everyonesName += `${user.firstName} ${user.lastName}\n`;
    });

这种方法的不好之处在于多加了一个变量everyonesName，并且这个变量存在mutation,每次都会被复写。

方法2: Better

    const everyonesName = users.map(
        user => `${user.firstName} ${user.lastName}\n`
    ).join('');

这个方法稍好一些，但是我们还是多了一个变量everyonesName，并且这变量是个数组类型，还需要调用join('')方法是永远返回第一个匹配的元素。所以在进行reduce

方法3: Good

    const everyonesName = users.reduce(
        (acc, user) => `${acc}${user.firstName} ${user.lastName}\n`,
        ''
    );

     
## 总结

其实关于reduce的话题无穷无尽，好的应用场景也是数不胜数。最近我也在看Redux源码，其中关于compose工具函数便妙用了reduce来进行连接，令人叹为观止。

如果有其他任何想法，请和我讨论！

Happy Coding!

PS: 作者[Github仓库](https://github.com/HOUCe)，欢迎通过代码各种形式交流。