---
title: 解析Twitter前端架构 学习复杂场景数据设计
categories: js
tags: js
date: 2017-04-20
author: 侯策
---

前几天刷Twitter，发现Nicolas（Engineering at @twitter. Technical Lead for Twitter Lite）发布了这么一条推文：

图片

大家可以忽略我未发的留言lol～

大体意思就是Twitter前端经过重构，已经完全迁移到React＋Redux技术栈了。
听到这个消息之后，我觉得去深挖一下Twitter的Redux store组织架构，将会非常有意思。
这对于在复杂场景下的前端数据学习，以及React、Redux数据流设计十分有意义。

本文将剖析Twitter前端数据结构层次，如果你对React技术栈不是很了解，也不妨碍阅读；同样，如果你对这套技术栈有兴趣的话，欢迎参看我的其他类似文章：

本文主体内容翻译自Ryan Johnson的文章：[Dissecting Twitter’s Redux Store](https://medium.com/statuscode/dissecting-twitters-redux-store-d7280b62c6b1)，笔者进行了一定程度的拓展。



## 准备工作
想要看Redux store的前提是你需要配有React Developer Tools (RDT)，在RDT tab中选中应用根节点。
确保选中之后，在console面板中输入：
    
    // $r is a shortcut that references the selected element in RDT
    $r.store.getState();

接下来，我们就可以看到Redux数据树，就像图中所示：

图片


## 设计分析
我建议大家花些时间对每个不同的state进行展开，并加以学习。这篇文章中，我挑选entities/tweets和homeTimeline两个最主要也是最核心的state进行剖析。这两个states包含了一条tweet的所有关联数据。

tweet，就像下图中我所发的：

图片

一条tweet内容的数据信息全部存储在entities/tweets/entities中，entities/tweets/entities可以理解为一个normalized的data table；
在这个table中，每一条tweet都是一个键值对类型的js object：key为该条tweet的id，value为该条tweet的数据。

下图中，我将第一条tweet展开，方便大家一探究竟：

图片

了解了tweet存储结构，我们接下来看一下Twitter首页的timeline结构。
每个用户的首页timeline信息可以在homeTimelines/timeline找到。首页timeline展示的顺序，按照timeline这个数组的顺序。也就是说，timeline数组index为0的条目，就是你在首页timeline上看到的第一条tweet；

首页timeline上的每条tweet，都有一个唯一的id，这个id和上面介绍的，存储在entities/tweets/entities之中的tweet id相匹配。

看到这里，你也许会感叹：“This is pretty much normalizing state shape 101 from Dan Abramov！”
没错，这样的范式也是Redux所推崇的，完全的扁平化设计带来的开发体验是无与伦比的。

当然，你可能会问为什么Redux包括Twitter都在推崇扁平化的数据结构呢？
这个问题建议参考：[Redux core concepts](http://redux.js.org/docs/recipes/reducers/NormalizingStateShape.html)，如果您英语阅读吃力，可以留言与我交流。


继续言归正传，首页timeline加载新tweets方式有两种：
1）上拉加载track tweets by top
2）下拉加载track tweets by bottom

第一种用于拉取更新的tweets，第二种用于拉取更旧的tweets；比如你新发了一条tweet，就要上拉，方可显示在timeline上；如果没有最新的，向下拉到底部后，自动加载更早的tweets。

这种情况下，homeTimelines下的lastFetch.bottom和lastFetch.top，分别为时间戳，记录最后一次更新数据的信息。
cursor.bottom和cursor.top值分别为一个tweet id，表示当前timeline上，最上边和最底部分别是哪一条tweet。

最后一个非常有意思的是，entities下除了存在entities/tweets之外，还分别有cards, lists and users；当你打开这其余三项的时候，会发现这三项与entities/tweets保持在相同的结构，他们都一个fetchStatus的data table，key为tweet id, value为加载状态，据统计一共有一下几种：
‘none’;
‘loading’; 
‘loaded’;
‘failed’.

这几种状态的设置无外乎这么几个目的：
保证在loading状态或loaded的tweet不会再发送请求给server；
在未加载完时，可以显示加载动画或者展位图；
在加载失败时，可以显示失败提示或者在此请求时进行补救。


## 总结
本文分析了Twitter在采用Redux架构下的数据设计结构，在一个复杂的场景下，希望引起读者对redux能有一个更深入的认识。

Happy coding!



PS: 作者[Github仓库](https://github.com/HOUCe)，欢迎通过代码各种形式交流。



