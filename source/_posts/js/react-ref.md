---
title: 通过React Ref细小知识点 谈谈前端工程师的进阶
categories: js
tags: js
date: 2017-06-04
author: 侯策
---

熟悉React的同学可能对其中的Ref并不陌生。不知道您是否听说"最新的React版本(v15.5.4)已经对这个API进行了修改并更新"。如果您一直保持对React框架的跟进与学习，相信会很清楚，**新版本中"ref改为使用回调函数的方式去引用"。**

这是一个非常细小但是微妙的改动。当我得知时，第一个念头就是** "这样的改动意义在哪里？"**

带着这个疑问，下班后我花了一晚上近2个小时时间去探究这一个微小改动，收获颇丰的同时，可谓"眼界大开"。

这篇文章我会记录一步一步探究的过程，以及总结分析最终答案。同时，对react框架及其周边技术栈感兴趣的同学可以参阅斧正我其他文章：

- [解析Twitter前端架构 学习复杂场景数据设计](http://www.jianshu.com/p/7a56ac1de2a8)
- [React Conf 2017 干货总结1: React + ES next = ♥](http://www.jianshu.com/p/83c86dd0802d)
- [React+Redux打造“NEWS EARLY”单页应用 一个项目理解最前沿技术栈真谛](http://www.jianshu.com/p/cde3cf7e2760)
- [一个react+redux工程实例](http://www.jianshu.com/p/8e28be0e7ab1)
- ......

## 古来青史谁不见,今见功名胜古人

在对Ref使用方式的改动分析前，我们先要对其有一个全面的了解和认识。当然，最直接最有效的方式就是去[官网](https://facebook.github.io/react/docs/refs-and-the-dom.html)学习。

因为React本身更迭迅速，关于Ref的文档信息也经历过多次修正。官网上保持着最新版的输出，正所谓"古来青史谁不见,今见功名胜古人"。

这里先总结一下基本概念。

*** Refs诞生背景

我们都知道React讲究的是**单项数据流**，也就是说父组件与其子组件之间的通信依靠props来实现。如果你想通过父组件去更改一个子组件，那么就要从父组件处传递一个新的props值，已达到让子组件re-render的目的。

但是在一些情况下，这样严格略显教条的数据流或通信方式并不能满足我们需求。这时候，就是Ref派上用场的时候了。

那么我们在具体哪些场景下会用到Ref呢？

- 处理输入框聚焦，文本选择，或其他多媒体反馈信息时；
- 触发需要的动画时；
- 与第三方操作DOM的类库交互时。

**其他情况下，为了避免破坏React的哲学思想，都是不被建议使用的。** 

比如，一个弹框组件，我们不应该暴露类似 open(), close()这样的方法，以通过ref控制来调用。更合理的做法是通过一个类似 isOpen的prop去在组件之间传递协调。

### Refs使用方法

正如先前所说，ref属性目前接收一个回掉函数。**当相应组件mounten或者unmounted时，该回掉函数会被立即调用**

比如，我们想让一个输入框组件在渲染后，自动聚焦，可以：


上述内容是在组件内引用Ref, 同样，类似父子组件通过props通信，我们可以完成**父组件通过Ref控制子组件行为**:


更多的用法和注意事项不再过多介绍，这并非此文主题，读者可以通过官网学到所有的内容。


## 问渠哪得清如许？为有源头活水来。

老版本关于Ref的用法远没有这么复杂。Ref完全可以通过字符串来定义。同样的功能，我们可以这样实现：



回到我们的探索之路，初期我是很难理解这样的变动有什么好处？并且更让我困惑的是，官方的一段说法：

> If you worked with React before, you might be familiar with an older API where the ref attribute is a string. **We advise against it because string refs are considered legacy, and are likely to be removed in one of the future releases**. Although string refs are not deprecated, they are considered legacy, and will likely be deprecated at some point in the future. Callback refs are preferred.

官方认为之前的"字符串模式"是一种"反模式" ,在未来版本中很有可能会被废除。强烈向大家推荐了回掉函数式的用法。

带着疑惑，我Google了此问题（原谅我搜商较低）：

![搜索内容](http://upload-images.jianshu.io/upload_images/4363003-00ebd947dadb6beb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

不要吐槽我"连句英文都说不清楚"，因为我认为这样的关键字搜索方式更有利于找到有价值的内容。（您真当您跟搜索引擎对话呢？）

不出所料，搜索结果很多且杂乱。
第一条结果属于React官网，下面便有Stack Overflow的相关信息。
第一条：


![类似问题](http://upload-images.jianshu.io/upload_images/4363003-b7d617b5b0016416.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

此提问者还加了一句：

> NB: I'm looking for the "official" answer to the statement in the documentation, I'm not asking about personal preferences and so on.

最高分的答案贡献了一下内容：


![问题答案](http://upload-images.jianshu.io/upload_images/4363003-b31dafe4c9fe8d8e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

[此答案](https://stackoverflow.com/questions/30710076/react-saving-a-component-in-the-ref-callback)列举了7条老版本用法的问题。其实我在工作当中，并不是用React技术栈，所有知识储备以理论为主。所以，其中的很多条我并不能完全赞同。

并且答案也没有具体的代码示例，虽然没一条目英语层面上一清二白，但是具体在说什么某些条目上并没有完全理解。

先存起来，后边慢慢消化尝试理解，继续寻找答案。 **变换策略，我直接去React Github仓库寻找答案。**果然，里面非常多详尽的干货。

我在[第8333号Pull Requests中](https://github.com/facebook/react/pull/8333)，找到了Redux原作者，现在已经加入Facebook React团队的Dan Abramov的"官方回答"：


![Dan Abramov](http://upload-images.jianshu.io/upload_images/4363003-292bb281019f1405.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

解答内容为 （意译非直译）：

- 老版本基于字符串的Ref会追踪其组件，因为不利于React的运行速度和性能保证；
- Ref在深层次的组件中，难以满足使用者对其的调用需求；
- 老版本基于字符串的Ref并不像新版本采用回掉方式带来的可组合(Composable)益处。

虽然是男神解惑，可是反应迟钝智商拙急的我还是很难彻底理解。

难道就这样睡去吗？下班回家后已经持续研究了一个小时了，女朋友貌似在下一秒就要爆发。。。

**我当然是不甘心的，追求答案在此刻优先级已经溢出！**

紧接着，我翻到了Facebook React团队成员Sebastian Markbåge早在2014年的[对Ref的 "吐槽"](https://github.com/facebook/react/issues/1373)，此issues一共多达30多个comments, 延续到了2015年。并最终在2014年底，对Ref的改进写入了react-future，具体信息 [点击这里可以追溯](https://github.com/reactjs/react-future/blob/master/01%20-%20Core/06%20-%20Refs.js)，当时的代码注释信息写的非常清晰：

> This is a refs proposal that uses callbacks instead of strings. The callback can be easily statically typed and solves most use cases.
When a ref is attached, it gets passed the instance as the first argument.
When a ref is detached, the callback is invoked with null as the argument.

接着，我找到了2015年9月react源码提交的[commit信息](https://github.com/facebook/react/commit/5ee8a93280987bf1547687f5d8665be89058f321#all_commit_comments)，最关键的内容提炼出来：

> Callback refs are preferred. We plan to deprecate string refs at some point in the future, but just haven't gotten around to it yet.
String refs are very "magical" and is not idiomatic javascript, which is bad on principal. But there are also some practical reasons...
String refs could never be implemented in user space (callbacks can; just have the component call the callback). The ramifications of this are far-reaching. For instance, if you wanted to create a HOC (Higher Order Component) that transparently wraps another component, you could forward all the props... but you couldn't forward a string ref. Or if you wanted a parent and a grandparent to both have a ref to a component, callback refs allow you to wrap the callback and pass it down, but string refs do not. The list goes on, but in the interest of time, I'll truncate it here.

建议读者先自己尝试去理解，下文中我会进行汉语总结。
同时我也看到了facebook react团队另外一名成员Jim的话：

>Basically, our team has had countless discussions on this topic and arrived at the consensus that string refs should be phased out in favor of callback refs.

可见，关于Ref这个API设计调整的问题，源码团队也是经历了无数次讨论与争执。此时，我更加迫不及待地去挖掘体会更多信息。


## 卷地风来忽吹散，望湖楼下水如天

说来有趣，最终让我把众多信息"融会贯通"的契机来自于[**Dan Abramov的一条Twitter：**‏ ](https://twitter.com/dan_abramov)


![Dan's Twitter](http://upload-images.jianshu.io/upload_images/4363003-df4edea15393570c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过这条Twitter以及该Twitter下的评论和回复，尤其是附带的代码截图，我瞬间理解了之前的很多信息。

先发给大家体会：


![代码1](http://upload-images.jianshu.io/upload_images/4363003-db606be5928801bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![代码2](http://upload-images.jianshu.io/upload_images/4363003-ae8681234a038bf7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![代码3](http://upload-images.jianshu.io/upload_images/4363003-9c40dbff15d1122a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

仔细揣摩，你能体会到什么呢？

其实Dan Abramov私人还维护着一个React组件库：React DnD，这个组件库其实就是一个 [HOC（Higher-Order Components）](https://facebook.github.io/react/docs/higher-order-components.html)，浅显来说本质上，DnD对外暴露一个高阶组件，这个组件接收用户定义的组件，并通过"一系列魔法"，又返回用户传入的自定义的组件，此时这个组件可拖拽。

用最简单的代码展示，类似：

我们看代码1，dragSourceRef函数作为自定义（即需要实现拖拽的组件）组件的ref回掉函数，在React DnD组件库中，变获取了这个组件instance；

如果还是字符串方式，在React DnD组件库中只能预先定义协商好的ref值，以方便该组件库进行"魔法处理"。
但是如果需要进行可拖拽化的组件很多时，ref值就面临冲突的问题。
当然，我们可以通过一个预留的数组，在组件库和业务代码里传递refs值。但是这样的实现显然是很丑陋的。

参考 [facebook react 8734号issue](https://github.com/facebook/react/issues/8734)：


![facebook react 8734号issue](http://upload-images.jianshu.io/upload_images/4363003-597c140d4d6d5613.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这显然，已经是新Ref API变革带来的巨大好处。

同时仔细揣摩上面代码2，3截图，便是对composable的有力说明。当然，通过老版本字符串方式可以实现同样的功能(有内存管理的瑕疵，具体见下文说明)，那么composable的优势，我认为可能只会体现在"先进的思想层面和灵活性层面"。


## 纸上得来终觉浅 绝知此事要躬行
除了上述React 仓库commit、issue、pull requests之外，我又参考了更多信息。

关于官方团队给出的优势，我认为很多条目是在向Functional Programming靠拢，更多意义上是一种思维模式的转变。

比如说，我想实现"在三层嵌套（Grandparent->Parent->child）结构中，Grandparent组件操控child组件"，这样的场景需求新老两种版本都可以实现：
回掉方式：

老版本字符串方式：

但是仔细对比这两种方式，在老版本中的实现，先不说我们使用了

      this.refs.myInput2.refs.myInput1.refs.myInput

这么一长串才能拿到instance,更应该注意到为了拿到第三层的instance，我们被迫新增了两个只用于中间传递的refs。

这还没有完，前文提到过内存管理。我们再来思考一下，当child unmount时，基于字符串的ref方式无法感知。注意这都是在内存层面，即反映在虚拟DOM的开销上。他并不是真正的DOM节点，自然不会有依赖于浏览器的垃圾回收机制。

当程序中有大量类似场景时，大量unmount组件出现，就会有大量的无意义内存占用。

如果使用新的Ref API方式，我们可以在track的组件unmount时得到通知。因为：

>The ref attribute takes a callback function, and **the callback will be executed immediately after the component is mounted or unmounted.**

注意**the callback will be executed immediately after the component is mounted or unmounted.**我们只需要在child组件中进行监测，unmount后将this.input = null，即可完成手动垃圾回收：











