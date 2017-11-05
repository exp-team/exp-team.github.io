---
title: 我们能从一场“数据设计的撕逼”中学到哪些精华
categories: js
tags: js
date: 2017-05-14
author: 侯策
---

最近看了一篇Chris Burgin的[文章：Rewriting JavaScript: Converting an Array of Objects to an Object](https://medium.com/dailyjs/rewriting-javascript-converting-an-array-of-objects-to-an-object-ec579cafbfc7)，其中对比讨论了两种数据设计的不同点。并推广从“一个包含对象的数组”迁移到扁平化的“对象”存储结构。

这不，遥相呼应的是，我也在YouTube上看到了一篇类似的视频：[[Redux] The Best Way to Store Data](https://www.youtube.com/watch?v=aJxcVidE0I0)

我们知道React＋Redux技术栈由于其设计思路，使得前端应用并管理复杂数据越来越重要。比如，读者朋友可以参考我之前分析过的Twitter前端数据架构[一文：解析Twitter前端架构 学习复杂场景数据设计。](http://www.jianshu.com/p/7a56ac1de2a8)

但是，上面提的数据设计方式到底是指的什么？当我们在说扁平化的设计时，到底在说什么？这样的设计真的“百利而无一害”吗？

别急，下文我讲细细道来。



## 通过场景来看透设计
假设我们需要维护一个婚恋交友用户列表，包含了所有注册用户。每个用户有唯一id作为辨识，同时含有用户名和年龄信息。
一个很常见的做法是：

    const userArray = [
        {
            id: 123,
            name: 'LucasHC',
            age: 18
        },
        {
            id: 456,
            name: 'maxiao',
            age: 38
        },
        {
            id: 789,
            name: 'chengwen',
            age: 22
        },
        {
            id: 101,
            name: 'yanhaijing',
            age: 28
        },
        {
            id: 102,
            name: 'zhaowenlin',
            age: 8
        }
    ]

这样的储存设计方式非常的简单，也符合逻辑。

但是，如果我们想选出id为101用户，那就必须要遍历这个数组。通常的做法包括但不限于：

    let idToSelect = 101;
    let selectedUser;

    for (let user of userArray) {
        if (user.id === idToSelect) {
            selectedUser = user;
            // ...
            break;
        }
    }

我们使用了ES6当中的for...of方法。
当然除此之外，使用数组的filter方法、find方法等等都是可行的。比如：

    const selectedUser = userArray.find(user => user.id === idToSelect)

但是不可避免的就是需要遍历这个数组。
想象一下如果这个用户数组比较大，参加婚恋交友的用户很多，性能上先不说，这样的更新用户信息方式也是比较复杂的。

值得注意的是，Chris Burgin的原文当中，观点还有

> This is fairly simple and will traverse the array and assign the correct person to selectedPerson. This presents many problems though, mainly it introduces mutation into your code. Scary!

不过，据我理解，上述的操作方式如果非要这么理解所谓的“mutation”的话，Chris Burgin可能想表达user赋值给selectedUser属于引用赋值，确实会存在“共享”的问题。但是他提供的方法（见下文），也同样没有规避这样的问题。

当然，不止我注意了这样的细节。原文下面的评论，也有很多读者留言表示：

>I don’t see a mutation here? 

>I don’t understand how it mutates the array. Does this portion “let person of peopleArray” somehow change the array?

看到这里，如果您有自己对于“mutation”的看法，也欢迎和我交流。


## 更好的方式
更理想的方式是把刚才的userArray数组转换为对象来存储。就像下面所做的这样：

    const userObject = {
        "123": { id: 123, name: "LucasHC", age: 18 },
        "456": { id: 456, name: "maxiao", age: 38 },
        "789": { id: 789, name: "chengwen", age: 22 },
        "101": { id: 101, name: "yanhaijing", age: 28 },
        "102": { id: 102, name: "zhaowenlin", age: 8 }
    }

这种方式下，我们想要选出id为101用户的操作更简单了：

    const idToSelect = "101";
    const selectedUser = userObject[idToSelect];
    // ...

只需要按照唯一id便可达到我们的目的。完全不再需要遍历一个数组！

这种方式更加简洁清晰。按照Chris Burgin的说法，同样还可以“removes mutation”。当然，我对此持保留态度。


## 数据转换
当然，在某些场景下就如上文所讲，数组转换为对象来存储数据是一种极好的方式。可是有时候我们并没有设计数据的主动权。比如从后端返回的数据就是

        const userArray = [
        {
            id: 123,
            name: 'LucasHC',
            age: 18
        },
        {
            id: 456,
            name: 'maxiao',
            age: 38
        },
        {
            id: 789,
            name: 'chengwen',
            age: 22
        },
        {
            id: 101,
            name: 'yanhaijing',
            age: 28
        },
        {
            id: 102,
            name: 'zhaowenlin',
            age: 8
        }
    ]

这种形式。这就需要我们进行手动转换。其中一种JS原生转换方式为：

    const arrayToObject = (array) =>
        array.reduce((obj, item) => { obj[item.id] = item; return obj }, {})

    const idToSelect = "101";
    const userObject = arrayToObject(userArray);
    console.log(userObject[idToSelect]);

关于reduce函数的用法，如果您还不熟悉，建议去补一补课。这个极具函数式风情的API绝对值得学习。

当然我们可以将转换函数做的更加通用一些：

    const arrayToObject = (array, keyField) =>
       array.reduce((obj, item) => { obj[item[keyField]] = item;return obj }, {})

我们接受keyField参数作为提取因子。

当然，我们还可以换一种姿势进行：

    const arrayToObject = (arr, keyField) =>
        Object.assign({}, ...arr.map(item => ({[item[keyField]]: item})))

这些都能完全达到我们的目的。



## 一些思考
实现了我们所有想要的结果，但是我们还应该进行更进一步的思考。
关于两种方式的比较，我们思考这么一个问题：如果场景需求我们在遍历数据时保证顺序性，或者存储的数据有严格的顺序要求。这时候也许使用数组是一种更加可靠的方法。

另外，我们实现了arrayToObject，但是有时候arrayToMap，使用Map结构来存取数据也许更加必要。ES6为我们提供了Map数据结构。它是一个”value-value”的对应。关于使用Object还是Map，这两种数据结构的比较，我给大家推荐[这里，进行了解](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Map#Objects_and_maps_compared)，毕竟某种情况下，Map的取值相比Object还是有优势的。

另外，关于arrayToMap的实现，也很简单：

    const arrayToMap = (array, keyField) => new Map(
        array.map((item) => [ item[keyField], item ])
    )

最后，关于广受争议的"Mutation"之争，我也给出我的想法。Chris Burgin的前后两种实现，我认为是否存在Mutation.关键在于：取出元素之后，接下来会如何操作去更新userObject/userArray.
拿update data举例，如下：

    const userObject = {
        "123": { id: 123, name: "LucasHC", age: 18 },
        "456": { id: 456, name: "maxiao", age: 38 },
        "789": { id: 789, name: "chengwen", age: 22 },
        "101": { id: 101, name: "yanhaijing", age: 28 },
        "102": { id: 102, name: "zhaowenlin", age: 8 }
    }

    const idToSelect = "101";
    const selectedUser = userObject[idToSelect];

    selectedUser.name = "xulinfeng";

    
    const userObject2 = Object.assign({}, userObject, {
        "101": selectedUser
    });

这种更改，我们得到更新后的userObject2同时，也更改了原有的userObject;

但是为了演示，我们如果嵌套两层Object.assign，那么就不会有类似问题。

    const userObject = {
        "123": { id: 123, name: "LucasHC", age: 18 },
        "456": { id: 456, name: "maxiao", age: 38 },
        "789": { id: 789, name: "chengwen", age: 22 },
        "101": { id: 101, name: "yanhaijing", age: 28 },
        "102": { id: 102, name: "zhaowenlin", age: 8 }
    }

    const idToSelect = "101";
    const selectedUser = userObject[idToSelect];
    
    const userObject2 = Object.assign({}, userObject, {
        "101": Object.assign({}, selectedUser, {name: "xulinfeng"})
    });

当然，存在更优雅的方法，比如使用对象的解构去实现。

最后，关于数据扁平化的设计上，大家可以关注[normalizr类库](https://github.com/paularmstrong/normalizr)



## 总结
我想，由Chris Burgin的文章，学习到的知识不少；更重要的是由此引出的思考。希望这篇文章对于大家在前端数据存储的设计上、Javascript语言特性上、甚至函数式编程思想上等都有不同程度的启发。
欢迎与我讨论。


Happy Coding!

PS: 作者[Github仓库](https://github.com/HOUCe)，欢迎通过代码各种形式交流。














