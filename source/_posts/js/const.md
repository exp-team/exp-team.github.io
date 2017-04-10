---
title: 从ES6 Const看Javascript语言的诡异与灵活
categories: js
tags: js
date: 2017-04-04
author: 侯策
---

不管您对Javascript有多大偏见，不管您究竟认为什么是世界上最好的语言。都不可否认的是，Javascript现今的火热与流行。根据Stack Overflow最新 年度开发者调查（2017年3月23日），JS又蝉联最热门的编程语言。

笔者对Javascript研究也就不到两年时间，资历尚浅。其实，阮一峰2015年的一篇文章也有谈到类似的话题：[JavaScript有多灵活？](http://www.ruanyifeng.com/blog/2015/02/flexible-javascript.html)，这篇文章非常不错，但是所举的例子稍微有些“偏”而像“奇技异巧”。

今天我们通过ES6中一个新特性来展开，更加大众而普遍，谈一谈Javascript这门语言的灵活性。

本文示例借鉴Jacopo Daeli的最新文章：[What does const stand for in ES6?](https://medium.com/the-node-js-collection/what-does-const-stand-for-in-es6-f7ab3d9e06fc)，喜欢看英文原版的同学可以直接戳链接。

## C-like VS JavaScript
如果你习惯了C-like语言，也许你会很奇怪下边这两段代码：

    // JavaScript
    const numbers = [1, 2, 3, 4, 6]
    numbers[4] = 5
    console.log(numbers[4]) // print 5 

    // C
    const int numbers[] = {1, 2, 3, 4, 6};
    numbers[4] = 5; // error: read-only variable is not assignable
    printf("%d\n", numbers[4]); 

没错，对于关键字const声明的变量在C语言中，编译会报错；但是在JavaScript里，一点问题也没有。

为什么呢？
其实原因在于，C语言中，const表述定义一个只读（read-only）变量。这个变量是不可被更改的。但是在JavaScript中，const表示的并不是变量值不可变，并不意味着变量值是个绝对常量或者只读常量。

那么，JavaScript里，关键字为什么脑残的也叫“const”呢？const究竟从何而来？
其实，const关键字顾名思义，一定是确定了某种紧密恒定关系。事实上，JavaScript里，const实际上保证的是变量指向的那个内存地址不得改动。对于简单类型的数据（数值、字符串、布尔值），值就保存在变量指向的那个内存地址，因此等同于常量。但对于复合类型的数据（主要是对象和数组），变量指向的内存地址，保存的只是一个指针，const只能保证这个指针是固定的，至于它指向的数据结构是不是可变的，就完全不能控制了。

我完全可以通过下面代码来证明：

    const numbers = [1, 2, 3, 4, 6]
    numbers = [7, 8, 9, 10, 11] // error: assignment to constant variable
    console.log(numbers[4])


## 深入理解
为了更加深入理解上面的逻辑，我们需要理解JavaScript基本类型和引用类型不同的存储方式：堆存储和栈存储。

社区上关于这个概念的讲解不少，其实大体都可以总结为一下这么两个方面：

1. 基本计算机概念
两者都是存放临时数据的地方。
栈是先进后出的。
堆是在程序运行时（而不是在程序编译时），申请某个大小的内存空间。即动态分配内存，对其访问和对一般内存的访问没有区别。对于堆，我们可以随心所欲的进行增加变量和删除变量，不用遵循次序。
栈区（stack）由编译器自动分配释放 ，存放函数的参数值，局部变量的值等。 
堆区（heap）一般由程序员分配释放，若程序员不释放，程序结束时可能由OS回收。 

2. JavaScript两种类型
基本数据类型：基本数据类型值指保存在栈内存中的简单数据段。访问方式是按值访问。
引用数据类型：引用数据类型值指保存在堆内存中的对象。也就是，变量中保存的实际上的只是一个指针，这个指针指向内存中的另一个位置，该位置保存着对象。访问方式是按引用访问。

还好吗？如果你被上面大段专业术语搞得云里雾里，可以去再找写资料进行了解。这里不再科普，针对上面示例代码，我画一个图，相信会很快明白：

    图片

numbers这个数组变量属于引用类型。他的存储包含两大部分：
栈内存中保存了变量标识符numbers和指向堆内存中该对象／数组的指针；
堆内存中保存了对象／数组的实际内容。

而const关键字，保证了第一个左边短箭头的指定关系。numbers这个数组变量标识符不能指向其他的引用指针。因此，

    const numbers = [1, 2, 3, 4, 6]
    numbers = [7, 8, 9, 10, 11]

就会报错，他的做法是把numbers指向了了其他的引用指针，该指针指向的是一个新的数组。

而右边的箭头并没有指定关系。我们可以任意改变[1, 2, 3, 4, 6]这个数组的内容。如先前所示。


## 如何产生一个完全不可变的数据
关于不可变数据，我之前的[一篇文章有过深入讲解](http://www.jianshu.com/p/89f1d4245b20)，这里所说的产生一个完全不可变数据，就类似于C语言当中的const用法。

当然，如果你理解了前文，就知道基本类型的变量，用const声明，永远都是不可变的：

    const a = 10
    a = a + 1 // error: assignment to constant variable

    const isTrue = true
    isTrue = false // error: assignment to constant variable

    const sLower = 'hello world'
    const sUpper = sLower.toUpperCase() // create a new string
    console.log(sLower) // print hello world
    console.log(sUpper) // print HELLO WORLD

那么对于一个引用类型，若想实现不可变性，我们可以使用ES5当中新增的Object.freeze方法。发方法产生一个“被冻结”的对象，具体用法不再科普，可以看下例子：


    const me = Object.freeze({name: “Jacopo”})
    me.age = 28
    console.log(me.age) // print undefined

    # Example 5
    const arr = Object.freeze([-1, 1, 2, 3])
    arr[0] = 0
    console.log(arr[0]) // print -1

但是他的问题在于，该方法只能对完全的“属性－值”（property-value pair）范式的对象起作用。也就是说，类似Date, Map and Set这种对象，该方法无解。
另外，这也是一种shallow处理，如果对象存在嵌套，超过一层的话，也无法完全冻结：

    const me = Object.freeze({
        name: 'Jacopo', 
        pet: {
            type: 'dog',
            name: 'Spock'
        }
    })
    me.pet.name = 'Rocky'
    me.pet.breed = 'German Shepherd'
    console.log(me.pet.name) // print Rocky
    console.log(me.pet.breed) // print German Shepherd

如果想把一个对象完全冻结（包括其嵌套对象），你可以递归实现一个deepFreeze()方法。实现思路并不难，使用递归就可以，比如：

    function deepFreeze(obj) {
        var propNames = Object.getOwnPropertyNames(obj);

        propNames.forEach(function(name) {
            var prop = obj[name];

            if (typeof prop == 'object' && prop !== null)
            deepFreeze(prop);
        });

        return Object.freeze(obj);
    }

    obj2 = {
        internal: {}
    };

    deepFreeze(obj2);
    obj2.internal.a = 'anotherValue';
    obj2.internal.a; // undefined

同时，我们还可以使用Immutable.js类库，他本身并不会实现冻结一个对象，但是提供了很对不可变数据类型。比如：List, Stack, Map, OrderedMap, Set, OrderedSet and Record等等。



## var, let or const?
到这里，我们已经深入理解了const这个关键字。那么，在实践中，该如何处理或者使用var， let or const这三个声明方式呢？
在ES6环境下，开发者应该永远避免使用var去声明一个变量或者常量。它现在是最没有语义指导性的关键字。

合理的使用 var， let 和 const对于保证代码的干净和可读性具有重要意义。既然const用来表明标识符的引用无法被再分配（the identifier can’t be reassigned），他应该用在所有标识符的引用固定不变的代码中。如果标识符引用会发生变化，我们应当使用let来代替。举个例子，对于大部分循环中的计数器index，我们需要使用let来声明，这是一种极好的做法。同样，在子程序中，值会被交换替换的情况，当然也需要使用let来声明。



## 总结
这篇文章深入分析了ES6中的const，并尝试实现了一个冻结对象的方法。
其实javascript语言的灵活性体现在很多层面：基于原型链的继承，执行环境上下文、this指向等等。
今天这些当然只是庐山一面，想象一下，不论是let还是const，或者是var，都是我们平时写程序天天用到的。理解其中内涵，不仅可以减少BUGs的出现，也会让我们对这门语言有更深入的了解。

