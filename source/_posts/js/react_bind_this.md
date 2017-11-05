在 Javascript 语言中，关于 this 这个关键字的行为一直以来困扰着一代又一代初级开发者。但是请别误会，这篇文章并不会对 this 的基本特征进行全方位讲解，因为这些内容都可以在各种前端书籍中找到答案。我试图结合 React 事件处理函数对 this 的绑定演化史，谈一谈这个框架以及 javascript 语言在这一细节上的不断进步和完善。同时对比 this 绑定的不同方案，让大家对 React 、ES next 有一个更清晰的认识。

React 处理 this 上下文环境已经有至少五年历史了。期间方案辈出，我们先来总结一下。

## 方法一：React.createClass自动绑定
React 中创建组件的方式已经很多，比较古老的诸如 React.createClass 应该很多人并不陌生。当然，从 React 0.13 开始，可以使用 ES6 Class 代替 React.createClass 了，这应该是今后推荐的方法。但是需要知道，React.createClass 创建的组件内，可以自动绑定 this。也就是说，this 这个关键字会自动绑定在组件实例上面。
    
    // This magically works with React.createClass
    // because `this` is bound for you.
    onChange = {this.handleChange}

当然很遗憾，截至目前，官方已经明确指出更推荐使用 class 声明组件或 functional 无状态组件：

> Later, classes were added to the language as part of ES2015, so we added the ability to create React components using JavaScript classes. Along with functional components, JavaScript classes are now the preferred way to create components in React.
For your existing createClass components, we recommend that you migrate them to JavaScript classes. 

我认为，这其实是 React 框架本身的自我完善和准确迎合未来，是框架和语言发展的大势所趋。


## 方法二：渲染时绑定
通过前文，我们知道传统的组件创建方式不会有 this 绑定的困扰。接下来，我们假定所有的组件都采取 ES6 classes 方式声明。这种情况下，this 无法自动绑定。一个常见的解决方案便是：

    onChange = {this.handleChange.bind(this)}

这种方法简明扼要，但是有一个潜在的性能问题：当组件每次重新渲染时，都会有一个新的函数创建。这听上去貌似是一个很大的问题，但是其实在真正的开发场景中因此引发的性能问题往往不值一提（除非是大型组件消费类应用或游戏）。


## 方法三：箭头函数绑定
这种方法其实和第二种类似，拜 ES6 箭头函数所赐，我们可以隐式绑定 this：

    onChange = {e => this.handleChange(e)}
    
当然，它同样存在潜在的性能问题。
下面将要介绍的两种方法，可以有效规避此问题，请继续关注。


## 方法四：Constructor 内绑定
constructor方法是类的默认方法，通过new命令生成对象实例时，自动调用该方法。
所以我们可以：

    constructor(props) {
        super(props);
        this.handleChange = this.handleChange.bind(this);
    }
    
这种方式往往被推荐为“最佳实践”，也是笔者最为常用的方法。
但是就个人习惯而言，我认为与前两种方法相比，constructor 内绑定在可读性和可维护性上也许有些欠缺。
同时，我们知道在 constructor 声明的方法不会存在实例的原型上，而落于实例本身的方法。每个实例都有同样一个 handleChange 方法，本身也是一种重复和浪费。

如果你对 ES next 一直抱有开放的思想，且能够使用 stage-2 的特性，不妨尝试一下最后一种方案。


## 方法五：Class 属性中使用 = 和箭头函数
这个方法依赖于 ES next 的新特性，请参考[tc 39: Public Class Fields](https://tc39.github.io/proposal-class-public-fields/)

    handleChange = () => {
          // call this function from render 
          // and this.whatever in here works fine.
    };

我们来总结一下这种方式的优点：

- 使用箭头函数，有效绑定了 this；
- 没有第二种方法和第三种方法的潜在性能问题；
- 避免了方法四的组件实例重复问题；
- 我们可以直接从 ES5 createClass 重构。


## 总结
本文在对比 React 绑定 this 的五种方法同时，我们也由远及近了解了 javascript 语言的发展：从 ES5 的 bind， 到 ES6 的箭头函数，再到 ES next 对 class 的改进。React 作为蓬勃发展的框架也同样在与时具进，不断完善。

最后，我们通过这张图片来完整回顾：


本文参考了 Cory House 的文章：[5 Approaches for Handling `this`](https://medium.freecodecamp.com/react-binding-patterns-5-approaches-for-handling-this-92c651b5af56)，并在此基础上进行延伸。









