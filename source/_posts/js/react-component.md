---
title: React 组件设计和分解思考
categories: js
tags: js
date: 2017-11-04
author: 侯策
---

![React](http://upload-images.jianshu.io/upload_images/4363003-7fd2c6c5f0f372a7.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


之前分享过几篇关于React技术栈的文章：

- [做出Uber移动网页版还不够 极致性能打造才见真章](http://www.jianshu.com/p/49029b49f2b4)
- [解析Twitter前端架构 学习复杂场景数据设计](http://www.jianshu.com/p/7a56ac1de2a8)
- [React Conf 2017 干货总结1: React + ES next = ♥](http://www.jianshu.com/p/83c86dd0802d)
- [React+Redux打造“NEWS EARLY”单页应用 一个项目理解最前沿技术栈真谛](http://www.jianshu.com/p/cde3cf7e2760)
- [一个react+redux工程实例](http://www.jianshu.com/p/8e28be0e7ab1)
- ......


今天再来同大家讨论 React 组件设计的一个有趣话题：**分解 React 组件的几种进阶方法。**

React 组件魔力无穷，同时灵活性超强。我们可以在组件的设计上，玩转出很多花样。但是保证组件的[Single responsibility principle: 单一原则](https://en.wikipedia.org/wiki/Single_responsibility_principle)非常重要，它可以使得我们的组件更简单、更方便维护，更重要的是使得组件更加具有复用性。

但是，如何对一个功能复杂且臃肿的 React 组件进行分解，也许并不是一件简单的事情。本文由浅入深，介绍三个分解 React 组件的方法。


## 切割 render() 方法
这是一个最容易想到的方法：当一个组件渲染了很多元素时，就需要尝试分离这些元素的渲染逻辑。最迅速的方式就是切割 render() 方法为多个 sub-render 方法。

看下面的例子会更加直观：

    class Panel extends React.Component {
      renderHeading() {
        // ...
      }
    
      renderBody() {
        // ...
      }
    
      render() {
        return (
          <div>
            {this.renderHeading()}
            {this.renderBody()}
          </div>
        );
      }
      
细心的读者很快就能发现，**其实这并没有分解组件本身，**该 Panel 组件仍然保持有原先的 state, props, 以及 class methods。

如何真正地做到减少复杂度呢？我们需要创建一些子组件。此时，采用最新版 React 支持并推荐的函数式组件／无状态组件一定会是一个很好的尝试：

    const PanelHeader = (props) => (
      // ...
    );
    
    const PanelBody = (props) => (
      // ...
    );
    
    class Panel extends React.Component {
      render() {
        return (
          <div>
            // Nice and explicit about which props are used
            <PanelHeader title={this.props.title}/>
            <PanelBody content={this.props.content}/>
          </div>
        );
       }
     }
     
同之前的方式相比，这个微妙的改进是革命性的。我们新建了两个单元组件：PanelHeader 和 PanelBody。这样带来了测试的便利，我们可以直接分离测试不同的组件。同时，借助于 React 新的算法引擎 [React Fiber，](https://github.com/acdlite/react-fiber-architecture)两个单元组件在渲染的效率上，乐观地预计会有较大幅度的提升。

## 模版化组件
回到问题的起点，为什么一个组件会变的臃肿而复杂呢？其一是渲染元素较多且嵌套，另外就是组件内部变化较多，或者存在多种 configurations 的情况。

此时，我们便可以将组件改造为模版：父组件类似一个模版，只专注于各种 configurations。

还是要举例来说，这样理解起来更加清晰。

比如我们有一个 Comment 组件，这个组件存在多种行为或事件。同时组件所展现的信息根据用户的身份不同而有所变化：用户是否是此 comment 的作者，此 comment 是否被正确保存，各种权限不同等等都会引起这个组件的不同展示行为。这时候，与其把所有的逻辑混淆在一起，也许更好的做法是利用 React 可以传递 React element 的特性，我们将 React element 进行组件间传递，这样就更加像一个模版：

    class CommentTemplate extends React.Component {
      static propTypes = {
        // Declare slots as type node
        metadata: PropTypes.node,
        actions: PropTypes.node,
      };
      
      render() {
        return (
          <div>
            <CommentHeading>
              <Avatar user={...}/>
              
              // Slot for metadata
              <span>{this.props.metadata}</span>
              
            </CommentHeading>
        
            <CommentBody/>
            
            <CommentFooter>
              <Timestamp time={...}/>
              
              // Slot for actions
              <span>{this.props.actions}</span>
              
            </CommentFooter>
          </div>
          ...
          
此时，我们真正的 Comment 组件组织为：

    class Comment extends React.Component {
      render() {
        const metadata = this.props.publishTime ?
          <PublishTime time={this.props.publishTime} /> :
          <span>Saving...</span>;
        
        const actions = [];
        if (this.props.isSignedIn) {
          actions.push(<LikeAction />);
          actions.push(<ReplyAction />);
        }
        if (this.props.isAuthor) {
          actions.push(<DeleteAction />);
        }
        
        return <CommentTemplate metadata={metadata} actions={actions} />;
      }
      
metadata 和 actions 其实就是在特定情况下需要渲染的 React element。

比如，如果 this.props.publishTime 存在，metadata 就是 <PublishTime time={this.props.publishTime} />；反正则为 <span>Saving...</span>。

如果用户已经登陆，则需要渲染（即actions值为） <LikeAction /> 和 <ReplyAction />，如果是作者本身，需要渲染的内容就要加入 <DeleteAction />。

## 高阶组件
在实际开发当中，组件经常会被其他需求所污染。

比如，我们想统计页面中所有链接的点击信息。在链接点击时，发送统计请求，同时包含此页面 document 的 id 值。常见的做法是在 Document 组件的生命周期函数 componentDidMount 和 componentWillUnmount 增加代码逻辑：

    class Document extends React.Component {
      componentDidMount() {
        ReactDOM.findDOMNode(this).addEventListener('click', this.onClick);
      }
      
      componentWillUnmount() {
        ReactDOM.findDOMNode(this).removeEventListener('click', this.onClick);
      }
      
      onClick = (e) => {
        if (e.target.tagName === 'A') { // Naive check for <a> elements
          sendAnalytics('link clicked', {
            documentId: this.props.documentId // Specific information to be sent
          });
        }
      };
      
      render() {
        // ...
        
这么做的几个问题在于：

- 相关组件 Document 除了自身的主要逻辑：显示主页面之外，多了其他统计逻辑；
- 如果 Document 组件的生命周期函数中，还存在其他逻辑，那么这个组件就会变的更加含糊不合理；
- 统计逻辑代码无法复用；
- 组件重构、维护都会变的更加困难。

为了解决这个问题，我们提出了高阶组件这个概念：[ higher-order components (HOCs)](https://facebook.github.io/react/docs/higher-order-components.html)。不去晦涩地解释这个名词，我们来直接看看使用高阶组件如何来重构上面的代码：

    function withLinkAnalytics(mapPropsToData, WrappedComponent) {
      class LinkAnalyticsWrapper extends React.Component {
        componentDidMount() {
          ReactDOM.findDOMNode(this).addEventListener('click', this.onClick);
        }
    
        componentWillUnmount() {
          ReactDOM.findDOMNode(this).removeEventListener('click', this.onClick);
        }
    
        onClick = (e) => {
          if (e.target.tagName === 'A') { // Naive check for <a> elements
            const data = mapPropsToData ? mapPropsToData(this.props) : {};
            sendAnalytics('link clicked', data);
          }
        };
        
        render() {
          // Simply render the WrappedComponent with all props
          return <WrappedComponent {...this.props} />;
        }
      }


需要注意的是，withLinkAnalytics 函数并不会去改变 WrappedComponent 组件本身，更不会去改变 WrappedComponent 组件的行为。而是返回了一个被包裹的新组件。实际用法为：

    class Document extends React.Component {
      render() {
        // ...
      }
    }
    
    export default withLinkAnalytics((props) => ({
      documentId: props.documentId
    }), Document);
    
这样一来，Document 组件仍然只需关心自己该关心的部分，而 withLinkAnalytics 赋予了复用统计逻辑的能力。

高阶组件的存在，完美展示了 React 天生的复合（compositional）能力，在 React 社区当中，react-redux，styled-components，react-intl 等都普遍采用了这个方式。值得一提的是，[recompose](https://github.com/acdlite/recompose/) 类库又利用高阶组件，并发扬光大，做到了“脑洞大开”的事情。



## 总结
React 及其周边社区的崛起，让函数式编程风靡一时，受到追捧。其中关于 decomposing 和 composing 的思想，我认为非常值得学习。同时，对开发设计的一个建议是，不要犹豫将你的组件拆分的更小、更单一，因为这样能换来强健和复用。

本文意译了[Chris Nwamba的：React Universal with Next.js: Server-side React 一文，](https://medium.com/dailyjs/techniques-for-decomposing-react-components-e8a1081ef5da)并对原文进行了升级，兼容了最新的 Next 设计。

Happy Coding!

PS: 作者[Github仓库](https://github.com/HOUCe)，欢迎通过代码各种形式交流。