---
title: 从JS对象开始，谈一谈究竟什么是“不可变数据”和函数式编程
categories: js
tags: js
date: 2017-03-25
author: 侯策
---

作为前端开发者，你会感受到JS中对象(Object)这个概念的强大。我们说“JS中一切皆对象”。最核心的特性，例如从String，到数组，再到浏览器的APIs，对象这个概念无处不在。[这里](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Objects)你可以了解到JS Objects中的一切。

同时，随着React的强势崛起，不管你有没有关注过这个框架，也一定听说过一个概念—不可变数据(immutable.js)。究竟什么是不可变数据？这篇文章会从JS源头—对象谈起，让你逐渐了解这个函数式编程里的重要概念。

JS中的对象是那么美妙：我们可以随意复制他们，改变并删除他们的某项属性等。但是要记住一句话：

>“伴随着特权，随之而来的是更大的责任。”
(With great power comes great responsibility)

的确，JS Objects里概念太多了，我们切不可随意使用对象。下面，我就从基本对象说起，聊一聊不可变数据和JS的一切。

这篇文章缘起于Daniel Leite在本月16日的[文章：Things you should know about Objects and Immutability in JavaScript](https://www.ckl.io/blog/objects-immutability-javascript/)，我进行了翻译，并改写了用到的例子，以及进行了大量更多的扩展。

## 可变和共享是万恶之源
不可变数据其实是函数式编程相关的重要概念。相对的，函数式编程中认为可变性是万恶之源。但是，为什么会有这样的结论呢？

这个问题可能很多程序员都会有。其实，如果你的代码逻辑可变，这并不是“政治错误”的。比如JS中的数组操作，很对都会对原数组进行直接改变，这当然并没有什么问题。比如：

    let arr = [1, 2, 3, 4, 5];
    arr.splice(1, 1); // 返回[2];
    console.log(arr); // [1, 3, 4, 5];

这是我们常用的“删除数组某一项”的操作。好吧，他一点问题也没有。

问题其实出现在“滥用”可变性上，这样会给你的程序带来“副作用”。先不必关心什么是“副作用”，他又是一个函数式编程的概念。

我们先来看一下代码实例：

    const student1 = {
        school: 'Baidu',
        name: 'HOU Ce',
        birthdate: '1995-12-15',
    }

    const changeStudent = (student, newName, newBday) => {
        const newStudent = student;
        newStudent.name = newName;
        newStudent.birthdate = newBday;
        return newStudent;
    }

    const student2 = changeStudent(student1, 'YAN Haijing', '1990-11-10');

    // both students will have the name properties
    console.log(student1, student2);
    // Object {school: "Baidu", name: "YAN Haijing", birthdate: "1990-11-10"} 
    // Object {school: "Baidu", name: "YAN Haijing", birthdate: "1990-11-10"}

我们发现，尽管创建了一个新的对象student2，但是老的对象student1也被改动了。这是因为JS对象中的赋值是“引用赋值”，即在赋值过程中，传递的是在内存中的引用(memory reference)。具体说就是“栈存储”和“堆存储”的问题。具体图我就不画了，理解不了可以单找我。


## 不可变数据的强大和实现
我们说的“不可变”，其实是指保持一个对象状态不变。这样做的好处是使得开发更加简单，可回溯，测试友好，减少了任何可能的副作用。
函数式编程认为：

>只有纯的没有副作用的函数，才是合格的函数。

好吧，现在开始解释下“副作用”(Side effect)：在计算机科学中，函数副作用指当调用函数时，除了返回函数值之外，还对主调用函数产生附加的影响。例如修改全局变量（函数外的变量）或修改参数。

函数副作用会给程序设计带来不必要的麻烦，给程序带来十分难以查找的错误，并降低程序的可读性。严格的函数式语言要求函数必须无副作用。

那么我们避免副作用，创建不可变数据的主要实现思路就是：一次更新过程中，不应该改变原有对象，只需要新创建一个对象用来承载新的数据状态。

我们使用纯函数(pure functions)来实现不可变性。纯函数指无副作用的函数。
那么，具体怎么构造一个纯函数呢？我们可以看一下代码实现，我对上例进行改造：

    const student1 = {
        school: "Baidu", 
        name: 'HOU Ce',
        birthdate: '1995-12-15',
    }

    const changeStudent = (student, newName, newBday) => {
        return {
            ...student, // 使用解构
            name: newName, // 覆盖name属性
            birthdate: newBday // 覆盖birthdate属性
        }
    }

    const student2 = changeStudent(student1, 'YAN Haijing', '1990-11-10');

    // both students will have the name properties
    console.log(student1, student2);
    // Object {school: "Baidu", name: "HOU Ce", birthdate: "1995-12-15"} 
    // Object {school: "Baidu", name: "YAN Haijing", birthdate: "1990-11-10"}

需要注意的是，我使用了ES6中的解构(destructuring)赋值。
这样，我们达到了想要的效果：根据参数，产生了一个新对象，并正确赋值，最重要的就是并没有改变原对象。


## 创建纯函数，过滤副作用
现在，我们知道了“不可变”到底指的是什么。接下来，我们就要分析一下纯函数应该如何实现，进而生产不可变数据。

其实创建不可变数据方式有很多，在使用原生JS的基础上，我推荐的方法是使用现有的Objects API和ES6当中的解构赋值（上例已经演示）。现在看一下Objects.assign的实现方式：

    const student1 = {
        school: "Baidu", 
        name: 'HOU Ce',
        birthdate: '1995-12-15',
    }

    const changeStudent = (student, newName, newBday) => Object.assign({}, student, {name: newName, birthdate: newBday})

    const student2 = changeStudent(student1, 'YAN Haijing', '1990-11-10');

    // both students will have the name properties
    console.log(student1, student2);
    // Object {school: "Baidu", name: "HOU Ce", birthdate: "1995-12-15"};
    // Object {school: "Baidu", name: "YAN Haijing", birthdate: "1990-11-10"};

同样，如果是处理数组相关的内容，我们可以使用：.map, .filter或者.reduce去达成目标。这些APIs的共同特点就是不会改变原数组，而是产生并返回一个新数组。这和纯函数的思想不谋而合。

但是，再说回来，使用Object.assign请务必注意以下几点：
1）他的复制，是将所有可枚举属性，复制到目标对象。换句话说，不可枚举属性是无法完成复制的。
2）对象中如果包含undefined和null类型内容，会报错。
3）最重要的一点：Object.assign方法实行的是浅拷贝，而不是深拷贝。

第三点很重要，也就是说，如果源对象某个属性的值是对象，那么目标对象拷贝得到的是这个对象的引用。这也就意味着，当对象存在嵌套时，还是有问题的。比如下面代码：

    const student1 = {
        school: "Baidu", 
        name: 'HOU Ce',
        birthdate: '1995-12-15',
        friends: {
            friend1: 'ZHAO Wenlin',
            friend2: 'CHENG Wen'
        }
    }

    const changeStudent = (student, newName, newBday, friends) => Object.assign({}, student, {name: newName, birthdate: newBday})

    const student2 = changeStudent(student1, 'YAN Haijing', '1990-11-10');

    // both students will have the name properties
    console.log(student1, student2); 
    // Object {school: "Baidu", name: "HOU Ce", birthdate: "1995-12-15", friends: Object}
    // Object {school: "Baidu", name: "YAN Haijing", birthdate: "1990-11-10", friends: Object}

    student2.friends.friend1 = 'MA xiao';
    console.log(student1.friends.friend1); // "MA xiao"

对student2 friends列表当中的friend1的修改，同时也影响了student1 friends列表当中的friend1。


## JS本身的苍白无力和强大的不可变数据类库
以上，我们分析了纯JS如何实现不可变数据。这样处理带来的一个负面影响在于：一些经典APIs都是shallow处理，比如上文提到的Object.assign。如果遇到嵌套很深的结构，我们就需要手动递归。这样做呢，又会存在性能上的问题。

比如我自己动手用递归实现一个深拷贝，需要考虑循环引用的“死环”问题，另外，当使用大规模数据结构时，性能劣势尽显无疑。我们熟悉的jquery extends方法，某一版本（最新版本情况我不太了解）的实现是进行了三层拷贝，也没有达到完备的deep copy。

总之，实现不可变数据，我们必然要关心性能问题。针对于此，我推荐一款已经“大名鼎鼎”的——[immutable.js类库](http://facebook.github.io/immutable-js/)来处理不可变数据。

他的实现原理很有意思，下面这段话，我摘自camsong前辈的[文章](https://zhuanlan.zhihu.com/p/20295971?columnSlug=purerender)：

>Immutable实现的原理是Persistent Data Structure（持久化数据结构），也就是使用旧数据创建新数据时，要保证旧数据同时可用且不变。
同时为了避免deepCopy把所有节点都复制一遍带来的性能损耗，Immutable使用了Structural Sharing（结构共享），即如果对象树中一个节点发生变化，只修改这个节点和受它影响的父节点，其它节点则进行共享。

感兴趣的读者可以深入研究下，这是很有意思的。


## 总结
我们使用JavaScript操纵对象，这样的方式很简单便捷。但是，这样操控的基础是在JavaScript灵活的对象机制的熟练掌握上。不然很容易使你“头大”。

在我开发的百度某部门[私信](http://jingyan.baidu.com/msg)项目中，因为使用了React+Redux技术栈，并且数据结构较为负责，也采用了immutable.js实现。

最后，在前端开发中，函数式编程越来越热，并且在某种程度上已经取代了“过程式”编程和面向对象思想。

我的感想是在某些特定的场景下，不要畏惧变化，拥抱未来。
就像我很喜欢的葡萄牙诗人安德拉德一首诗中那样说的：

>我同样不知道什么是海，
赤脚站在沙滩上，
急切地等待着黎明的到来。

Happy Coding!