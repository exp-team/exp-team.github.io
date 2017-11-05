---
title: 从 setState promise 化的探讨 体会 React 团队设计思想
categories: js
tags: js
date: 2017-11-04
author: 侯策
---

![Tomorrow Land 2017 - Martin Garrix](http://upload-images.jianshu.io/upload_images/4363003-cf1dab2cd6997834.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 从 setState 那个众所周知的小秘密说起...
在 React 组件中，调用 this.setState() 是最基本的场景。这个方法描述了 state 的变化、触发了组件 re-rendering。但是，也许看似平常的 this.setState() 里面却也许**蕴含了很多鲜为人知的设计和讨论。**

相信很多开发者已经意识到，setState 方法“或许”是**异步的。**也许你觉得，看上去更新 state 是如此轻而易举的操作，这并没有什么可异步处理的。但是要意识到，因为 state 的更新会触发 re-rendering，而 re-rendering 代价昂贵，短时间内反复进行渲染在性能上肯定是不可取的。所以，React 采用 batching 思想，它会 batches 一系列连续的 state 更新，而只触发一次 re-render。

关于这些内容，如果你还不清楚，推荐参考@程墨的系列文章：[setState：这个API设计到底怎么样](https://zhuanlan.zhihu.com/p/25954470)；英语好的话，可以直接关注长发飘飘的 [Eric Elliott 著名的引起系列口水战的吐槽文：setState() Gate](https://medium.com/javascript-scene/setstate-gate-abc10a9b2d82)。

或者，直接看下面的一个小例子。
比如，最简单的一个场景是：

    function incrementMultiple() {
      this.setState({count: this.state.count + 1});
      this.setState({count: this.state.count + 1});
      this.setState({count: this.state.count + 1});
    }

直观上来看，当上面的 incrementMultiple 函数被调用时，组件状态的
 count 值被增加了3次，每次增加1，那最后 count 被增加了3。但是，实际上的结果只给 state 增加了1。不信你自己试试～
 

## 让 setState 连续更新的几个 hack

如果想让 count 一次性加3，**应该如何优雅地处理潜在的异步操作，规避上述问题呢？**

以下提供几种解决方案：

- **方法一：**常见的一种做法便是将一个回调函数传入 setState 方法中。即 setState 著名的函数式用法。这样能保证即便在更新被 batched 时，也能访问到预期的 state 或 props。（后面会解释这么做的原理）

- **方法二：**另外一个常见的做法是需要在 setState 更新之后进行的逻辑（比如上述的连续第二次 count + 1），封装到一个函数中，并作为第二个参数传给 setState。这段函数逻辑将会在更新后由 React 代理执行。即：

   > setState(updater, [callback])

- **方法三：**把需要在 setState 更新之后进行的逻辑放在一个合适的生命周期 hook 函数中，比如 componentDidMount 或者 componentDidUpdate 也当然可以解决问题。也就是说 count 第一次 +1 之后，出发 componentDidUpdate 生命周期 hook，第二次 count +1 操作直接放在 componentDidUpdate 函数里面就好啦。


## 一个引起广泛讨论的 Issue

这些内容貌似已经不再新鲜，很多 React 资深开发者其实都是了解的，或能很快理解。

**可是，你想过这个问题吗：**
**现代 javascript 处理异步流程，很流行的一个做法是使用 promises，那么我们能否应用这个思路解决呢？**

说具体一些，就是调用 setState 方法之后，返回一个 promise，状态更新完毕后我们在调用 promise.then 进行下一步处理。

**答案是肯定的，但是却被官方否决了。**

我是如何得出“答案是肯定的，但是是不被官方建议的。”这个结论，喜欢刨根问底的读者请继续往下阅读，相信你一定会有所启发，也能更充分理解 React 团队的设计思想。


## 第 2642 Issue 解读和深入分析
我是一步一步在 Facebook 开源 React 的官方 [Github仓库](https://github.com/facebook/react)上，找到了线索。

整个过程跟下来，相信在各路大神的 comments 之间，你会对 React 的设计理念以及 javascript 解决问题的思路有一个更清晰的认识。

一切的探究始于 React 第 [#2642 号 issue: Make setState return a promise](https://github.com/facebook/react/issues/2642)，上面关于 count 连续 +3 大家已经有所了解。接下来我举一个真正在生产开发中的例子，方便大家理解讨论。

我们现在开发一个可编辑的 table，需求是：当用户敲下“回车”，光标将会进入下一行（调用 setState 进行光标移动）；如果用户当前已经在最后一行，那么敲下回车时，第一步将先创建一个新行（调用 setState 创建新的最后一行），在新行创建之后，再去新的最后一行进行光标聚焦（调用 setState 进行光标移动）。

常见且错误的处理在于：

    this.setState({
      selected: input
      // 创建新行
    }.bind(this));
    this.props.didSelect(this.state.selected);
    
因为第一个 this.setState 是异步进行的话，下一处 didSelect 方法执行 this.setState 时，所处理的参数 this.state.selected 可能还不是预期的下一行。很明显，这就是 this.setState 的异步性带来的问题。

为了解决这个完成这样的逻辑，想到了 setState 第二个参数解决方案，用代码简单表述就是：

    this.setState({
      selected: input
      // 创建新行
    }, function() {
        this.props.didSelect(this.state.selected);
    }).bind(this));

这种解决方案是使用嵌套的 setState 方法。但这无疑潜在地会带来嵌套地狱的问题。


## Promise 化方案登场

这一切是不是像极了传统 Javascript 处理异步老套路？解决回调地狱，你是不是应激性地想到了 promise？

如果 setState 方法返回的是一个 promises，自然会更加优雅：

> setState() currently accepts an optional second argument for callback and returns undefined.
This results in a callback hell for a very stateful component. Having it return a promise would make it much more managable.

如果用 promise 风格解决问题的话，无非就是：

    this.setState({
      selected: input
    }).then(function() {
      this.props.didSelect(this.state.selected);
    }.bind(this));

看上去没什么问题，一个很时髦的设计。但是，我们进一步想：**如果想让 React 支持这样的特性，采用提出 pull request 的方式，我们该如何去改源代码呢？**


## 探索 React 源码，完成 setState promise 化的改造

首先找到源码中关于 setState 定义的地方，它在 react/src/isomorphic/modern/class/ReactBaseClasses.js 这个目录下：

    ReactComponent.prototype.setState = function(partialState, callback) {
      invariant(
        typeof partialState === 'object' ||
          typeof partialState === 'function' ||
          partialState == null,
        'setState(...): takes an object of state variables to update or a ' +
          'function which returns an object of state variables.',
      );
      this.updater.enqueueSetState(this, partialState, callback, 'setState');
    };

我们首先看到一句注释：

> You can provide an optional callback that will be executed when the call to setState is actually completed.

这是采用 setState 第二个参数传入处理回调的基础。

另外，从注释中我们还找到：

> When a function is provided to setState, it will be called at some point in the future (not synchronously). It will be called with the up to date component arguments (state, props, context). 

这是给 setState 方法直接传入一个函数的基础。

 **言归正传，如何改动源码，使得 setState promise 化呢？**
 其实很简单，我直接上代码：

     ReactComponent.prototype.setState = function(partialState, callback) {
       invariant(
         typeof partialState === 'object' ||
           typeof partialState === 'function' ||
           partialState == null,
          'setState(...): takes an object of state variables to update or a ' +
            'function which returns an object of state variables.',
        );
     +  let callbackPromise;
     +  if (!callback) {
     +    class Deferred {
     +      constructor() {
     +        this.promise = new Promise((resolve, reject) => {
     +          this.reject = reject;
     +          this.resolve = resolve;
     +        });
     +      }
     +    }
     +    callbackPromise = new Deferred();
     +    callback = () => {
     +      callbackPromise.resolve();
     +    };
     +  }
        this.updater.enqueueSetState(this, partialState, callback, 'setState');
     +
     +  if (callbackPromise) {
     +    return callbackPromise.promise;
     +  }
      }; 

我用 “＋” 标注了对源码所做的更改。如果开发者调用 setState 方法时，传入的是一个 javascript 对象的话，那么会返回一个 promise，这个 promise 将会在 state 更新完毕后 resolve。
如果您看不懂的话，建议补充一下相关的基础知识，或者留言与我讨论。



## 解决方案有了，可是 React 官方会接受这个 PR 吗？

很遗憾，答案是否定的。我们来从 React 设计思想上，和 React 官方团队的回应上，了解一下否决理由。

sebmarkbage（Facebook 工程师，React 核心开发者）认为：解决异步带来的困扰方案其实很多。比如，我们可以在合适的生命周期 hook 函数中完成相关逻辑。在这个场景里，就是在行组件的 componentDidMount 里调用 focus，自然就完成了自动聚焦。

此外，还有一个方法：新的 refs 接口设计支持接收一个回调函数，当其子组件挂载时，这个回调函数就会相应触发。

**所有上述模式都可以完全取代之前的问题方案，即使不能也不意味着要接受 promises 化这个PR。**

为此，sebmarkbage 说了一段很扎心的话：

> Honestly, the current batching strategy comes with a set of problems right now. I'm hesitant to expand on it's API before we're sure that we're going to keep the current model. I think of it as a temporary escape until we figure out something better.

问题的根源在于现有的 batching 策略，实话实说，这个策略带来了一系列问题。也许这个在后期后有调整，在 batching 策略是否调整之前，**盲目的扩充 setState 接口只会是一个短视的行为。**

对此，Redux 原作者 Dan Abramov 也发表了自己的看法。他认为，以他的经验来看，任何需要使用 setState 第二个参数 callback 的场景，都可以使用生命周期函数  componentDidUpdate (and/or componentDidMount) 来复写。

> In my experience, whenever I'm tempted to use setState callback, I can achieve the same by overriding componentDidUpdate (and/or componentDidMount).

另外，在一些极端场景下，如果开发者确实需要同步的处理方式，比如如果我想在某 DOM 元素挂载到屏幕之前做一些操作，promises 这种方案便不可行。因为 Promises 总是异步的。反过来，如果 setState 支持这两种不同的方式，那么似乎也是完全没有必要而多余的。

在社区，确实很多第三方库渐渐地接受使用 promises 风格，但是这些库解决的问题往往都是强异步性的，比如文件读取、网络操作等等。 React 似乎没有必要增加这么一个 confusing 的特性。

另外，如果每个 setState 都返回一个 promises，也会带来性能影响：对于 React 来说，setState 将必然产生一个 callback，这些 callbacks 需要合理储存，以便在合适时间来触发。

总结一下，解决 setState 异步带来的问题，有很多方式能够完美优雅地解决。在这种情况下，直接让 setState 返回 promise 是画蛇添足的。另外，这样也会引起性能问题等等。

我个人认为，这样的思路很好，但是难免有些 Overengineering。



## 这一次为自己疯狂，我和我的倔强

怎么样，是否说服你了呢？如果没有，在不能更改 React 源码情况下，你就是想用 promise 化的 setState，怎么办呢？

这里提供一个“反模式”的方案：我们不改变源码，自己也可以进行改造，原理上就是直接对 this.setState 进行拦截，进而进行 promise 化，再封装一个新的接口出来。

    import Promise from "bluebird";

    export default {
      componentWillMount() {
        this.setStateAsync = Promise.promisify(this.setState);
      },
    };

之后，便可以异步地：

    this.setStateAsync({
      loading: true,
    }).then(this.loadSomething).then((result) => {
      return this.setStateAsync({result, loading: false});
    });
    
当然，也可以使用原声的 promises：

    function setStatePromise(that, newState) {
        return new Promise((resolve) => {
            that.setState(newState, () => {
                resolve();
            });
        });
    }

甚至...我们还可以脑洞大开使用 async/await。
    
最后，所有这种做法非常的 dirty，我是不建议这么使用的。



## 总结
其实研究一下 React Issue，深入源码学习，收获确实很多。总结也没有更多想说的了，无耻滴做个广告吧：

 **我的其他关于 React 文章：**
 
- [React Redux 中间件思想遇见 Web Worker 的灵感（附demo）](https://zhuanlan.zhihu.com/p/28525821)

- [通过实例，学习编写 React 组件的“最佳实践”](https://zhuanlan.zhihu.com/p/27825741)

- [React 组件设计和分解思考](https://zhuanlan.zhihu.com/p/27727292)

- [从 React 绑定 this，看 JS 语言发展和框架设计]()

- [React 服务端渲染如此轻松 从零开始构建前后端应用](https://zhuanlan.zhihu.com/p/28004982)

- [做出Uber移动网页版还不够 极致性能打造才见真章](http://www.jianshu.com/p/49029b49f2b4)

- [解析Twitter前端架构 学习复杂场景数据设计](http://www.jianshu.com/p/7a56ac1de2a8)

- [React Conf 2017 干货总结1: React + ES next = ♥](http://www.jianshu.com/p/83c86dd0802d)

- [React+Redux打造“NEWS EARLY”单页应用 一个项目理解最前沿技术栈真谛](http://www.jianshu.com/p/cde3cf7e2760)

- [一个react+redux工程实例](http://www.jianshu.com/p/8e28be0e7ab1)





Happy Coding!

PS: 
作者[Github仓库](https://github.com/HOUCe) 和 [知乎问答链接](https://www.zhihu.com/people/lucas-hc/answers)
欢迎各种形式交流。