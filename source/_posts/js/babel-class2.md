---
title: 揭秘babel之class super魔法处理
categories: js
tags: js
date: 2017-01-12
author: 侯策
---

如果你已经看过第一篇[揭秘babel的魔法之class魔法处理](http://www.jianshu.com/p/d36fb31f9cff)，这篇将会是一个延伸；
如果你还没看过，并且也不想[现在就去读一下](http://www.jianshu.com/p/d36fb31f9cff)，单独看这篇也没有关系。

上一篇针对Babel对ES6里面基础“class”的编译进行分析，这一篇将会对class的继承，包括extends和super进行讲解。

什么？你还不了解ES6环境的继承？推荐参考[阮一峰《ES6 标准入门》的相关章节](http://es6.ruanyifeng.com/#docs/class#Class的继承)

再啰嗦一句，这一系列的文章并不是科普ECMAScript新规范。她的意义在于分析Babel对ES6的编译，从而希望读者对JS语言基础，设计理念等等有更深刻的认识。


## Class的继承
在这篇文章中，我会讲解Babel如何处理ES6 Class里面的继承功能，同样，这其实是一系列语法糖的实现。
我们先来温习一下实现方式：

### ES6 实现继承
首先，我们定义一个父类：

    class Person {
        constructor(){
            this.type = 'person'
        }
    }

然后，实现一个Student类，这个“学生”类继承“人”类：

    class Student extends Person {
        constructor(){
            super()
        }
    }

当然，从简出发，我们定义的Person类只包含了name为person这一个属性，不含有方法。所以我们extends+super()之后，Student类
也继承了同样的属性。
如下：

    var student1 = new Student();
    student1.type // "person"

我们进一步可以验证原型链上的关系

    student1 instanceof Student // true
    student1 instanceof Person // true
    student1.hasOwnProperty('type') // true

一切看上去cool极了，我们实现了ES6里面的继承。并且用instanceof验证了ES6中一系列的实质就是“魔法糖”的本质。
那么，经过Babel编译，我们的代码是什么样呢？


### Babel transformation
我们一步一步来看，

Step1: Person定义
我们从最头来看：

    class Person {
        constructor(){
            this.type = 'person'
        }
    }

被编译为：

    var Person = function Person() {
        _classCallCheck(this, Person);
        this.type = 'person';
    };

如果你看过这一篇的[前传](http://www.jianshu.com/p/d36fb31f9cff)，
你应该很明白这一系列的变换，也可能会记得_classCallCheck函数到底是什么鬼。这里因为篇幅和去冗余的原因，就不再展开。


Step2：Student探秘
我们这次尝试观察Student子类：

    class Student extends Person {
        constructor(){
            super()
        }
    }

编译结果：

    var Student = (function(_Person) {
        _inherits(Student, _Person);

        function Student() {
            _classCallCheck(this, Student);            
            _get(Object.getPrototypeOf(Student.prototype), 'constructor', this).call(this);
        }

        return Student;
    })(Person);

    var _get = function get(_x, _x2, _x3) {
        var _again = true;
        _function: while (_again) {
            var object = _x,
                property = _x2,
                receiver = _x3;
            _again = false;
            if (object === null) object = Function.prototype;
            var desc = Object.getOwnPropertyDescriptor(object, property);
            if (desc === undefined) {
                var parent = Object.getPrototypeOf(object);
                if (parent === null) {
                    return undefined;
                } else {
                    _x = parent;
                    _x2 = property;
                    _x3 = receiver;
                    _again = true;
                    desc = parent = undefined;
                    continue _function;
                }
            } else if ('value' in desc) {
                return desc.value;
            } else {
                var getter = desc.get;
                if (getter === undefined) {
                    return undefined;
                }
                return getter.call(receiver);
            }
        }
    };

    function _inherits(subClass, superClass) {
        if (typeof superClass !== 'function' && superClass !== null) {
            throw new TypeError('Super expression must either be null or a function, not ' + typeof superClass);
        }
        subClass.prototype = Object.create(superClass && superClass.prototype, {
            constructor: {
                value: subClass,
                enumerable: false,
                writable: true,
                configurable: true
            }
        });
        if (superClass) Object.setPrototypeOf ? Object.setPrototypeOf(subClass, superClass) : subClass.__proto__ = superClass;
    }

看上去恶心极了！没关系，下面我们进行拆解，并对代码加上注释，你很快就能明白。


Step3：抽丝剥茧
我们首先看Student的编译结果

    var Student = (function(_Person) {
        _inherits(Student, _Person);

        function Student() {
            _classCallCheck(this, Student);            
            _get(Object.getPrototypeOf(Student.prototype), 'constructor', this).call(this);
        }

        return Student;
    })(Person);

这是一个自执行函数，它接受一个参数Person（就是他要继承的父类），返回一个构造函数Student。

_inherits方法的本质其实就是让Student子类继承Person父类原型链上的方法。它实现原理可以归结为一句话：

    Student.prototype = Object.create(Person.prototype);
    Object.setPrototypeOf(Student, Person)

同时利用Object.create可以接收第二个参数来实现对Student的constructor修复。
如果你不了解Object.create，那么安利一下，[请参考](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/create)

上面，通过_inherits实现了对父类原型链上属性的继承，那么对于父类的实例属性（就是constructor定义的属性）的继承，也可以归结为一句话：

    Person.call(this);

如果你还不理解使用call或者apply或者bind来改变JS中this的指向，那么请参考[这篇文章](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/call)

这样，我们便透析了Babel编译的这一切秘密。上面的代码我加上了自己的理解和注释，你可以参考：


    // 实现定义Student构造函数，它是一个自执行函数，接受父类构造函数为参数
    var Student = (function(_Person) {
        // 实现对父类原型链属性的继承
        _inherits(Student, _Person);
        
        // 将会返回这个函数作为完整的Student构造函数
        function Student() {
            // 使用检测
            _classCallCheck(this, Student);  
            // _get的返回值可以先理解为父类构造函数       
            _get(Object.getPrototypeOf(Student.prototype), 'constructor', this).call(this);
        }

        return Student;
    })(Person);

    // _x为Student.prototype.__proto__
    // _x2为'constructor'
    // _x3为this
    var _get = function get(_x, _x2, _x3) {
        var _again = true;
        _function: while (_again) {
            var object = _x,
                property = _x2,
                receiver = _x3;
            _again = false;
            // Student.prototype.__proto__为null的处理
            if (object === null) object = Function.prototype;
            // 以下是为了完整复制父类原型链上的属性，包括属性特性的描述符
            var desc = Object.getOwnPropertyDescriptor(object, property);
            if (desc === undefined) {
                var parent = Object.getPrototypeOf(object);
                if (parent === null) {
                    return undefined;
                } else {
                    _x = parent;
                    _x2 = property;
                    _x3 = receiver;
                    _again = true;
                    desc = parent = undefined;
                    continue _function;
                }
            } else if ('value' in desc) {
                return desc.value;
            } else {
                var getter = desc.get;
                if (getter === undefined) {
                    return undefined;
                }
                return getter.call(receiver);
            }
        }
    };

    function _inherits(subClass, superClass) {
        // superClass需要为函数类型，否则会报错
        if (typeof superClass !== 'function' && superClass !== null) {
            throw new TypeError('Super expression must either be null or a function, not ' + typeof superClass);
        }
        // Object.create第二个参数是为了修复子类的constructor
        subClass.prototype = Object.create(superClass && superClass.prototype, {
            constructor: {
                value: subClass,
                enumerable: false,
                writable: true,
                configurable: true
            }
        });
        // Object.setPrototypeOf是否存在做了一个判断，否则使用__proto__
        if (superClass) Object.setPrototypeOf ? Object.setPrototypeOf(subClass, superClass) : subClass.__proto__ = superClass;
    }


## 总结
如果你看完这一系列的文章可能会有体会：我想表达的肯定不是ES6新特性的使用，关于这些东西有太多的文章、博客、书籍去讨论。
我是再讲Babel对这些新特性的编译产出，那为什么会在乎这些呢？
其实通过分析，我们悄然回顾了JS中很多重点以及难点概念，以及程序设计上的一些小思想。
最近面试了很多前端“新同学”：有的人框架用的很熟，可以使用React或者Vue做出页面炫酷的交互，甚至自觉SPA也不在话下；
有的人ES6、ES7了解很多，generator，async都能说出一二，仿佛Promise处理异步已经成为了“时代弃儿”。
可是同样是这些人，对原型原型链、this、作用域、闭包都没有深刻地理解和掌握。
对此，他们的解释大体都是：“哦，我看过，我了解过，不过后来用不到，就忘了。”
同样是这些页面，即便用callback处理异步回调，嵌套最多不到两层。
对此，他们的解释翻译一下都是：“放着先进的生产力我不用，我怎么做保持进步的青年？”

也许，同样一批人也会问：“我能用前端框架、ES6撸出好多页面，可是为什么感觉进步很慢、面试也总被挂呢？”





