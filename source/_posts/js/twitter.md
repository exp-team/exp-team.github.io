---
title: 解析Twitter前端架构 学习复杂场景数据设计
categories: js
tags: js
date: 2017-04-20
author: 侯策
---

前几天刷Twitter，发现Nicolas（Engineering at @twitter. Technical Lead for Twitter Lite）发布了这么一条推文：


![twitter.jpeg](http://upload-images.jianshu.io/upload_images/4363003-8aaeb69cd47cb2bf.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




大体意思就是Twitter前端经过重构，已经**完全迁移到React＋Redux＋PWA技术栈**了，后端也使用了nodeJS，实现了“前端一统天下”，lol。

听到这个消息之后，我觉得去深挖一下Twitter的Redux store组织架构，将会非常有意思。
这对于在复杂场景下的前端数据学习，以及React、Redux数据流设计十分有意义。

因为，在Redux数据流框架的思想下，对于数据的处理和分配完全由前端掌握。
**前端数据如何设计，设计的功力如何直接完全决定整个项目的开发进度以及代码强健性，甚至还决定着页面的性能。**

本文将剖析Twitter前端数据结构层次，如果你对React技术栈不是很了解，也不妨碍阅读；同样，如果你对这套技术栈有兴趣的话，欢迎参看我的其他类似文章：

- [React Conf 2017 干货总结1: React + ES next = ♥](http://www.jianshu.com/p/83c86dd0802d)
- [React+Redux打造“NEWS EARLY”单页应用 一个项目理解最前沿技术栈真谛](http://www.jianshu.com/p/cde3cf7e2760)
- [一个react+redux工程实例](http://www.jianshu.com/p/8e28be0e7ab1)
- ......

欢迎关注[我的主页](http://www.jianshu.com/u/452568260db5)，更多技术文章不再错过。

本文主体内容翻译自Ryan Johnson的文章：[Dissecting Twitter’s Redux Store](https://medium.com/statuscode/dissecting-twitters-redux-store-d7280b62c6b1)，笔者进行了一定程度的拓展。



## 准备工作
想要看Redux store的前提是你需要配有React Developer Tools (RDT)，在RDT tab中选中应用根节点。
确保选中之后，在console面板中输入：
    
    // $r is a shortcut that references the selected element in RDT
    $r.store.getState();

接下来，我们就可以看到Redux数据树，就像图中所示：


![数据结构](http://upload-images.jianshu.io/upload_images/4363003-8d48af5175e3ed3a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## 设计分析
我建议大家花些时间对每个不同的state进行展开，并加以学习。但在这篇文章中，由于篇幅所限，我会挑选并深挖：

- entities/tweets和
- homeTimeline

两个最主要也是最核心的state进行剖析。这两个states包含了一条tweet的所有关联数据。

一条tweet，就像下图中我所发的：


![推文举例](http://upload-images.jianshu.io/upload_images/4363003-f62005116139e5a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


一条tweet内容的数据信息全部存储在**entities/tweets/entities**中，entities/tweets/entities可以理解为一个normalized的data table，它存储了所有tweets推文的信息；
在这个table中，每一条tweet都是一个键值对类型的js object：key为该条tweet的id，value为该条tweet的数据，也是一个js object。

下图中，我将第一条tweet展开，方便大家一探究竟：


![推文设计.png](http://upload-images.jianshu.io/upload_images/4363003-ce6f3f6d0ff409ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




了解了tweet存储结构，我们接下来看一下Twitter首页的**timeline**结构。
直观上，timeline一定包含了个人主页展示推文的信息。通过tweet id和刚才介绍过的**entities/tweets/entities**中的tweet相匹配,并最终加以在timeline上展示。
如下图：


![timeline数据结构](http://upload-images.jianshu.io/upload_images/4363003-8e57db708c65bf2f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


每个用户的首页timeline信息可**homeTimelines/timeline**找到。首页timeline展示的顺序，则按照timeline这个数组的顺序。也就是说，timeline数组index为0的条目，就是你在首页timeline上看到的第一条tweet；

**重要的话再说一遍：**
首页timeline上的每条tweet，都有一个唯一的id，这个id和上面介绍的，存储在entities/tweets/entities之中的tweet id相匹配。

看到这里，你也许会感叹：

>“This is pretty much normalizing state shape 101 from Dan Abramov！”

**没错，这样的范式也是Redux所推崇的，完全的扁平化设计带来的开发体验和性能提升是无与伦比的。**

当然，你可能会问为什么Redux设计哲学，包括Twitter都在推崇扁平化的数据结构呢？
这个问题建议参考：[Redux core concepts](http://redux.js.org/docs/recipes/reducers/NormalizingStateShape.html)，这里讲的非常清晰，被收录在Redux core concepts中，强烈建议阅读。
如果您英语吃力，可以留言与我交流，就不再展开了。


继续言归正传，我们来讨论一下**滚动时的异步请求设计。**
首页timeline加载新tweets方式有两种：
- **上拉加载** track tweets by top
- **下滑加载** track tweets by bottom

第一种用于拉取更新的tweets，第二种用于拉取更旧的tweets；比如你新发了一条tweet，就要上拉，方可显示在timeline上；如果没有最新的，向下滑动到底部后，自动加载时间上更早的tweets。
用一个等式来表达：
> **top = new tweets,**
> and 
> **bottom = older tweets**

这种情况下，homeTimelines下的lastFetch.bottom和lastFetch.top，分别为时间戳，记录最后一次更新数据的信息(上拉和下滑)。

> - **lastFetch.bottom**: 记录最后一次向下滑动而更新数据的信息；
> - **lastFetch.top**: 记录最后一次下上拉取而更新数据的信息；

同时，
cursor.bottom和cursor.top值分别为一个tweet id，表示当前timeline上，最上边和最底部分别是哪一条tweet。

> - **cursor.bottom**: 记录屏幕最底部tweet ID；
> - **cursor.top**: 记录屏幕最顶部tweet ID；

同时， homeTimelines里面还记录了isLoadingDirections.bottom和isLoadingDirections.top来表示数据加载的触发源头。

如图：

![记录信息.png](http://upload-images.jianshu.io/upload_images/4363003-ae4bfcdef0e28794.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**最后一个非常有意思的是**，entities下除了存在entities/tweets之外，还分别有cards, lists and users；

- entities/tweets
- entities/cards
- entities/lists
- entities/users

**来表示不同的推文特性。**

当你打开这其余三项的时候，会发现这三项与entities/tweets保持在相同的结构，他们都有一个fetchStatus的data table，key为tweet id, value为加载状态，据统计一共有一下几种：
- ‘none’;
- ‘loading’; 
- ‘loaded’;
- ‘failed’.



![状态截图](http://upload-images.jianshu.io/upload_images/4363003-92a4747d756d8702.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**这几种状态的设置无外乎这么几个目的：**
- 保证在loading状态或loaded的tweet不会再发送请求给server；
- 在未加载完时，可以显示加载动画或者展位图；
- 在加载失败时，可以显示失败提示或者在此请求时进行补救。


## 总结
本文分析了Twitter在采用Redux架构下的数据设计结构，在一个复杂的场景下，希望引起读者对redux能有一个更深入的认识。


本文主体内容翻译自Ryan Johnson的文章：[Dissecting Twitter’s Redux Store](https://medium.com/statuscode/dissecting-twitters-redux-store-d7280b62c6b1)，笔者进行了一定程度的拓展。

Happy coding!



PS: 作者[Github仓库](https://github.com/HOUCe)，欢迎通过代码各种形式交流。