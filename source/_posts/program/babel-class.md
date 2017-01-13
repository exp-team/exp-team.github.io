---
title: 揭秘babel的魔法之class魔法处理
categories: program
tags: program
date: 2017-01-12
author: 侯策
---

2017，很多人已经开始接触ES6环境，并且早已经用在了生产当中。我们知道ES6在大部分浏览器还是跑不通的，因此我们使用了伟大的Babel来进行编译。很多人可能没有关心过，经过Babel编译之后，我们华丽的ES6代码究竟变成了什么样子？

这篇文章，针对Babel对ES6里面“类class”的编译进行分析，你可以[在线测试编译结果](https://babeljs.io/repl/)，毕竟纸上得来终觉浅，自己动手，才能真正体会其中的奥秘。

另外，这篇文章并不适合初学者阅读。如果你还不明白JS中原型链等OOP相关知识，建议出门左转找到经典的《JS高级程序设计
》来补课；如果你对JS中，通过原型链来实现继承一直云里雾里，安利一下[颜海镜大大的文章](http://yanhaijing.com/javascript/2014/11/09/object-inherit-of-js/)

<!-- more -->

## 为什么使用选择Babel
我们知道，现在大部分浏览器或者类似NodeJS的javascript引擎还不能直接支持Babel。但这并不是障碍，比如Babel的出现，使得在生产环境中书写ES6代码成为了现实，它工作原理是编译ES6的新特性为老版本的ES5，从而得到宿主环境的支持。

## Class例子
在这篇文章中，我会讲解Babel如何处理ES6新特性：Class，这其实是一系列语法糖的实现。

### Old school方式实现继承
在探究ES6之前，我们先来回顾一下ES5环境下，我们如何实现类的继承：

    // Person是一个构造器
    function Person(name) {
        this.type = 'Person';
        this.name = name;
    }

    // 我们可以通过prototype的方式来加一条实例方法
    Person.prototype.hello = function() {
        console.log('hello ' + this.name);
    }

    // 对于私有属性(Static method)，我们当然不能放在原型链上了。我们可以直接放在构造函数上面
    Person.fn = function() {
        console.log('static');
    };

我们可以这么应用：

    var julien = new Person('julien');
    var darul = new Person('darul');
    julien.hello(); // 'hello julien'
    darul.hello(); // 'hello darul'
    Person.fn(); // 'static'

    // 这样会报错，因为fn是一个私有属性
    julien.fn(); //Uncaught TypeError: julien.fn is not a function


### New school方式(ES6)实现继承
在ES6环境下，我们当然迫不及待地试一试Class

    class Person {
        constructor(name) {
            this.name = name;
            this.type="person"
        }
        hello() {
            console.log('hello ' + this.name);
        }
        static fn() {
            console.log('static');
        };
    }

这样写起来当然很cool，但是经过Babel编译，我们的代码是什么样呢？

### Babel transformation
我们一步一步来看，

Step1: 定义
我们从最简单开始，试试不加任何方法和属性的情况下，

    Class Person{}

被编译为：

    function _classCallCheck(instance, Constructor) {
        // 检查是否成功创建了一个对象
        if (!(instance instanceof Constructor)) {  
            throw new TypeError("Cannot call a class as a function"); 
        } 
    }

    var Person = function Person() {
        _classCallCheck(this, Person);
    };

你可能会一头雾水,_classCallCheck是什么？其实很简单，它是为了保证调用的安全性。
比如我们这用调用

    // ok
    new p = new Person();

是没有问题的，但是直接调用：
    
    // Uncaught TypeError: Cannot call a class as a function
    Person();

就会报错，这就是_classCallCheck所起的作用。具体原理自己看代码就好了，很好理解。


我们发现，Class关键字会被编译成构造函数，于是我们便可以通过new来实现实例的生成。

Step2：Constructor探秘
我们这次尝试加入constructor,再来看看编译结果

    class Person() {
        constructor(name) {  
            this.name = name;
            this.type = 'person'
        }
    }

编译结果：

    var Person = function Person(name) {
        _classCallCheck(this, Person);
        this.type = 'person';
        this.name = name;
    };

看上去棒极了，我们继续探索。

Step3：增加方法
我们尝试给Person类添加一个方法：hello

    class Person {
        constructor(name) {
            this.name = name;
            this.type = 'person'
        }

        hello() {
            console.log('hello ' + this.name);
        }
    }

编译结果(已做适当省略)：

    // 如上，已经解释过
    function _classCallCheck.... 

    // MAIN FUNCTION
    var _createClass = (function () { 
        function defineProperties(target, props) { 
            for (var i = 0; i < props.length; i++) { 
                var descriptor = props[i]; 
                descriptor.enumerable = descriptor.enumerable || false; 
                descriptor.configurable = true; 
                if ('value' in descriptor) 
                descriptor.writable = true; 
                Object.defineProperty(target, descriptor.key, descriptor); 
            } 
        } 
        return function (Constructor, protoProps, staticProps) { 
            if (protoProps) 
                defineProperties(Constructor.prototype, protoProps); 
            if (staticProps) 
                defineProperties(Constructor, staticProps); 
            return Constructor; 
        }; 
    })();

    var Person = (function () {
        function Person(name) {
            _classCallCheck(this, Person);

            this.name = name;
        }

        _createClass(Person, [{
            key: 'hello',
            value: function hello() {
                console.log('hello ' + this.name);
            }
        }]);

        return Person;
    })();

Oh...no,看上去有很多需要消化!不要急，我尝试先把他精简一下，并加上注释，你就会明白核心思路：

    var _createClass = (function () {   
        function defineProperties(target, props) { 
            // 对于每一个定义的属性props，都要完全拷贝它的descriptor,并扩展到target上
        }  
        return defineProperties(Constructor.prototype, protoProps);    
    })();

    var Person = (function () {
        function Person(name) { // 同之前... }

        _createClass(Person, [{
            key: 'hello',
            value: function hello() {
                console.log('hello ' + this.name);
            }
        }]);

        return Person;
    })();

如果你不明白defineProperty方法, [请参考这里](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)

现在，我们知道我们添加的方法

    hello() {
        console.log('hello ' + this.name);
    }

被编译为：

    _createClass(
        Person, [{
        key: 'hello',
        value: function hello() {
            console.log('hello ' + this.name);
        }
    }]);

而_createClass接受2个－3个参数，分别表示：

    参数1 => 我们要扩展属性的目标对象，这里其实就是我们的Person
    参数2 => 需要在目标对象原型链上添加的属性，这是一个数组
    参数3 => 需要在目标对象上添加的属性，这是一个数组

这样，Babel的魔法就一步一步被揭穿了。

## 总结
希望这篇文章能够让你了解到Babel是如何初步把我们ES6 Class语法编译成ES5的。下一篇文章我会继续介绍Babel如何处理Super(), 并会通过一段函数桥梁，使得ES5环境下也能够继承ES6定义的Class



