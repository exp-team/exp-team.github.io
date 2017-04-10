---
title: 关于数组乱序的深挖——“感觉一直在写毒代码”
categories: js
tags: js
date: 2017-04-20
author: 侯策
---

最近看了一篇非常有趣的文章：[关于JavaScript的数组随机排序](http://mp.weixin.qq.com/s/0j7iMJwaXYt3BD036M8s-w)，其作者为oldj前辈。
文中指出了我们用来“将一个数组随机排序”的经典写法存在的问题，获益匪浅。

本文将以更加详尽的材料和更多样的code demo进行阐述。并尝试用“Fisher–Yates shuffle”洗牌算法进行终极解答。

## 多个熟悉的场景
将一个数组进行乱序处理，是一个非常简单但是非常常用的需求。比如，“猜你喜欢”、“点击换一批”、“中奖方案”等等都可能应用到这样的处理。包括我自己在写代码的时候，也确实遇到过。当时的处理很简单也很流行：对对象数组采用array.sort()方法，并传入一个比较函数（comparison function），这个比较函数随机返回一个介于［－0.5， 0.5］之间的数值：

    var numbers = [12,4,16,3];
    numbers.sort(function() {
        return .5 - Math.random();
    });

关于这个做的理论基础这里不再进行阐释。如果您不明白，可以了解一下JS中sort函数的使用方法。


## 有毒的array.sort方法
正像oldj前辈文章指出的那样，其实使用这个方法乱序一个数组是有问题的。

为此，我写了一个脚本进行验证。强烈建议读者去[Github]()围观一下，可以clone下来自己试验。

脚本中，我对

    var letters = ['A','B','C','D','E','F','G','H','I','J'];

letters这样一个数组使用上述方法进行了10000次排序，并把排序结果存储在countings当中。结果在页面上进行输出：

    var countings = [
        {A:0,B:0,C:0,D:0,E:0,F:0,G:0,H:0,I:0,J:0},
        {A:0,B:0,C:0,D:0,E:0,F:0,G:0,H:0,I:0,J:0},
        {A:0,B:0,C:0,D:0,E:0,F:0,G:0,H:0,I:0,J:0},
        {A:0,B:0,C:0,D:0,E:0,F:0,G:0,H:0,I:0,J:0},
        {A:0,B:0,C:0,D:0,E:0,F:0,G:0,H:0,I:0,J:0},
        {A:0,B:0,C:0,D:0,E:0,F:0,G:0,H:0,I:0,J:0},
        {A:0,B:0,C:0,D:0,E:0,F:0,G:0,H:0,I:0,J:0},
        {A:0,B:0,C:0,D:0,E:0,F:0,G:0,H:0,I:0,J:0},
        {A:0,B:0,C:0,D:0,E:0,F:0,G:0,H:0,I:0,J:0},
        {A:0,B:0,C:0,D:0,E:0,F:0,G:0,H:0,I:0,J:0}
    ];
    var letters=['A','B','C','D','E','F','G','H','I','J'];
    for (var i = 0; i < 10000; i++) {
        var r = ['A','B','C','D','E','F','G','H','I','J'].sort(function() {
            return .5 - Math.random();
        });
        for(var j = 0; j <= 9; j++) {
            countings[j][r[j]]++;
        }
    }
    for(var i = 0; i <= 9;i++) {
        for(var j = 0;j <= 9;j++) {
            document.getElementById('results').rows[i + 1].cells[j + 1].firstChild.data = countings[i][letters[j]];
        }
    }

做中得到结果如图：

    图片

如果点击“recalculate”按钮，可以进行多次10000次取样试验。

不管点击按钮几次，你都会发现整体乱序之后的结果绝对不是“完全随机”。由此可以得出结论，使用array.sort方法进行乱序处理，绝对是有问题的。


## array.sort方法底层究竟如何实现？
但是为什么会有问题呢？这需要从array.sort方法排序底层说起。
在[Chrome v8引擎源码中](https://github.com/v8/v8/blob/master/src/js/array.js)，可以清晰看到，

>v8在处理sort方法时，使用了插入排序和快排两种方案。当目标数组长度小于10时，使用插入排序；反之，使用快排。
Chrome’s v8 uses a combination of InsertionSort and QuickSort. That is, if the array is less than 10 elements in length, it uses an InsertionSort.


其实不管用什么排序方法，大多数排序算法的时间复杂度介于O(n)到O(n2)之间，元素之间的比较次数通常情况下要远小于n(n-1)/2，也就意味着有一些元素之间根本就没机会相比较（也就没有了随机交换的可能），这些 sort 随机排序的算法自然也不能真正随机。

怎么理解上边这句话呢？其实我们想使用array.sort进行乱序，理想的方案或者说纯乱序的方案是数组中每两个元素都要进行比较，这个比较有50%的交换位置概率。这样一来，比较次数为n(n-1)。而在sort排序算法中，大多数情况都不会满足这样的条件。因而当然不是完全随机的结果了。

顺便说一下，关于v8引擎的排序方案，源码使用JS实现的，利于前端程序员阅读。其中，使用了大量的性能优化技巧，尤其是关于快排的pivot选择上十分有意思。感兴趣的读者不妨研究一下。


## 真正意义上的乱序
要想实现真正意义上的乱序，其实不难。我们首先要规避不稳定的array.sort方法。在计算机科学中，有一个专门的：[洗牌算法Fisher–Yates shuffle。](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle)如果你对算法天生迟钝，也不要慌张。这里我一步一步来实现，相信您一定要得懂。

先来整体看一下所有代码实现，一共也就10行：

    Array.prototype.shuffle = function() {
        var input = this;
        for (var i = input.length-1; i >=0; i--) {
            var randomIndex = Math.floor(Math.random()*(i+1)); 
            var itemAtIndex = input[randomIndex]; 
            input[randomIndex] = input[i]; 
            input[i] = itemAtIndex;
        }
        return input;
    }
    var tempArray = [ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 ]
    tempArray.shuffle();
    console.log(tempArray);  

解析：
首先我们有一个已经排好序的数组：
图片

Step1：
第一步需要做的就是，从数组末尾开始，选取最后一个元素。
在数组一共9个位置中，随机产生一个位置，该位置元素与最后一个元素进行交换。

Step2：
我们已经把数组末尾元素进行随机置换。
接下来，对数组倒数第二个元素动手。在除去已经排好的最后一个元素位置以外的8个位置中，随机产生一个位置，该位置元素与倒数第二个元素进行交换。

Step3：
理解了前两部，接下来就是依次进行，如此简单。


## 自己实现乱序
以上方法，是基于Fisher–Yates shuffle洗牌算法。下面，我们就需要自己开动脑筋，完成一个乱序方案。
其实这并不难，关键在于如何生产真正的乱序。因为往往生成的并不是完全意义上的乱序，关于这一点，读者可以参考[The Danger of Naïveté](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle)一文。

我们来看一下社区上[刘哇勇](http://www.cnblogs.com/Wayou/p/fisher_yates_shuffle.html)的一系列进阶方案：

    function shuffle (array) {
        var copy = [],
            n = array.length,
            i;
        while (n) {
            i = Math.floor(Math.random() * array.length);
            if (i in array) {
                copy.push(array[i]);
                delete array[i];
                n--;
            }
        }
        return copy;
    }

关于这种方案，也给出了分析：
我们创建了一个copy数组，然后遍历目标数组，将其元素复制到copy数组里，同时将该元素从目标数组中删除，这样下次遍历的时候就可以跳过这个序号。而这一实现的问题正在于此，即使一个序号上的元素已经被处理过了，由于随机函数产生的数是随机的，所有这个被处理过的元素序号可能在之后的循环中不断出现，一是效率问题，另一个就是逻辑问题了，存在一种可能是永远运行不完。

改进的方案为：

    function shuffle(array) {
        var copy = [],
            n = array.length,
            i;
        while (n) {
            i = Math.floor(Math.random() * n--);
            copy.push(array.splice(i, 1)[0]);
        }
        return copy;
    }

改进的做法就是处理完一个元素后，用Array的splice()方法将其从目标数组中移除同时也更新了目标数组的长度。如此一来下次遍历的时候是从新的长度开始，不会重复处理的情况了。

当然这样的方案也有不足之处：比如，我们创建了一个copy数组进行返回，在内存上开辟了新的空间。
当然，这可以完全避免：

    function shuffle(array) {
        var m = array.length,
            t, i;
        while (m) {
            i = Math.floor(Math.random() * m--);
            t = array[m];
            array[m] = array[i];
            array[i] = t;
        }
        return array;
    }

有趣的是，这样的实现已经完全等同于上文洗牌算法Fisher–Yates shuffle的方案了。



## 总结
本文剖析了乱序数组，这么一个简单但是有趣的需求场景。并且介绍了部分V8引擎对array.sort的处理、洗牌算法Fisher–Yates等内容。
同时也安利了一个[可视化array.sort乱序的例证。]()希望对读者有所启发。










