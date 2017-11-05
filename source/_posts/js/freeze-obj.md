---
title: JS冻结对象的《人间词话》 完美实现究竟有几层？
categories: js
tags: js
date: 2017-06-09
author: 侯策
---

王国维在《人间词话》里谈到了治学经验，他说：“古今之成大事业、大学问者，必经过三种之境界。”

巧合的是，最近受 [git chat ／ git book](http://gitbook.cn/) 邀请，做了一个分享。
其中谈到JS中冻结一个对象几种由浅入深的实践。想想也暗合国学大师所谓的三重境界。

这篇文章由浅入深讨论JS中对象的一些锁定特性。但都是一些基础语法的实现，相信即便是前端小白也可以大体领会。不过需要读者预先了解JS中对象的特性，尤其是对象自身属性的描述符：configurable、writable...


另外，如果您对JS中对象操作、不可变数据、函数式编程感兴趣，同样推荐我的其他一些相关文章：

- [如何优雅安全地在深层数据结构中取值](http://www.jianshu.com/p/11fc75f28302)
- [从JS对象开始，谈一谈“不可变数据”和函数式编程](http://www.jianshu.com/p/89f1d4245b20)
- [解析Twitter前端架构 学习复杂场景数据设计](http://www.jianshu.com/p/7a56ac1de2a8)

等等。

## 昨夜西风凋碧树 独上高楼 望尽天涯路

第一种境界：“昨夜西风凋碧树，独上高楼，望尽天涯路。”
该词句出自晏殊的《蝶恋花》，原意是说，“我”上高楼眺望所见的更为萧飒的秋景，西风黄叶，山阔水长，案书何达？

王国维此句中解成：做学问成大事业者，首先要有执着的追求，登高望远，瞰察路径，明确目标与方向，了解事物的概貌。

 **我们就从最基本的场景说起，究竟为什么要冻结一个对象？**

- 场景一：
我们造了一个轮子，对外暴露一个对象，开放出来给第三方使用。同时需要保证这个对外暴露的对象完全安全，不能被业务代码所改写覆盖或下钩子（hook）函数。

- 场景二：
如果你看过Vue 2.* 版本源码，你会发现冻结一个对象的操作频繁出现。

**我们先来看冻结对象的第一层实现 —— 扩展特性锁：**

他包含了两个基本方法：

- Object.isExtensible
- Object.preventExtensions

如果一个对象可以添加新的属性，则这个对象是可扩展的。扩展特性锁就是让这个对象变的不可扩展，也就是不能再有新的属性。


### Object.isExtensible

MDN上内容概述：

    概述
        Object.isExtensible() 方法判断一个对象是否是可扩展的（是否可以在它上面添加新的属性）。
    语法
        Object.isExtensible(obj)
    参数
        obj 需要检测的对象


例如，我们正常使用对象字面量声明的对象都是可扩展的：

    var person1 = {};
    person1.name = "Lucas";
    console.log(person1);
    // {name: "Lucas"}

同时：

    Object.isExtensible(person1) === true; // true

你可能要问了，那么使用Object.create方法声明的对象，并对该对象属性进行配置是什么情况呢？
我们知道，用上面对象字面量声明的对象相当于：

    var person1 = Object.create({},{
        "name":{
            value : "Lucas",
            configurable : true, //不可配置
            enumerable : true , //可枚举
            writable : true //可写
        }
    });

即便尝试将configurable设置为false:

    var person1 = Object.create({},{
        "name":{
            value : "Lucas",
            configurable : false, //不可配置
            enumerable : true, //可枚举
            writable : true //可写
        }
    });

仍然得到：

    Object.isExtensible(person1) === true; // true


## Object.preventExtensions

当然，我们还是有方法可以使得一个对象变的不可扩展。

MDN上内容概述：

    概述
        Object.preventExtensions() 方法让一个对象变的不可扩展，也就是永远不能再添加新的属性。
    语法
        Object.preventExtensions(obj)
    参数
        obj 将要变得不可扩展的对象

几个注意点包括但不限于：

- 不可扩展的对象的属性通常**仍然可以被删除。**
- 尝试给一个不可扩展对象添加新属性的操作将会失败，不过**可能是静默失败，也可能会抛出 TypeError 异常**（严格模式下）。
- Object.preventExtensions 只能阻止一个对象不能再添加新的自身属性，**仍然可以为该对象的原型添加属性。**

比如：

    var person1 = {
        name: "Lucas"
    }
    
    Object.preventExtensions(person1);
    person1.age = 18;
    // 非严格模式下，这里不会有报错，属于静默失败
    person1.age // undefined
    // 扩展新属性失败了

仍然可以向原型链添加属性：

    person1.__proto__.age = 18;
    person1.age // 18
    // 可以从原型链上取到

同样也可以复写一些属性：

    person1.name = "Eros";
    person1.name // "Eros"

也可以删除已有属性：

    person1.name; //  "Eros",
    delete person1.name;
    person1.name; // undefined


通过以上方法，我们实现了对一个对象属性扩展的冻结。但是同样也认识到，这并不是全面的保护：例如可以随意改动去覆盖已有属性，在对象原型链上增加属性也还是难以屏蔽。


## 衣带渐宽终不悔 为伊消得人憔悴

第二种境界：“衣带渐宽终不悔，为伊消得人憔悴。”
这引用的是北宋柳永《蝶恋花》最后两句词，原词是表现作者对爱的艰辛和爱的无悔。若把“伊”字理解为词人所追求的理想和毕生从事的事业，亦无不可。王国维则别有用心，以此两句来比喻成大事业、大学问者，不是轻而易举，随便可得的，必须坚定不移，经过一番辛勤劳动，废寝忘食，孜孜以求，直至人瘦带宽也不后悔。

**下面介绍一个更深一层的做法：密封特性。**

密封对象是指那些不能添加新的属性，不能删除已有属性，以及不能修改已有属性的可枚举性（enumerable）、可配置性（configurable）、可写性（writable），但**可能可以修改已有属性的值的对象。**

他同样包含了两个基本方法：

- Object.isSealed
- Object.seal

### Object.isSealed

MDN上内容概述：
    
    概述 
        Object.isSealed() 方法判断一个对象是否是密封的（sealed）。
    语法 
        Object.isSealed(obj)
    参数
        obj 将要检测的对象

正常对象字面量声明的对象是不被密封的：

    var person1 = {
        name: "Lucas"
    }

    Object.isSealed(person1); // false

当将这个对象禁止扩展时，它也不会变成密封的：

    var person1 = {
        name: "Lucas"
    }
    Object.preventExtensions(person1);
    Object.isSealed(person1); // false

但是在此基础上，使用Object.defineProperty方法，把属性变得不可配置（configurable），则这个对象也就成了密封对象：

    var person1 = {
        name: "Lucas"
    }

    Object.defineProperty(person1, "name", {configurable : false});

    Object.isSealed(person1); // true

此时，我们有：

    Object.getOwnPropertyDescriptor(person1, 'name');
    // 得到：
    Object {
        value: "Lucas", 
        writable: true, 
        enumerable: true, 
        configurable: false
    }

根据这个getOwnPropertyDescriptor，我们可以更加深入的理解密封特性：被密封的对象，就是在不可扩展基础上讲属性描述符configurable设置为false; 同时，**被密封的对象，仍然有机会改变属性的值。**只不过对于此对象本身而言，**不可以再扩展新的属性，不可以更改已有属性的配置信息。**


### Object.seal

相对应我们也有一个方法将一个对象密封。

MDN上内容概述：

    概述
        Object.seal() 方法可以让一个对象密封，并返回被密封后的对象。
    语法
        Object.seal(obj)
    参数
        obj 将要被密封的对象


比如：

    var person1 = {
        name: "Lucas"
    }
    Object.getOwnPropertyDescriptor(person1, 'name');
    // 得到：
    Object {
        value: "Lucas", 
        writable: true, 
        enumerable: true, 
        configurable: true
    }
    
将此对象密封后：

    Object.seal(person1);
    Object.getOwnPropertyDescriptor(person1, 'name');
    // 得到：
    Object {
        value: "Lucas", 
        writable: true, 
        enumerable: true, 
        configurable: false
    }

也就是说：

    person1.age = 18;
    person1.age; // undefined
    // 扩展新属性失败
    
    // 同时调用defineProperty失败
    Object.defineProperty(person1,"name",{get : function(){return "g";}});
    // 抛出异常
    
任何除更改属性值以外的操作，非严格模式下都会静默失败，如上并如下：

    delete person1.name;
    person1.name; // "Lucas"

而更改属性值可以成功：

    person1.name = "Eros";
    person1.name; // "Eros"

怎么理解这样的现象呢？牢记，被密封的对象拥有如下的属性描述符：

    Object {
        value: "Lucas", 
        writable: true, 
        enumerable: true, 
        configurable: false
    }

**而删除属性属于configurable，更改属性才属于writable；**


### 一点延伸

借助于此，我们其实已经可以完成冻结对象的第三重境界：达到即密封又不可修改原属性值。因为可以这样做：

    var person1 = {name: "Lucas"};
    Object.defineProperty(person1, "name", {configurable: false, writable: false});
    Object.preventExtensions(o);

总结下就是设置：

> configurable: false ＋ writable: false ＋ preventExtensions

或者因为

> configurable: false＋ preventExtensions ＝ seal

所以也可以设置：

> seal + writable: false


## 众里寻他千百度，蓦然回首，那人却在，灯火阑珊处

第三种境界：“众里寻他千百度，蓦然回首，那人却在，灯火阑珊处。”
这是引用南宋辛弃疾《青玉案》词中的最后四句。梁启超称此词“自怜幽独，伤心人别有怀抱”。这是借词喻事，与文学赏析已无交涉。王国维已先自表明，“吾人可以无劳纠葛”。他以此词最后的四句为“境界”之第三，即最终最高境界。

这虽不是辛弃疾的原意，但也可以引出悠悠的远意：做学问、成大事业者，要达到第三境界，必须有专注的精神。反复追寻、研究，下足功夫，自然会豁然贯通，有所发现，有所发明，就能够从必然王国进入自由王国。

上边那种冻结对象的方法，其实也有原生实现，可谓：“众里寻他千百度，蓦然回首，那人却在，灯火阑珊处”

我们这里所说的一个对象的冻结（frozen）是指它不可扩展，所有属性都是不可配置的（non-configurable），且所有数据属性（data properties）都是不可写的（non-writable）。

或者说，冻结对象是指那些不能添加新的属性，不能修改已有属性的值，不能删除已有属性，以及不能修改已有属性的可枚举性、可配置性、可写性的对象。也就是说，这个对象永远是不可变的。

同样，包含了两个基本方法：

- Object.isFrozen
- Object.freeze

### Object.isFrozen

MDN上内容概述：

    概述
        Object.isFrozen() 方法判断一个对象是否被冻结（frozen）。
    语法
        Object.isFrozen(obj)
    参数
        obj 被检测的对象


### Object.freeze 方法

MDN上内容概述：

    概述
        Object.freeze() 方法可以冻结一个对象。
    语法
        Object.freeze(obj)
    参数
        obj 将要被冻结的对象

可以先理解为，这是最高一层的冻结对象：

    var person1 = {
        name: "Lucas"
    }

    Object.freeze(person1);

此时，我们有：

    Object.getOwnPropertyDescriptor(person1, 'name')

    Object {
        value: "Lucas", 
        writable: false, 
        enumerable: true, 
        configurable: false
    }

    // 对冻结对象的任何操作都会失败
    person1.name = "Eros"; // 改写属性值，非严格模式下静默失败;
    person1.age = 18; // 扩展属性值，非严格模式下静默失败;
    Object.defineProperty(person1,"name",{value: "Eros"}); // 使用defineProperty会直接报错

改写属性值，扩展新属性，调用defineProperty，全部都会失败。

但是，**这种层面的冻结，只是浅冻结。**如果对象里面还嵌套有对象，那么这个内部对象丝毫不受影响。

    var person1 = {
        name: "Lucas",
        family: {
            brother: "Eros"
        }
    }
    
    Object.freeze(person1);

    person1.family.brother = "Tim";
    person1.family.brother // "Tim"


### 终极实现

那么，如果我们想深层次冻结一个对象呢？思路和深拷贝暗合，使用递归：

    Object.prototype.deepFreeze = Object.prototype.deepFreeze || function (o){
        var prop, propKey;
        Object.freeze(o); // 首先冻结第一层对象
        for (propKey in o){
            prop = o[propKey];
            if(!o.hasOwnProperty(propKey) || !(typeof prop === "object") || Object.isFrozen(prop)){
                continue;
            }
            deepFreeze(prop); // 递归
        }
    }

这样子，我们再回过头来看：

    var person1 = {
        name: "Lucas",
        family: {
            brother: "Eros"
        }
    }
    
    Object.deepFreeze(person1);

    person1.family.brother = "Tim";
    person1.family.brother // "Eros"

已经达到了深层次对象属性的冻结。

## 总结

本文先后介绍了关于冻结一个对象的三种进阶方法。他们层层递进，却又相互关联。关系如图：


![关系图](http://upload-images.jianshu.io/upload_images/4363003-55750ea1c4e62989.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


文章部分概念粘取了MDN语法介绍和[Tomson的文章。](https://segmentfault.com/a/1190000003894119)

在《文学小言》一文中，王国维把上述三境界说成“三种之阶级”。并说：“未有不阅第一第二阶级而能遽跻第三阶级者，文学亦然。此有文学上之天才者，所以又需莫大之修养也。”

与大家共勉。

Happy coding!

PS: 作者[Github仓库](https://github.com/HOUCe)，欢迎通过代码各种形式交流。









