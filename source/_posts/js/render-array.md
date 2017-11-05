---
title: React Render Array 性能大乱斗
categories: js
tags: js
date: 2017-11-04
author: 侯策
---

现在关于 React 最新 v16 版本新特性的宣传、讲解已经“铺天盖地”了。你最喜欢哪一个 new feature？
截至目前，组件构建方式已经琳琅满目。那么，你考虑过他们的性能对比吗？这篇文章，聚焦其中一个小细节，进行对比，望读者参考的同时，期待大神斧正。

## 从 React.PureComponent 说起
先上结论：在我们的测试当中，使用 React.PureComponent 能够提升 30% JavaScript 执行效率。测试场景是反复操作数组，这个“反复操作”有所讲究，我们计划持续不断地改变数组的某一项（而不是整个数组的大范围变动）。

线上参考地址： [请点击这里](https://react-array-perf.now.sh/)

那么这样的场景，作为开发者有必要研究吗？如果你的应用并不涉及到高频率的更新数组某几项，那么大可不必在意这些性能的微妙差别。但是如果存在一些“实时更新”的场景，比如：

- 用户输入改变数组（点赞者显示）；
- 轮询（股票实时）；
- 推更新（比赛比分实时播报）；

那么就需要进行考虑。我们定义：changedItems.length / array.length 比例越小，本文所涉及的性能优化越应该实施，即越有必要使用 React.PureComponent。

## 代码和性能测试
在使用 React 开发时，相信很多开发者在搭配函数式的状态管理框架 Redux 使用。Redux reducers 作为纯函数的同时，也要保证 state 的不可变性，在我们的场景中，也就是说在相关 action 被触发时，需要返回一个新的数组。

    const users = (state, action) => {
      if (action.type === 'CHANGE_USER_1') {
        return [action.payload, ...state.slice(1)]
      }
      return state
    }
    
如上代码，当 CHANGE_USER_1 时，我们对数组的第一项进行更新，使用 slice 方法，不改变原数组的同时返回新的数组。

我们设想所有的 users 数组被 Users 函数式组件渲染：

    import User from './User'
    const Users = ({users}) =>
      <div>
        {
          users.map(user => <User {...user} />
        }
      </div>

问题的关键在于：users 数组作为 props 出现，当数组中的**第 K 项**改变时，所有的 <User> 组件都会进行 reconciliation 的过程，即使**非 K 项**并没有发生变化。


这时候，我们可以引入 React.PureComponent，它通过浅对比规避了不必要的更新过程。即使浅对比自身也有计算成本，但是一般情况下这都不值一提。


以上内容其实已经“老生常谈”了，下面直接进入代码和性能测试环节。

我们渲染了一个有 200 项的数组：

    const arraySize = 200;
    const getUsers = () =>
      Array(arraySize)
        .fill(1)
        .map((_, index) => ({
          name: 'John Doe',
          hobby: 'Painting',
          age: index === 0 ? Math.random() * 100 : 50
        }));
        
注意在 getUsers 方法中，关于 age 属性我们做了判断，保证每次调用时，getUsers 返回的数组只有第一项的 age 属性不同。
这个数组将会触发 400 次 re-renders 过程，并且每一次只改变数组第一项的一个属性(age):
 
      const repeats = 400;
      componentDidUpdate() {
        ++this.renderCount;
        this.dt += performance.now() - this.startTime;
        if (this.renderCount % repeats === 0) {
          if (this.componentUnderTestIndex > -1) {
            this.dts[componentsToTest[this.componentUnderTestIndex]] = this.dt;
            console.log(
              'dt',
              componentsToTest[this.componentUnderTestIndex],
              this.dt
            );
          }
          ++this.componentUnderTestIndex;
          this.dt = 0;
          this.componentUnderTest = componentsToTest[this.componentUnderTestIndex];
        }
        if (this.componentUnderTest) {
          setTimeout(() => {
            this.startTime = performance.now();
            this.setState({ users: getUsers() });
          }, 0);
        } else {
          alert(`
            Render Performance ArraySize: ${arraySize} Repeats: ${repeats}
            Functional: ${Math.round(this.dts.Functional)} ms
            PureComponent: ${Math.round(this.dts.PureComponent)} ms
            Component: ${Math.round(this.dts.Component)} ms
          `);
        }
      }

为此，我们采用三种方式设计 <User> 组件。
### 函数式方式

    export const Functional = ({ name, age, hobby }) => (
      <div>
        <span>{name}</span>
        <span>{age}</span>
        <span>{hobby}</span>
      </div>
    );

### PureComponent 方式

    export class PureComponent extends React.PureComponent {
      render() {
        const { name, age, hobby } = this.props;
        return (
          <div>
            <span>{name}</span>
            <span>{age}</span>
            <span>{hobby}</span>
          </div>
        );
      }
    }

### 经典 class 方式

    export class Component extends React.Component {
      render() {
        const { name, age, hobby } = this.props;
        return (
          <div>
            <span>{name}</span>
            <span>{age}</span>
            <span>{hobby}</span>
          </div>
        );
      }
    }

同时，在不同的浏览器环境下，我得出：

- Firefox 下，PureComponent 收益 30%；
- Safari 下，PureComponent 收益 6%；
- Chrome 下，PureComponent 收益 15%；

测试硬件环境：


![机器](http://upload-images.jianshu.io/upload_images/4363003-b4ad265d8e72d92c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最终结果：

![](http://upload-images.jianshu.io/upload_images/4363003-e5bf30012d678fa7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



最后，送给大家鲁迅先生的一句话：

> “Early optimization is the root of all evil” - 鲁迅