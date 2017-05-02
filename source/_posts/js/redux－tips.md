---
title: 拒绝Redux文档“毒害” 一个项目告诉你Redux最新真正哲学
categories: js
tags: js
date: 2017-04-04
author: 侯策
---

如果你正在React应用中使用Redux，那么你十有八九使用了react-redux库来连接你的组件和state，并且使用了connect方法。这个方法其实是一个非常微妙或者狡猾（tricky）的API，尽管他的使用方式非常的简单，非常魔幻。

大部分人通过Redux文档来学习这个超级流行的数据流管理框架。可是，笔者发现，在中文的自述文档中，已经远远的与此框架脱节了。里面的内容和倡导的哲学理念，已经有失“先进”，甚至某种程度上有所误导。

本文将借助Ryan Johnson的[文章:Quick Redux tips for connecting your React components](https://medium.com/dailyjs/quick-redux-tips-for-connecting-your-react-components-e08da72f5b3)，来告诉你Redux的秘密和演化。同时，**穿插一个很简单的场景，来说明旧时思想的保守性，以及最新的设计理念。**

同时，本文提供了**一个结合“最先进”Redux理念的todo list demo，[地址请点击这里](https://github.com/HOUCe/react-redux-multiConnect-renderCountProp)**，欢迎围观并下载。

如果你对这套技术栈有兴趣的话，欢迎参看我的其他类似文章：

- [解析Twitter前端架构 学习复杂场景数据设计](http://www.jianshu.com/p/7a56ac1de2a8)
- [React Conf 2017 干货总结1: React + ES next = ♥](http://www.jianshu.com/p/83c86dd0802d)
- [React+Redux打造“NEWS EARLY”单页应用 一个项目理解最前沿技术栈真谛](http://www.jianshu.com/p/cde3cf7e2760)
- [一个react+redux工程实例](http://www.jianshu.com/p/8e28be0e7ab1)
- ......



## Redux原作者的转变
> “周虽旧邦，其命惟新”（出自《诗经·大雅》）

大意是“周朝虽然是旧的国家，但却禀受了新的使命”。同样，Redux框架的发展历程，虽然历史并不太长久，也没有“南朝四百八十寺,多少楼台烟雨中”的昙花一现状态。但是其设计理念，也禀受了新的使命，得而更新。

在最初版的Redux设计哲学中，Dan Abramov强调**“在顶层保持一个容器组件”**，其实这个是值得推敲的。正确的思想是不要把这个当做准则。

>让你的展现层组件保持独立。然后创建容器组件并在合适时进行连接。当你感觉到你是在父组件里通过复制代码为某些子组件提供数据时，就是时候抽取出一个容器了。只要你认为父组件过多了解子组件的数据或者action，就可以抽取容器。
总之，试着在数据流和组件职责间找到平衡。

后来，Dan Abramov也发表了推文，对这一思想原则进行了“推翻”：


![推文内容](http://upload-images.jianshu.io/upload_images/4363003-29a267048bda0dbe.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## 场景举例
React-Redux提供connect方法，用于从UI组件生成容器组件。connect的意思，就是将这两种组件连起来。

最初的Redux理念强调在最顶层connect我们的组件。
这究竟意味着什么呢？

我们看下边的例子，<Page>组件出现在组件树的最顶层，用于进行与外部的通信，将数据传给子孙组件，由UI组件渲染出视图。


![设计结构](http://upload-images.jianshu.io/upload_images/4363003-ed48e3446ba9e7cf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**那么，这样做的弊端在哪里呢？**

- 当你更改profile image时，整个<Page>组件将会重新渲染；
- 当你删除feed list中一项时，整个<Page>组件将会重新渲染；
- 当你增加一个image时，整个<Page>组件将会重新渲染；

那么，我们是否可以把 <Profile>, <Feed>, 和 <Images>都抽出成容器组件，去关联他们所关心的state呢？
当然可以，就像下面这样：


![设计截图](http://upload-images.jianshu.io/upload_images/4363003-ff65b8c97351f00c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



**这样做，有哪些优势呢？**
- 当你更改profile image时，只有<Profile>组件将会重新渲染；
- 当你删除feed list中一项时，只有<Feed>组件将会重新渲染；
- 当你增加一个image时，只有<Images>组件将会重新渲染；


时刻要记得，当使用connect进行组件和state连接时，你告诉了React：当前组件任何一处props改变时，都要进行渲染。
如果我们把容器组件进行拆分，当然就避免了那些不必要的渲染。


## 彩蛋环节

如果你在使用Redux，我推荐一个[reselect类库](https://github.com/reactjs/reselect)。它提供了带cache功能的selector。如果  Store/State和构造view的参数没有变化，那么每次Component获取的数据都将来自于上次调用/计算的结果。

另外，仔细研读Redux最新文档，你会发现connect方法除了mapStateToProps和mapDispatchToProps之外，还存在很多选项。比如renderCountProp，它可以帮助我们记录组件渲染次数。

最简单的todo list，你可以：

    const VisibleTodoList = connect(mapStateToProps, mapDispatchToProps, null, { renderCountProp: 'renderCount' })(TodoList)



## 总结和实战

之前说了这么多理论内容。笔者觉得有必要写一个简易demo，来实践一下本文主题。原谅我时间有限，选取了已经有骨架的todo list，融入了多connect，多容器组件思想，同时也使用了renderCountProp：


![组件](http://upload-images.jianshu.io/upload_images/4363003-52bbcef9207bf074.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



如上图，我们有两个connect化的组件，分别为<AddTodo>和<TodoList>;


![renderCountProp](http://upload-images.jianshu.io/upload_images/4363003-c85dec06f426e764.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


另外，我采用了react-redux: 5.0.1版本，这个版本下，经过代码处理，我们可以看到在组件中多了一个renderCountProp，这对于我们的性能调试非常重要。

所有的代码实现，我已经托管在了[Github仓库当中](https://github.com/HOUCe/react-redux-multiConnect-renderCountProp)，欢迎围观评论！


Happy Coding!

PS: 作者[Github仓库](https://github.com/HOUCe)，欢迎通过代码各种形式交流。