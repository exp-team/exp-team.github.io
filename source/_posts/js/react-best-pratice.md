---
title: 通过实例，学习编写React组件的最佳实践
categories: js
tags: js
date: 2017-04-14
author: 侯策
---

现在前端程序员都知道，React是组件化的。当我开始学习React的时候，我记得当时已经存在了很多不同编写组件的方式了。这个教程，那个教程，也许组件的组织方式不尽相同。如今，React社区已经愈发成熟，但是对于组件正确编写姿势却没有一个相对完备的指导。
这篇文章从作者的观点出发，来谈一谈我们究竟应该如何来写React组件。

在开始前，需要说明以下几个问题：
1）这篇文章以及代码实例，都采用了ES6或者ES7的写法；
2）对于一些基本概念不再进行科普。比如，如果你还不知道木偶组件（展示组件）和容器组件的区别，建议先对React基础进行学习；
3）如果有任何问题，欢迎留言交流。

另外，这篇文章并不是我原创，我翻译了[Our Best Practices for Writing React Components](https://engineering.musefind.com/our-best-practices-for-writing-react-components-dec3eb5c3fc8)一文，并在此基础上进行了较大扩展。
如果您对React生态有兴趣，推荐我的其他几篇文章：



## 基于Class的组件编写Class Based Components
基于Class的组件是状态化的，包含有方法和属性等。最佳实践包括但不限于以下内容：

### 引入CSS依赖Importing CSS
我很喜欢CSS in JavaScript这一理念。但是在此之前，这都是停留于理论层面的。直到我们可以为每一个React组件引入相应的CSS文件，这一“梦想”成为了现实。在下面的代码示例中，我把CSS文件的引入与其他依赖隔行分开，以示区别：

    import React, {Component} from 'react'
    import {observer} from 'mobx-react'

    import ExpandableForm from './ExpandableForm'
    import './styles/ProfileContainer.css'

### 设定初始状态Initializing State
在编写组件过程中，一定要注意的是初始状态的设定。并且利用ES6模块化的知识，我们确保该组件的对外暴露都是“export default”形式，方便其他模块（组件）的调用。

    import React, {Component} from 'react'
    import {observer} from 'mobx-react'

    import ExpandableForm from './ExpandableForm'
    import './styles/ProfileContainer.css'

    export default class ProfileContainer extends Component {
        state = { expanded: false }


### 设定propTypes和defaultProps
propTypes和defaultProps都是组件的静态属性。在组件的代码中，这两个属性的设定位置越高越好。因为这样方便其他阅读代码者或者自己，一眼就能看到这些信息。这些信息就如同组件文档一样，对于理解或熟悉当前组件非常重要。
同样，原则上，你编写的组件都需要有propTypes属性。如同以下代码：


    export default class ProfileContainer extends Component {
        state = { expanded: false }

        static propTypes = {
            model: React.PropTypes.object.isRequired,
            title: React.PropTypes.string
        }

        static defaultProps = {
            model: {
                id: 0
            },
            title: 'Your Name'
        }

### 组件方法Methods
在编写方法时，尤其是你将一个方法作为props传递给子组件时，你需要确保this的正确指向。我们通常使用bind或者ES6箭头函数来达到此目的。

    export default class ProfileContainer extends Component {
        state = { expanded: false }

        handleSubmit = (e) => {
            e.preventDefault()
            this.props.model.save()
        }

        handleNameChange = (e) => {
            this.props.model.changeName(e.target.value)
        }

        handleExpand = (e) => {
            e.preventDefault()
            this.setState({ expanded: !this.state.expanded })
        }

### setState接受一个函数作为参数Passing setState a Function
在上面的代码示例中，我们使用了：

    this.setState({ expanded: !this.state.expanded })

这里，关于setState hook函数，其实有一个非常“有意思”的问题。React在设计时，为了性能上的优化，采用了Batch思想，会收集“一波”state的变化，统一进行处理。就像浏览器绘制的实现一样。所以setState之后，state也许不会马上就发生变化。

这说明，我们要谨慎地在setState中使用当前的state，因为当前的state也许是不可靠的。
为了规避这个问题，我们可以这样做：

    this.setState(prevState => ({ expanded: !prevState.expanded }))

我们给setState方法传递一个函数，函数参数为上一刻state，来保证setState能够立刻执行。

关于React setState的设计，我的长发“男神”Eric Elliott也曾经这么喷过：[setState() Gate](https://medium.com/javascript-scene/setstate-gate-abc10a9b2d82#.ftefj7nn2)
如果你对setState方法的异步性还有困惑，可以同我讨论，这里不再展开。


### 合理利用解构Destructuring Props
这个其实没有太多可说的，仔细观察代码吧。我们使用了解构赋值。除此之外，如果一个组件有很多的props的话，每个props应该都另起一行，这样子书写上和阅读性上都有更好的体验。

    export default class ProfileContainer extends Component {
        state = { expanded: false }

        handleSubmit = (e) => {
        e.preventDefault()
        this.props.model.save()
        }

        handleNameChange = (e) => {
            this.props.model.changeName(e.target.value)
        }

        handleExpand = (e) => {
            e.preventDefault()
            this.setState(prevState => ({ expanded: !prevState.expanded }))
        }

        render() {
            const {model, title} = this.props

            return ( 
                <ExpandableForm 
                onSubmit={this.handleSubmit} 
                expanded={this.state.expanded} 
                onExpand={this.handleExpand}>
                    <div>
                        <h1>{title}</h1>
                        <input
                        type="text"
                        value={model.name}
                        onChange={this.handleNameChange}
                        placeholder="Your Name"/>
                    </div>
                </ExpandableForm>
            )
        }
    }


### 使用修饰器Decorators
这一条是对使用mobx的开发者来说的。如果你不懂mobx，可以大体扫一眼（作为翻译者，其实我也不是用mobx的）。
我们强调使用decorate来修饰我们的组件，如同：
    
    @observer
    export default class ProfileContainer extends Component {

使用修饰器更加灵活且可读性更高的实践。即便你不使用修饰器，也需要如此暴露你的组件：

    class ProfileContainer extends Component {
        // Component code
    }
    export default observer(ProfileContainer)


### 闭包Closures
一定要尽量避免以下用法：

    <input
        type="text"
        value={model.name}
        // onChange={(e) => { model.name = e.target.value }}
        // ^ Not this. Use the below:
        onChange={this.handleChange}
        placeholder="Your Name"/>

总结一下，就是不要

    onChange = {(e) => { model.name = e.target.value }}

而是：

    onChange = {this.handleChange}

原因其实很简单，每次父组件render的时候，都会新建一个新的函数并传递给input。
如果input是一个React组件，这会粗暴地直接导致这个组件的re-render，需要知道，Reconciliation可是React成本最高的部分。
另外，我们推荐的方法，会使得阅读、调试和更改更加方便。


## 函数式组件Functional Components
以上内容是针对Class Components来讲的，下面我们看一下Functional Components的最佳实践。
Functional Components是指没有状态、没有方法，纯组件。我们应该最大限度地编写和使用这一类组件。

### 使用propTypes
在组件声明之前，我们需要使用propTypes，以达到更好的代码组织效果。
当然，依赖JavaScript的函数提升function hoisting，这么做不会报错。


    ExpandableForm.propTypes = {
        onSubmit: React.PropTypes.func.isRequired,
        expanded: React.PropTypes.bool
    }
    // Component declaration


### 解构Destructuring Props and defaultProps
我们先来看一下反例：

    ExpandableForm.propTypes = {
        onSubmit: React.PropTypes.func.isRequired,
        expanded: React.PropTypes.bool,
        onExpand: React.PropTypes.func.isRequired
    }
    function ExpandableForm(props) {
        const formStyle = props.expanded ? {height: 'auto'} : {height: 0}
        return (
            <form style={formStyle} onSubmit={props.onSubmit}>
                {props.children}
                <button onClick={props.onExpand}>Expand</button>
            </form>
        )
    }

Our component is a function, which takes its props as its argument. We can expand them like so:
组件其实是一个function，他的参数为props，在充分利用解构的情况下，我们可以这样重构：

    function ExpandableForm({ onExpand, expanded = false, children, onSubmit }) {
        const formStyle = expanded ? {height: 'auto'} : {height: 0}
        return (
            <form style={formStyle} onSubmit={onSubmit}>
                {children}
            <button onClick={onExpand}>Expand</button>
            </form>
        )
    }

上面代码，我们也给expanded参数设置了默认值。


### 封装Wrapping
对于functional components，无法使用修饰符。这种情况下，只需要简单地将函数作为参数传递就可以了：

    
    function ExpandableForm({ onExpand, expanded = false, children, onSubmit }) {
        const formStyle = expanded ? {height: 'auto'} : {height: 0}
        return (
            <form style={formStyle} onSubmit={onSubmit}>
                {children}
                <button onClick={onExpand}>Expand</button>
            </form>
        )
    }


## JSX中的条件判别Conditionals in JSX
真正写过React项目的同学一定会明白，JSX中可能会存在大量的条件判别，以达到根据不同的情况渲染不同组件形态的效果。
就像下图这样：


![截图1](http://upload-images.jianshu.io/upload_images/4363003-b75e43cb28cd54db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


然后，这样的结果是不理想的。我们丢失了代码的可读性，也使得代码组织显得混乱异常。多层次的嵌套也是应该避免的。

针对于此，有很对类库来解决此类问题JSX-Control Statements，但是与其引入第三方类库的依赖，还不如我们先自己尝试探索解决问题。
是不是有点怀念if...else？
我们可以使用大括号内包含立即执行函数IIFE，来达到使用if...else的目的：


![截图2](http://upload-images.jianshu.io/upload_images/4363003-1f091fcee9ad6376.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


当然，大量使用立即执行函数会造成性能上的损失。所以，考虑代码可读性上的权衡，还是有必要好好斟酌的。


## 总结
React最佳实践，想必每个团队都有自己的一套“心得”，欢迎大家与我分享。

Happy Coding!

PS: 作者[Github仓库](https://github.com/HOUCe)，欢迎通过代码各种形式交流。
