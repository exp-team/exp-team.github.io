---
title: JavaScript事件章分享
categories: program
tags: program
date: 2016-9-4
author: 赵文琳
---
最近做了一个启明星计划的分享，至于为什么叫启明星呢，这个得问我偶像，我们是一个造星平台，欢迎大家踊跃报名~

本文是我整理的一些关于原生事件的东西。

<!-- more -->

## DOM事件流
DOM事件流主要分为:事件捕获阶段->处于目标阶段->事件冒泡阶段，但是IE的事件流里没有捕获阶段。
![](/bimg/b425.png)

> 在一个事件完成了所有阶段的传播路径后，它的event.currentTarget会被设置为null并且event.eventPhase会被设为0。event的所有其他属性都不会改变（包括指向事件目标的Event.target属性）

## 事件处理程序
事件处理程序主要分为以下三种：

![](/bimg/b426.png)

ddEventListener是W3C工作组在DOM Level 2开始引入的一个注册事件监听器的方法；而在此之前，传统的事件监听方法是通过element['on' + type]的方式来注册的。

它们两之间的主要区别是，element['on' + type]的方式无法使用事件捕获,attachEvent也不会支持捕获。


**注：**element['on' + type]不支持对同一个元素的同一个事件注册多个事件监听器;
后面注册的事件处理程序会覆盖前面的事件处理事件程序。

### 原生事件对象：
window.event是IE支持的原生事件对象，event是其他浏览器是原生事件对象，这里列举的是事件对象的常用属性。

![](/bimg/b427.png)

->其实在DOM中的事件对象也有bubbles和defaultPrevented属性可以赋值来表示冒泡和默认事件。

### 原生事件类型
    
- UI事件
- 焦点事件
- 鼠标事件
- 滚动事件
- 文本事件
- 键盘事件
- HTML5事件

### UI事件：用户与页面交互
- load：加载页面的时候触发
- unload：卸载页面的时候触发
- abort：加载被终止的时候被触发
- error：js错误、无法加载图像或者其他内容
- select：当用户选择文本框中的一个或者多个字符时触发
- resize：窗口或者框架的大小发生变化
- scroll：滚动带滚动条的元素

**注：**select在IE8之前不必释放鼠标就可以触发，resize在初始化页面的时候不会被触发。


### 焦点事件：获取或者失去焦点
- blur：失去焦点
- focus：获得焦点
- focusin：与focus等价，冒泡
- focusout：与blur等价，冒泡

Blur和focus在JavaScript早期就得到了所有浏览器的支持，后来IE出了focusout和focusin支持冒泡，Opera的DOMFocusout以及DOMFocusin也支持冒泡，但是后来DOM3采用了IE的作为标准，当页面从一个元素移动到另外一个页面的时候依次会触发：focusout-focusin-blur-focus，


###鼠标和滚动事件：通过鼠标滚动发生
- click：单击
- dbclick ：双击
- mousedown：鼠标按下
- mouseup：鼠标释放
- mouseenter：首次移动到目标元素，并且在子元素上不会重复触发，不冒泡
- mouseleave：与mouseenter对应
- mousemove：当鼠标移动时触发
- mouseout：鼠标从一个元素移动到另一个元素时触发，这个可以是其子元素
- mouseover：与mouseout对应 

**注：**我们在执行双击操作的时候会经过mousedown->mouseup->click->mousedown->mouseup->dbclick，但是IE8之前会省略第二个mousedown；
鼠标事件event.button的值一共分为以下8个值：
- 0：表示没有按下按钮
- 1：按下了主鼠标
- 2：按下次鼠标
- 3：按下次鼠标
- 4：同时按下主次鼠标
- 5：主和中
- 6：次和中
- 7：三个键一起
**注：**IE中7会返回0,6返回2,4返回1。


这里主要区别以下mouseenter与mouseover的区别，来看一下以下这个图：

![](/bimg/b428.png)

从这个图可以看到，当我们的鼠标从C元素移动到B元素的时候，会在B元素上触发mouseenter和mouseover，在C元素上触发mouseout和mouseleave，当鼠标从B元素移动到A元素上的时候，在B元素上会触发mouseout但是不会触发mouseleave，因为A元素是B元素的子元素。

AB元素互为相对元素，BC互为相对元素，在IE中相对元素是保存在FromElement，和toElement属性中，其他浏览器则保存在relateTarget。

### 滚动鼠标事件

![](/bimg/b429.png)

**注：**正常的值应该是120的倍数，Opera的wheelDelta的值的正负号是颠倒的，Firefox的detail值是3的倍数

###键盘与文本事件：在文档中输入文本以及通过键盘在
页面上执行操作时

- keydown：键盘任意键按下
- keyup：键盘任意键释放
- keypress：按住键盘任意键不放会重复触发
- textInput：在文本插入文本框之前触发
- even.keyCode:键码，发生在keydown和keyup事件
- Event.charcode:字符编码，发生在keypress事件
- DOM3：key || keyIdentifier / char

前三个事件所有浏览器都支持，textInput是对keypress的补充，用意在于将文本显示给用户之前更容易拦截文本，按下键盘时能影响到文本的显示的时候会触发keypress，
如果用户按下了一个非字符的键，则会一直触发keydown，不会触发keypress。
键盘每个键都对应一个keyCode，DOM3在字符键按下的时候key和char一样，在非字符键按下char返回null。



### HTML5事件：

- contextmenu：可以取消默认的上下文菜单
- beforeunload：在页面被卸载前触发
- DOMcontentLoaded：在形成完整的dom树之后就会触发
- readystatechange：提供文档或者元素的加载状态
- Pageshow:页面被打开时触发（document上触发）
- Pagehide：在浏览器卸载时触发，在unload之前
- Hashchange:url的参数列表发生变化时触发

**注：**每个窗口都会保存着用户的浏览记录，包括页面的DOM和js的状态，如果不关闭窗口我们都可以在左上角那里来返回或者前进历史的浏览记录，如果页面来至于bfcache，不会触发load事件，但是会触发Pageshow和Pagehide

### jquery的事件绑定

- bind()  不能给创建的元素添加事件
- delegate() 通过事件委托加在祖先元素上
- live() 通过事件委托加在document上，比较消耗性能
- on() bind和delegate函数最终都是调用了on函数，只是改变了传参方式，让其更语义化一点


### jquery.on所做的事情

- 兼容地实现了addEventListener的全部功能，包括多事件绑定事件函数按绑定顺序执行
- 动态事件绑定
- 自定义事件，自定义事件还能冒泡
- 事件函数分组，方便事件函数解绑




