---
title: ES6 Async/Await 完爆Promise的6个原因
categories: js
tags: js
date: 2017-04-14
author: 侯策
---

自从Node的7.6版本，已经默认支持async/await特性了。如果你还没有使用过他，或者对他的用法不太了解，这篇文章会告诉你为什么这个特性“不容错过”。本文辅以大量实例，相信你能很轻松的看懂，并了解Javascript处理异步的一大杀器。

文章灵感和内容借鉴了[6 Reasons Why JavaScript’s Async/Await Blows Promises Away (Tutorial)](https://hackernoon.com/6-reasons-why-javascripts-async-await-blows-promises-away-tutorial-c7ec10518dd9)，英文好的同学可以直接戳原版参考。


## 初识Async/await
对于还不了解Async/await特性的同学，下面一段是一个“速成”培训。
Async/await 是Javascript编写异步程序的新方法。以往的异步方法无外乎回调函数和Promise。但是Async/await建立于Promise之上。对于Javascript处理异步，是个老生常谈却历久弥新的话题：

>从最早的回调函数，到 Promise 对象，再到 Generator 函数，每次都有所改进，但又让人觉得不彻底。它们都有额外的复杂性，都需要理解抽象的底层运行机制。
异步编程的最高境界，就是根本不用关心它是不是异步。
async 函数就是隧道尽头的亮光，很多人认为它是异步操作的终极解决方案。


## Async/await语法

试想一下，我们有一个getJSON方法，该方法发送一个异步请求JSON数据，并返回一个promise对象。这个promise对象的resolve方法传递异步获得的JSON数据。具体例子的使用如下：

    const makeRequest = () =>
        getJSON()
            .then(data => {
                console.log(data)
                return "done"
            })

    makeRequest()

在使用async/await时，写法如下：

    const makeRequest = async () => {
        console.log(await getJSON())
        return "done"
    }

    makeRequest()



对比两种写法，针对第二种，我需要进一步说明：

1）第二种写法（使用async/await），在主体函数之前使用了async关键字。在函数体内，使用了await关键字。当然await关键字只能出现在用async声明的函数体内。该函数会隐式地返回一个Promise对象，函数体内的return值，将会作为这个Promise对象resolve时的参数。
可以使用then方法添加回调函数。当函数执行的时候，一旦遇到await就会先返回，等到异步操作完成，再接着执行函数体内后面的语句。

2）示例中，await getJSON() 说明console.log的调用，会等到getJSON()返回的promise对象resolve之后触发。

我们在看一个例子加强一下理解，该例子取自阮一峰大神的《ECMAScript 6 入门》一书：

    function timeout(ms) {
        return new Promise((resolve) => {
            setTimeout(resolve, ms);
        });
    }

    async function asyncPrint(value, ms) {
        await timeout(ms);
        console.log(value);
    }

    asyncPrint('hello world', 50);

上面代码指定50毫秒以后，输出hello world。


## Async/await究竟好在哪里？
那么，同样是处理异步操作，Async/await究竟好在哪里呢？
我们总结出以下6点。

### 简约而干净Concise and clean
我们看一下上面两处代码的代码量，就可以直观地看出使用Async/await对于代码量的节省是很明显的。对比Promise，我们不需要书写.then，不需要新建一个匿名函数处理响应，也不需要再把数据赋值给一个我们其实并不需要的变量。同样，我们避免了耦合的出现。这些看似很小的优势其实是很直观的，在下面的代码示例中，将会更加放大。

### 错误处理Error handling
Async/await使得处理同步＋异步错误成为了现实。我们同样使用try/catch结构，但是在promises的情况下，try/catch难以处理在JSON.parse过程中的问题，原因是这个错误发生在Promise内部。想要处理这种情况下的错误，我们只能再嵌套一层try/catch，就像这样：

    const makeRequest = () => {
        try {
        getJSON()
            .then(result => {
                // this parse may fail
                const data = JSON.parse(result)
                console.log(data)
            })
            // uncomment this block to handle asynchronous errors
            // .catch((err) => {
            //   console.log(err)
            // })
            } 
        catch (err) {
            console.log(err)
        }
    }

但是，如果用async/await处理，一切变得简单，解析中的错误也能轻而易举的解决：

const makeRequest = async () => {
    try {
        // this parse may fail
        const data = JSON.parse(await getJSON())
        console.log(data)
    } 
    catch (err) {
        console.log(err)
    }
}

### 条件判别Conditionals
想象一下这样的业务需求：我们需要先拉取数据，然后根据得到的数据判断是否输出此数据，或者根据数据内容拉取更多的信息。如下：

    const makeRequest = () => {
        return getJSON()
            .then(data => {
                if (data.needsAnotherRequest) {
                    return makeAnotherRequest(data)
                            .then(moreData => {
                                console.log(moreData)
                                return moreData
                            })
                } 
                else {
                    console.log(data)
                    return data
                }
            })
    }

这样的代码会让我们看的头疼。这这么多层（6层）嵌套过程中，非常容易“丢失自我”。
使用async/await，我们就可以轻而易举的写出可读性更高的代码：

    const makeRequest = async () => {
        const data = await getJSON()
        if (data.needsAnotherRequest) {
            const moreData = await makeAnotherRequest(data);
            console.log(moreData)
            return moreData
        } 
        else {
            console.log(data)
            return data    
        }
    }

### 中间值Intermediate values
一个经常出现的场景是，我们先调起promise1，然后根据返回值，调用promise2，之后再根据这两个Promises得值，调取promise3。使用Promise，我们不难实现：

    const makeRequest = () => {
        return promise1()
            .then(value1 => {
                // do something
                return promise2(value1)
                    .then(value2 => {
                        // do something          
                        return promise3(value1, value2)
                    })
            })
    }

如果你难以忍受这样的代码，我们可以优化我们的Promise，方案是使用Promise.all来避免很深的嵌套。
就像这样：
    
    const makeRequest = () => {
        return promise1()
            .then(value1 => {
                // do something
                return Promise.all([value1, promise2(value1)])
            })
            .then(([value1, value2]) => {
                // do something          
                return promise3(value1, value2)
            })
    }


Promise.all这个方法牺牲了语义性，但是得到了更好的可读性。
但是其实，把value1 & value2一起放到一个数组中，是很“蛋疼”的，某种意义上也是多余的。

同样的场景，使用async/await会非常简单：

    const makeRequest = async () => {
        const value1 = await promise1()
        const value2 = await promise2(value1)
        return promise3(value1, value2)
    }


### 错误堆栈信息Error stacks
想象一下我们链式调用了很多promises，一级接一级。紧接着，这条promises链中某处出错，如下：

    const makeRequest = () => {
        return callAPromise()
            .then(() => callAPromise())
            .then(() => callAPromise())
            .then(() => callAPromise())
            .then(() => callAPromise())
            .then(() => {
                throw new Error("oops");
            })
    }

    makeRequest()
        .catch(err => {
            console.log(err);
            // output
            // Error: oops at callAPromise.then.then.then.then.then (index.js:8:13)
        })

此链条的错误堆栈信息并没用线索指示错误到底出现在哪里。更糟糕的事，他还会误导开发者：错误信息中唯一出现的函数名称其实根本就是无辜的。
我们再看一下async/await的展现：

    const makeRequest = async () => {
        await callAPromise()
        await callAPromise()
        await callAPromise()
        await callAPromise()
        await callAPromise()
        throw new Error("oops");
    }

    makeRequest()
        .catch(err => {
            console.log(err);
            // output
            // Error: oops at makeRequest (index.js:7:9)
        })

也许这样的对比，对于在本地开发阶段区别不是很大。但是想象一下在服务器端，线上代码的错误日志情况下，将会变得非常有意义。你一定会觉得上面这样的错误信息，比“错误出自一个then的then的then。。。”有用的多。


### 调试Debugging
最后一点，但是也是很重要的一点，使用async/await来debug会变得非常简单。
在一个返回表达式的箭头函数中，我们不能设置断点，这就会造成下面的局面：

    const makeRequest = () => {
        return callAPromise()
            .then(()=>callAPromise())
            .then(()=>callAPromise())
            .then(()=>callAPromise())
            .then(()=>callAPromise())
    }

我们无法在每一行设置断点。但是使用async/await时：

    const makeRequest = async () => {
        await callAPromise()
        await callAPromise()
        await callAPromise()
        await callAPromise()
    }


## 总结
Async/await是近些年来JavaScript最具革命性的新特性之一。他让读者意识到使用Promise存在的一些问题，并提供了自身来代替Promise的方案。
当然，对这个新特性也有一定的担心，体现在：
他使得异步代码变的不再明显，我们好不容易已经学会并习惯了使用回调函数或者.then来处理异步。新的特性当然需要时间成本去学习和体会；
退回来说，熟悉C#语言的程

PS: 作者[Github仓库](https://github.com/HOUCe)，欢迎通过代码各种形式交流。
