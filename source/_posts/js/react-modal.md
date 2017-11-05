---
title: React 模态框秘密和“轮子”渐进设计
categories: js
tags: js
date: 2017-11-04
author: 侯策
---

今天上午组内小朋友们谈到 React 实践，提到 React 模态框（弹窗）的使用。我发现很多一些 React 开发者对于 React 模态框的具体设计思路和实现存在一些疑惑。因而特写此文，分享我对**模态框这个“重要且典型”的前端交互，在 React 框架里实现的一些想法。**准备时间短促且匆忙，难免有遗漏之处，希望大神给予斧正。

这篇文章“进阶式”渐进地，由浅入深分析三种实现。从最初的简单粗暴到接近 [react-modal](https://github.com/reactjs/react-modal) 库设计思想，一步步打磨分析，适合初学者阅读思考。

## 原始级实现 —— 暴力美学
世界上大部分网站都离不开模态框交互。事实上，模态框就是我们俗称的“弹窗”，只不过这个弹窗相比简单的：

    alert('我是一个简单、原生的 alert～');
    
多了更多的信息承载和交互行为。同时为了更佳美化和吸引眼球，模态框往往伴随着深色透明的遮罩。比如下图：


![模态框举例](http://upload-images.jianshu.io/upload_images/4363003-bdbf145a382c6272.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

想想常见的用户登录框、错误信息提示等等，都是非常典型的模态框实现。

在传统的 jQuery 操作 DOM 类库的技术栈下，我们可以“肆无忌惮”地选择 DOM 节点，完成 append, remove 等操作，实现模态框并不复杂。可是在 React 和 Redux 世界里，我们该如何实现？

我们先来看一下场景和初版设计思路：

![场景和初版设计思路](http://upload-images.jianshu.io/upload_images/4363003-48669842b6993eef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



如图，箭头标记的组件需要触发模态框的出现。
图中组件树对应基本页面代码如下：

    export default class App extends Component {
        render() {
            <div className="app">
                <div className="left">
                    <h1>Hello left</h1>
                    // ...
                </div>

                <div className="right">
                    <h1>Hello right</h1>
                    // ...
                    <div>
                        <BadModal>
                            // 模态框内容
                            <h1> Modal title </h1>
                            <p> Modal content</p>
                        </BadModal>
                    </div>
                </div>
            </div>
        }
    }


细心的读者会发现，作为 amazing 的程序员，尽管这是最初版本的实现，但是还是思考一些最基本的“复用”问题。

我们设计完成的模态框组件 <BadModal>，因为每个模态框里内容和交互不尽相同，所以在 <BadModal> 组件内，我们渲染 child component，这个 child component 即业务对应的模态框内容，它将会由业务逻辑开发完成，实现模态框内容、交互的复用。如下代码：

    
    class BadModal extends Comment {
        render() {
            return (
                <div className="modal">
                    { this.props.children }
                </div>
            )
        }
    }

至此，我们已经实现了最基本的模态框。可是为什么说这是最原始、简陋的方法呢？细想一下，似乎不完美的地方还很多。翻开我们的样式表：

    body .modal {
        position: fixed;
        // ...
    }
    .left {
        z-index: 3
    }
    .right {
        z-index: 1
    }
    
你会发现恼人的 z-index 问题，我们模态框是 .right 节点的子孙节点，而 .right 的 z-index 小于 .left 的 z-index，这样造成的直接问题就是模态框最终不能脱离页面整体而“突出显示”！

**细想一下，这个问题的根本就出现在我们的组件设计图中。**
仔细观察上图，因为很深层次的子孙组件触发模态框，而使得该组件内的模态框组件层级较深。如果你对 z-index 比较规则有所了解的话，这样的情况很难完成模态框凌驾于页面整体而出现的，遮罩也无法覆盖整个页面。

想想我们平时使用的 jQuery 是怎么做的吧：

    $('body').append('<div class="overlay"></div>');
    
一般情况，模态框和遮罩总是作为在 body 下的第一层子节点出现。由此，引出了我们的第二种进阶思路。请读者继续阅读。

## 实现方案二 —— 乾坤大挪移
解决方法很简单，我们可以很自然地想到：只需要**对 <Modal> 组件出现的位置进行移动。可是这就需要 <Modal> 组件和触发模态框出现的深层次组件进行某种意义上的通信。**

传统的 React 组件间通信无外乎 props 和基于 props 的回调实现（不考虑 context 的黑魔法）。可是这样的做法太过复杂，也难以实现复用，更不利于维护。

至于我这里采用的做法，还要从调整后的页面组件树设计出发：

![](http://upload-images.jianshu.io/upload_images/4363003-97c2399ecd828fb4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


如图，我们在 document.body 下加入了 <Modal> 组件，并列于 Root Component。同时，至关重要的一步设计是，**我们在触发模态框的组件下，加入了一个 Fake Modal 组件。**

这个神秘的 Fake Modal 组件做了什么呢？
**事实上，他并不渲染任何结果，而是借助其生命周期函数，完成在 document.body 下新建并插入 <Modal> 组件的使命。**

借助代码进行理解：

    class Modal extends Comment {
        componentDidMount() {
            this.modalTarget = document.createElement('div');
            this.modalTarget.className = 'modal';
            document.body.appendChild(this.modalTarget);
            this.renderModal();
        }
        componentWillUpdate() {
            this.renderModal();
        }
        componentWillUnmount() {
            ReactDom.unmountComponentAtNode(this.modalTarget);
            document.body.removeChild(this.modalTarget)l
        }
        renderModal() {
            ReactDom.render(
                <div>{ this.props.children }</div>,
                this.modalTarget
            );
        }
        render() {
            return <noscript />
        }
    }
    
具体进行分析在真正的 render 方法中，我们不渲染任何实质的内容，而是：

    return <noscript />;
    
同时，借助生命周期函数 componentDidMount，我们使用原生 JavaScript 实现在 body 下的模态框创建：

    this.modalTarget = document.createElement('div');
    this.modalTarget.className = 'modal';
    document.body.appendChild(this.modalTarget);
    this.renderModal();
    
并最终调用 renderModal 方法完成插入：

    ReactDom.render(
        <div>{ this.props.children }</div>,
        this.modalTarget
    );
    

## 实现方案三 —— 搭配 Redux
相信很多 React 开发者都会使用 Redux 来做数据管理。仔细看上图的结构中，我们难以实现对 Redux 的友好兼容。

![image.png](http://upload-images.jianshu.io/upload_images/4363003-8039ace2b5b26aaa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


图片

比如说，如果在 <Modal> 组件的子组件 child component 中，需要使用 Redux store 里的数据，那么因为 <Provider> 实质上是一个“高阶组件”且不在 <Modal> 组件的组件链中，因为 child component 无法感知 Redux store 的存在。

为了解决这个问题，我们继续改进组件树结构为：

![image.png](http://upload-images.jianshu.io/upload_images/4363003-4643613c42286b86.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

图片

为此，我们引入应用的 store，以及 react-redux 包提供的 <Provider> 组件：

    import { store } from '../index';
    import { Provider } from 'react-redux';

同时改动先前的 renderModal 方法，加入对 <Provider> 的支持：

    renderModal() {
        ReactDom.render(
            <Provider store={ store }>
                <div>{ this.props.children }</div>
            <Provider>,
            this.modalTarget
        );
    }
  

## 著名的 react-modal 探秘和 React16 版本惊喜
在 React 开发中，我想很多工程师对 [react-modal](https://github.com/reactjs/react-modal) 非常熟悉。我们往往依赖它，完成模态框的使用。

这个库设计良好，请封装完善。如果你好奇它是如何实现的，源码又是如何组织？那么我可以告诉你，你已经了解了他的设计哲学。事实上，文章介绍的思路就是它的奥秘。

了解了这些，你也可以动手实现“一个轮子”，或者扩充本文源码，实现更多的功能。比如样式的自定义、弹出前后的回调等等。相信一定会有很多收获。

同时，React 最新版本 0.16 已经横空出世，它带来的很多新特性之一就与本文密切相关。那就是 —— Portal，Portal 我们把它翻译为“传送门、任意门”。Portals 允许将组件渲染到父节点之外的 DOM 节点中。它的基本使用如下代码示例：

    render() {
        return ReactDOM.createPortal(
                    this.props.children,
                    anyDomNode,
                );
    }

这里 React 并不会在当前结构中渲染组件，而是向 anyDomNode 中渲染 this.props.children，这里的 anyDomNode 是任何有效的DOM节点，无论它处于哪个层级位置。

了解了这些，我们当然能够使用此特性，简化上文逻辑。翻开 react-modal 最新提交的源码，便能够发现对这一新特性的支持，react-modal/src/components/Modal.js 文件中：

    const isReact16 = ReactDOM.createPortal !== undefined;
    const createPortal = isReact16
      ? ReactDOM.createPortal
      : ReactDOM.unstable_renderSubtreeIntoContainer;

这里对 React 版本进行判断，并设置 isReact16 标识位表示是否支持 createPortal 的方法（个人认为这个标识位的命名非常不合适...）

最终在 render 方法内：

    render() {
        if (!canUseDOM || !isReact16) {
            return null;
        }
    
        if (!this.node && isReact16) {
            this.node = document.createElement("div");
        }
    
        return createPortal(
            <ModalPortal
                ref={this.portalRef}
                defaultStyles={Modal.defaultStyles}
                {...this.props}
            />,
            this.node
        );
    }
    
非常明显地看到，对于不支持 createPortal 的情况采用与我们类似的 return null; 否则愉快地使用 createPortal 方法。

## 总结
本文介绍内容虽然基础，但是很好地贯穿了 React 思想以及实现一个“模态框轮子”的演进思路。同时介绍了 React 新版本的一项特性。

我的其他几篇关于React技术栈的文章：

- [React Redux 中间件思想遇见 Web Worker 的灵感（附demo）](https://zhuanlan.zhihu.com/p/28525821)
- [了解 Twitter 前端架构 学习复杂场景数据设计](https://zhuanlan.zhihu.com/p/29732224)
- [React 探秘 - React Component 和 Element（文末附彩蛋demo和源码）](https://zhuanlan.zhihu.com/p/29711902)
- [从setState promise化的探讨 体会React团队设计思想](https://zhuanlan.zhihu.com/p/28905707)
- [通过实例，学习编写 React 组件的“最佳实践”](https://zhuanlan.zhihu.com/p/27825741)
- [React 组件设计和分解思考](https://zhuanlan.zhihu.com/p/27727292)
- [从 React 绑定 this，看 JS 语言发展和框架设计](https://zhuanlan.zhihu.com/p/28905707/edit)
- [React 服务端渲染如此轻松 从零开始构建前后端应用](https://zhuanlan.zhihu.com/p/28004982)
- [做出Uber移动网页版还不够 极致性能打造才见真章**](http://link.zhihu.com/?target=http%3A//www.jianshu.com/p/49029b49f2b4)
- [React+Redux打造“NEWS EARLY”单页应用 一个项目理解最前沿技术栈真谛**](http://link.zhihu.com/?target=http%3A//www.jianshu.com/p/cde3cf7e2760)

Happy Coding!
PS: 作者 [Github仓库**](http://link.zhihu.com/?target=https%3A//github.com/HOUCe) 和 [知乎问答链接](https://www.zhihu.com/people/lucas-hc/answers) 欢迎各种形式交流。