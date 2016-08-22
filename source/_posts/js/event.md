---
title: js原生事件总结
categories: js
tags: js
date: 2016-08-21
author: 赵文琳
---
最近把自己学习js原生事件的学习总结了一下，可供大家参考一下下，哪里有不对的地方还请大家指正哦~

<!-- more -->

事件的第一个概念就是事件流，所谓的事件流就是页面接受事件的顺序。IE的事件流叫冒泡事件，Netscape Communicator提出了另一个事件流叫事件捕获，一下两张图就能很直观的理解这两种事件流
冒泡：

![](/bimg/b2.png)

捕获：

![](/bimg/b3.png)

所有的浏览器都支持冒泡，但是具体实现还是有一些差别，有的会跳过<HTML>直接从body到文档，有的是会冒泡到window对象

事件捕获呢是不太具体的节点应该更早的接收到事件，而最具体的节点应该最后接受到事件，事件的捕获在于事件达到预期目标之前捕获它。

事件流包括三个阶段：事件捕获阶段，处于目标阶段，和冒泡阶段，
target在事件流的目标阶段；currentTarget在事件流的捕获，目标及冒泡阶段。只有当事件流处在目标阶段的时候，两个的指向才是一样的， 而当处于捕获和冒泡阶段的时候，target指向被单击的对象而currentTarget指向当前事件活动的对象（一般为父级）。这里不知道大家有没有理解，简单来说，就是一个点击事件会先从document一级一级的往下响应，此过程会经过目标元素的父元素、祖先元素，这个currentTarget就是事件流被某个dom响应时的那个标签元素，而target仅指的是目标元素。

原生的事件绑定是这样的：btn.addEventListener("click",function(event){}, false);
IE浏览器提供的是btn.attachEvent("click",function(event){}, false)
相应的移除事件绑定这是removeEventListener和detachEvent
true:表示捕获阶段调用事件处理程序；
false表示冒泡阶段调用事件处理程序。
大多数情况下我们将事件处理程序添加到事件流的冒泡阶段，这样可以最大限度的兼容各个浏览器，每个事件都有一个event对象，IE下则是window.event这个对象里面包含了很多关于这个事件的对象，例如target、type等等，这里我就不一一列举了，其中常用的是preventDefault()以及stopPropagation()两个函数，前者则是取消事件的默认行为，后者即是取消事件的进一步取消或者冒泡；
event对象包含的eventPhase的值：1,2,3分别代表事件处于捕获、目标和冒泡阶段的
相关元素。

UI事件：
load, unload,abord,error,select,resize,scroll，事件的具体触发条件我就不一一解释了，顾名思义，看事件名就可以知道，嘿嘿~
检测是否支持DOM3级事件：document.implementation.hasFeature('UIEvent', "3.0");

焦点事件：
blur、DOMfocusOut、focusout是事件目标是失去焦点的元素
focus、DOMfocusIn、focusin是事件获取到焦点的元素

鼠标与滚动事件：
DOM3事件级定义了9个鼠标事件：click，dbclick,mousedown，mouseenter、mouseleave、mousemove、mouseout、mouseover、mouseup
其中mouseout与mouseleave都是鼠标离开元素时触发，但是mouseout是将鼠标还在元素内只是移动到他的子元素都会触发，然而mouseleave只要鼠标还在元素范围内就不会再次触发，不管鼠标在该元素的哪个子元素上，其中mouseover与mouseout对应，mouseleave与mouseenter对应；
只有mouseover和mouseout事件，会获取相关元素，（非IE浏览器：eventTarget  IE：fromElement）所谓相关元素是鼠标从A区域移动到B区域，则A是B的相对元素，B是A的相对元素。

鼠标按钮：
event对象存在一个button的属性，用来判断用户是按下了左键还是右键还是中间建或者其中的组合。

鼠标滚动事件：
mousewheel，即当用户通过鼠标滚轮与页面交互的时候会触发，最终会冒泡到document或者window上，它包含的event对象包含鼠标事件的所有标准信息外，还包含一个特殊的wheelData属性，wheelDelta是120的倍数，向前滚动是120x，向后滚动是-120x

键盘事件：
1、keydown：用户按下键盘上的任意键时候出发，而且如果按住不放的话就会一直触发，会重复触发这个事件，
2、keypress：当用户按下键盘上的字符键时触发，
3、keyup：用户释放键盘上的键时触发
4、textInput:是keypress的补充，在文本显示给用户之前更容易拦截文本，在文本插入文本框之前会触发该事件，该event对象包含一个data属性，这个属性是值就是用户输入的字符，event对象的inputMethod属性会判断用户输入到文本框的方式，一共有0~7个值，其中常用的就是0和1；
0：表示浏览器不知道怎么输入的，
1：表示用户使用键盘输入的，2表示用户是粘贴进来的
keyCode的值可以用来表示键盘值，DOM3之后就不再包含charCode了，新增key和char，兼容：keyIdentifier

变动事件：DOM2级的变动事件能再DOM中的某一部分发生变化时触发。
包括：DOMSubtreeModified、DomNodeInserted、DomNodeRemove、DomNodeInsertIntoDocument等。
1、删除节点：removeChild，replaceChild，目标事件：event.target（被删除的节点）；event.relateNode(相关节点)会触发DOMNodeRemoved事件，会冒泡
2、插入节点：appendChild、repalceChild、insertBefore

HTML5事件：
1、contextmenu事件
冒泡，这个事件的目标是发生用户操作的元素。通常使用contextmenu事件来显示自定义的上下文菜单，而使用onclick事件处理程序来隐藏该菜单，
2、beforeunloaded事件
为了让开发人员有可能在页面卸载前阻止这一操作。
3、DOMContent事件
在window的load事件会在页面中的一切都加载完毕时候触发，但是这个事件在形成DOM树的时候就会触发，这个事件会冒泡到window但它的目标实际上是document，它不会提供任何额外的信息（其target属性是document）。
4、pageShow、pagehide(Firefox,Opera)
pageShow会在load事件触发后触发，当页面离开并且点击返回时候，返回到上一个页面的时候会触发，event事件的persisted为true的时候页面会保存在bfchache中，
pagehide事件，该事件会在浏览器卸载的时候触发，会在unload事件之前触发，和pageShow一样在document上触发，
5、haschange事件
当URL的参数列表发生变化的时候触发，（#后面的所有字符）

设备事件
1、orientationchange事件
Safari浏览器，用户是否横屏
window.orientation属性包含三个值，0表示肖像模式，90表示向左旋转的横向模式，-90表示向右旋转的横向模式。
2、MozOrientation事件
Fiefox为检测设备的方向引入一个名为MozOrientation的新事件。
3、deviceorientation事件，
它是在加速计检测到设备方向变化时在window上触发的，该事件是告诉开发人员设备在空间中朝向哪儿，而不是如何移动，设备的三维空间是靠X,Y,Z轴来定位的。
4、devicemotion事件
告诉开发人员设备怎么时候移动而不仅仅是设备方向如何改变。例如：是不是手机正在往下掉，或者是不是被人拿在行走的人的手里。
触摸与手势事件

触摸事件
touchstart：当手指触摸屏幕时触发，即使已经有一个手指放在屏幕上也会触发
touchmove：当手指在屏幕上滑动时连续的触发，在这个事件的发生期间，调用preventDefault（）可以组织滚动，
touchend：当手指从屏幕移开的时候会触发
touchcancel：当系统停止跟踪触摸时触发。
除了常见的DOM属性外，触摸事件还包含下列三个用于跟踪触摸的属性
touchs：表示当前跟踪的触摸操作Touch对象的数组
targetTouchs:特定于事件目标Touch对象的数组。
changeTouches:表示自上次触摸以来发生了什么改变的Touch对象的数组
每个Touch对象包含以下属性：
clientX:触摸目标在市口的X坐标
clientY:触摸目标在市口中的Y坐标
identifier:表示触摸的唯一ID
pageX:触摸目标在页面中的X坐标
pageY：触摸目标在页面中的y坐标
screenX:触摸目标在屏幕中的x坐标
screenY:触摸目标在屏幕中的y目标
target:触摸的DOM节点目标
手势事件
当两个手指头触摸屏幕时就会产生手势。有三个手势事件：
gesturestart：当一个手指头已经按在屏幕上，而另一个手指又触摸屏幕的时候触发。
gesturechange：当触摸屏幕的任何一个手指头的位置发生变化时触发
gestureend：当任何一个手指从屏幕上面移开时触发
只有两个手指头都触摸到事件的接受容器时才会触发这些事件。
比触摸事件多了两个属性：
rotation：手指变化引起的旋转角度。
scale：表示两个手指间距离的变化情况。这个值从1开始并且随着距离的拉大而增长，随着距离的缩小而缩小
内存和性能

事件委托
对事件处理程序过多问题的解决方案就是事件委托。事件委托利用了事件冒泡，只指定一个事件处理程序，就可以管理某一类型的所有事件。
