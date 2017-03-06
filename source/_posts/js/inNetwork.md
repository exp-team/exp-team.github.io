---
title: 由浅入深的前端面试题 和矫情的“浪漫主义”诗句
categories: js
tags: js
date: 2017-03-06
author: 侯策
---

好吧，我承认太标题党了，这篇文章是通过一道前端面试题引出的纯技术讨论。我先要矫情无比的从中外诗歌说起。

传统的佛学经典里，被世人熟知的有这样一句话：“一花一世界，一叶一菩提，一木一浮生，一草一天堂，一砂一极乐，一方一净土，一笑一尘缘，一念一清静。”。

昔时佛祖拈花，惟迦叶微笑，既而步往极乐。从一朵花中便能悟出整个世界，得升天堂，佛祖就是佛祖，谁人能有这样的境界。

同时，早在18世纪，英国伟大的浪漫主义诗人Black名为《天真的暗示》的诗中，也类似写道："To see a world in a grain of sand, and a heaven in a wild flower"，一颗沙里一个世界，一朵野花一座天堂。

转念，虽卑为码农，我们写出的代码，却彰显了功力：菜鸟和大神之间的差距，往往工程线上卑微的几行代码，便有天壤之差。

一道系列面试题，在JS知识体系中虽沧海一粟，但考察点充分评判面试者的能力。
管中窥豹，期待读者有不同想法与我讨论。

## 题目背景
题目是我在《effective javascript》一书中提取的。这一星期陆陆续续面试了不少于10个人，其中不乏工作履历突出的候选者。
但是很遗憾没有能完全在较短时间内有较高质量的回答。

## 题目前身
这道题可以分为前后两个部分，第一部分很简单，一般有一定JS OOP基础的候选者应该都可以答好：

一个社交网络有一组成员（member），每个成员有一个自己的名字，和存储其朋友信息的列表。请实现这样一个Member构造器。

正确答案不难理解：

    function Member (name) {
        this.name = name;
        this.friends = [];
    }

是不是非常简单。它的典型错误包括但是不限于：

    function Member (name) {
        this.name = name;
    }
    Member.prototype.friends = [];

关于方法和属性是应该放在原型上，还是构造函数中，如果您不明白的话，是时候补一补原型原型链的知识了。推荐给大家看一下我的同事颜海镜早在3年前的[一篇文章](http://yanhaijing.com/javascript/2014/05/15/a-code-explain-javascript-oop/)

同样，这道题上我会顺便考察一下面试者对JS中变量的存储方式，包括堆栈存储的不同情况和引用赋值的掌握情况。


## 题目变身
以上是对JS基础的考察，但是在这道题目的基础上，我进行了更深一步提问。希望对候选者的临场思维、JS基础甚至一些设计能力，又更进一步认识。

我要实现一个带环社交网络（社交圈）：

    var a = new Member('Alice');
    var b = new Member('Bob');
    var c = new Member('Carol');
    var d = new Member('Dieter');
    var e = new Member('Eli');
    var f = new Member('Fatima');

    a.friends.push(b);
    b.friends.push(c);
    c.friends.push(e);
    d.friends.push(b);
    e.friends.push(d, f);

这种情况下，需要实现一个inNetwork方法，判断某目标成员是否在另一个对象成员的社交圈中。规定：顺着社交链能找到目标成员，就认为在社交圈中。否则，不在其社交圈。


## 解题思路
如果刚接触到这样的题目，尤其是在面试现场，作为面试者很可能会慌乱一下。这时候，需要做的就是先准确分析题目。
根据题目，画出符合上述题目代码的实例化网络：

![实例社交圈图示](/bimg/network.png)

接下来思考，搜索意味着需要遍历整个社交网络。我们应该：

1）以单个根节点开始，

2）然后添加发现的节点，

3）移除访问过的节点，防止死环

最终实现：

    Member.prototype.inNetwork = function (target) {
        var visited = {};
        var worklist = [this]; // 用于存放社交链上的个体信息，初始时以“自己”作为根节点

        while (worklist.length > 0) {
            // 将worklist里的最后一项成员删除并取出
            var member = worklist.pop();
            // 如果存在环的情况，需要避免重复访问
            if (member.name in visited) {
                 continue;
            }
            visited[member.name] = member;
            if (member === target) {
                return true;
            }
            // 将当前成员的朋友列表加入worklist当中，他们都在根节点的社交链上
            member.friends.forEach(function(friend) {
                worklist.push(friend);
            })
        }
        return false;
    }

我在代码中加上了注释，如果您还不明白也没有关系。建议去跑一下程序，进行debugger和console，尝试去理解。

测试：

    a.inNetwork(f) // true
    f.inNetwork(a) //false

哈哈，果然Alice能通过朋友圈查找到Fatima，而Fatima却不能反向找到Alice!当然，这样我认为是违反人类社会常识的。但是，谁让他是题目呢？


一道简单的题却覆盖了很多知识点，比如：while循环中的流程控制（continue），数组的基本方法（pop,forEach,push），for...in等等。

它的典型错误包括但是不限于：使用对象承载worklist，然后用for...in循环遍历worklist。

这样做的问题在于：for...in循环并没有要求枚举对象的修改与当前循环保持一致。事实上，标准规范规定了：

“如果被枚举对象在枚举期间添加了新的属性，那么枚举期间并不能保证新添加的属性能够访问”。


## 总结
这道题考察了面试者包括JS OOP在内的多项基础，尤其是后半道考察了候选者的思维能力、反应能力。

扯回原诗句，谈一下感悟，在天体的转动和岁月的轮回中，我们分明的感受到每一个个体所拥有生命周期的单薄无力，在宏大的宇宙观中恐怕渺小不及沧海一粟。诗句的后半句拿出来共勉：“Hold infinity in the palm of your hand, and eternity in an hour 把无限放在你的手上，永恒在一刹那里收藏”。

在前端快速迭代发展的学习中，作为初学者，往往面对浩瀚的知识海洋望洋兴叹，此时基础便是那能够收藏“永恒和无限”的潘多拉魔盒。







