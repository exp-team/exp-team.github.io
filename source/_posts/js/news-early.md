---
title: React+Redux打造“NEWS EARLY”单页应用 一步步让你理解最前沿技术栈的真谛
categories: js
tags: js
date: 2017-03-30
author: 侯策
---

之前写过一篇文章，分享了我利用闲暇时间，使用React+Redux技术栈重构的百度某产品个人中心页面。您可以[参考这里](http://www.jianshu.com/p/8e28be0e7ab1)，或者参考[Github代码仓库地址。](https://github.com/HOUCe/react-redux-demo)
这个工程实例中，我采用了厂内的工程构建工具－FIS，并贯穿了react+redux基本思想。

今天这篇文章给大家分享一个更加复杂，但是非常有趣的一个项目-
> News Early单页应用。

我把这个项目所有代码托管在了我个人Github之中，感兴趣的读者可以跟我探讨。

最近我发现，React Redux生态圈项目活跃。但是作品质量“良莠不齐”，很多非常热门的项目不仅没有起到“布道”作用，而且在一定程度上“误导”了读者。在这篇文章里面我会有详细说明。当然，我自己也是资历浅显，水平有限。希望大神能够给与斧正。

同时通过这个项目实例和这篇文章，一步一步说明了这个项目开发细节，并且包括了优化手段等内容。希望使大家对于React技术栈，包括：Redux数据流框架＋React Router路由管理＋Webpack构建工具等，有一个更加清晰深刻的理解。


## 项目背景
在国外上学和工作期间，能畅通无阻的访问诸如：BBC，CNN，ESPN，Le Figaro等新闻媒体是一大便利，也是我个人闲暇时期一个喜好之一。
甚至外出旅游时，在酒店收看这些媒体卫视（尤其CNN）竟然也是放松休闲的一大方式。。。

当然，国内环境对于这些境外媒体显然不是太友好。
基于此，我设计开发了News Early项目。

>这个项目是一个包括：BBC，CNN，The NewYork Times等70多个国际知名媒体的即时头条新闻聚合APP。

>News Early is a simple and easy-to-use Web APP that gathers the headlines currently published on a range of news sources and blogs (70 and counting so far).

整个项目我使用了包括但不限于以下技术栈和构建工具：
1）React UI框架from Facebook；
2）JSX模版；
3）Redux数据流设计；
4）Webpack构建工具；
5）Less预处理器；
......

## 项目设计
整个Web APP的部分使用体验，我用以下GIF图示来呈现：
（请耐心等待GIF图加载）

![APP 使用截图](http://upload-images.jianshu.io/upload_images/4363003-f960f8050f049833.gif?imageMogr2/auto-orient/strip)



* 1）**页面顶部导航条**
包括：侧栏菜单开启按钮和右侧的刷新页面按钮。
* 2）**页面内容头部轮播图**
支持自动播放和手势滑动操控。
* 3）**页面主体部分**
主体部分是所对应的新闻频道的headlines头条新闻，一般有10-20个items左右。每一个item包含一张新闻图片，新闻导读（Abstract）以及新闻发布时间（publish time）。
* 4）**左侧折叠菜单栏**
功能用于新闻频道的筛选。
以Gif图截取为止，一共接入了：BBC News，BBC Sport，CNN，ESPN，Financial Times，USA Today，MTV News7家国际媒体。

因为我不是搞视觉设计的，也不是做页面交互设计的。我只是一枚码农。所以为了节省时间，整体APP的样式上，包括界面颜色等，我参考了[卖座网](http://m.maizuo.com/v4/?co=maizuo#/?_k=h7qksx)的实现。


## 项目架构和落地
下面，我为大家介绍一下整个项目的设计构成和开发细节。

###  数据流状态演示
熟悉Redux数据流框架的同学，应该对于store，dispatch，action，reducer，以及中间件等概念比较熟悉。这里不再进行讲解。
这套架构中，最重要的就是**数据流的设计。**

首先，我们先整体看一下在“切换频道”这个交互发生时，整个项目的数据流向和数据结构的演示：

![数据流动示意图](http://upload-images.jianshu.io/upload_images/4363003-df843395274f854c.gif?imageMogr2/auto-orient/strip)


### 目录结构
如图所示：

![目录结构](http://upload-images.jianshu.io/upload_images/4363003-abcbfc9811943bd2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

整个项目业务代码部分，我拆分成9个UI组件，1个全局Store，一个actions定义文件。

- app是开发目录
    - actions目录集中了全局所有的actions
    - components目录集中了全局用到的所有UI组件
    - reducers目录集中了Redux架构中的所有reducers
    - store目录定义全局唯一的store
    - style目录集中了全局所有组件的样式文件
    - main.js为全局的入口函数
- build是打包后结果目录
    - index.html是输出页面文件
    - bundle.js开发目录下脚本文件打包后的产出
    - img文件定义了APP开启时的loading图片
- node_modules相信大家不会陌生，这是依赖文件
...
其他配置文件不再一一介绍。

10个组件包括：
- appIndex: 组件容器
- billboardCarousel: 页面轮播图组件
- currentChanel: 页面headlines新闻头条组件
- homeView: 主体页面
- imagePlaceholder: 占位图组件
- loading: 加载提示组件
- navBar :顶部导航组件
- sideBar: 侧边栏组件
- routerWrap: 路由相关组件


### 骨架构建
我认为，redux之所以学习曲线陡，很大程度上就在于**数据流的贯通上。**

>“组件触发(dispatch)各种action，单向数据流流向reducer，reducer是一个纯函数(函数式编程思想)，接收处理action，返回新的数据，组件进而更新”

这一套理论并不难理解。

但是落实在工程上，尤其要结合react，那就不好做了。即使有人做出来，业务就算可以跑得通，但是相比核心思想，却是背道而驰。社区上我看过很多项目，在写法上不分青红皂白，只要能运行，胡乱设计一通，误导初学者。

比如在整个项目中，存在多个stores这种常见的问题。

**那么，为什么不建议存在多个store呢？**
答案可以在[官方FAQ](http://redux.js.org/docs/faq/StoreSetup.html)中找到。内容较多，如果英文阅读吃力，我大体翻译一下：
*熟悉Flux原始模型的读者可能了解，Flux存在多个stores，每个store都维护了不同层次的数据。这样设计的问题在于，一个store需要等待另外一个store的操作处理。我们Redux实现了切分数据层次，避免了这种情况的发生。
仅维持单个store不仅可以使用Redux DevTools，还能简化数据的持久化及深加工、精简订阅的逻辑处理。
单一store这种方式，我们不用考虑store模块的导入、 Redux应用的封装，后期支持服务器渲染也将变得更为简便。*

如果上边这段话过于抽象，难以理解的话，那就直接看我的代码实现吧。

定义全局唯一的store：

    const store = createStore(
        combineReducers({
            sideBarChange,
            contents,
            routing: routerReducer
        }),
        composeEnhancers(applyMiddleware(thunkMiddleware)),
    );

其中，我使用了redux-thunk作为中间件，用于处理异步action。这样，把异步过程放在action级别解决，对component没有影响。
另外composeEnhancers是用于使用redux devtool的设置。


容器组件构建：

    const mapStateToProps = (state) => {
        return {
            showLeftNav: state.sideBarChange.showLeftNav,
            loading: state.contents.loading,
            contents: state.contents.contents,
            currentChanel: state.contents.currentChanel
        }
    }

    var App = connect(mapStateToProps)(AppIndex);
    render(
        <Provider store={store}>
            <Router history={history}>
                <Route path="/" component={App}>
                    <Route path="home" component={HomeView}/>
                </Route>
            </Router>
        </Provider>,
        document.getElementById('app')
    );

其中，我使用了react-redux进行连接。AppIndex是整个项目唯一的容器组件。进行action的dispatch，以及向下传递props给UI组件（木偶组件）。

如果你还不理解**容器组件**和**UI组件**的区别，可以去[官方文档学习。](http://redux.js.org/docs/basics/UsageWithReact.html)这两个概念极其重要，它直接决定你是否能设计出有效且合理的组件架构。

另外，你会发现我使用了react－router进行路由管理。其实整个项目没有必要使用单页路由。这个路由管理的引入，说实话，比较鸡肋。但并不会对项目产生任何影响。我引入他的原因主要有两点。
- 第一是，后续进行二次开发，考虑到更多的产品迭代的话，使用路由管理是必须的，我们要为长远准备。
- 另一个原因就是，我从来没用用过，好吧，想尝鲜下。



### actions设计
actions当然是必不可少的，我这里选取最重要的“fetchContents”这个action creator来讨论一下。

初次进入页面时，以及左侧边栏点击选择新闻频道时，都要去拉取数据。比如，APP第一次渲染，默认加载“BBC News”新闻频道，页面主体组件在挂载完成后：

    componentDidMount() {
        //获取内容
        this.props.fetchContents('bbc-news');
    }

向上调用fetchContents方法，并逐级上传到容器组件。由容器组件进行dispatch:

    fetchContents={(source)=>{this.props.dispatch(action.fetchContents(source))}}

source表示拉取的新闻频道。此处当然是'bbc-news'。

在actions.js文件中，进行异步action的处理并拉取数据。这里，我使用了最新的**fetch API**来代替古老的XHR，并利用fetch的promise的理念，封装了一层_get方法，用于AJAX异步请求：

    const sendByGet = ({url}, dispatch) => {
    let finalUrl = url + '&apiKey=1a445a0861e'
    return fetch(finalUrl)
            .then(res => {
                if (res.status >= 200 && res.status < 300) {
                    return res.json();
                }
                return Promise.reject(new Error(res.status));
            })
    }

对应的action操作：

    export const fetchContents = (source) => {
        const url = '...';
        return (dispatch) => {
            dispatch({type: FETCH_CONTENTS_START});
            if (sessionStorage.getItem(source)) {
                console.log('get from sessionStorage');
                let articles = JSON.parse(sessionStorage.getItem(source));
                dispatch({type: FETCH_CONTENTS_SUCCESS, contents: Object.assign(articles, {currentChanel: source.toUpperCase()})})
            }
            else {
                sendByGet({url}, dispatch)
                .then((json) => {
                    if (json.status === 'ok') {
                        sessionStorage.setItem(source, JSON.stringify(json.articles)); 
                        return dispatch({type: FETCH_CONTENTS_SUCCESS, contents: Object.assign(json.articles, {currentChanel: source.toUpperCase()})})
                    }
                    return Promise.reject(new Error('FETCH_CONTENTS_SUCCESS failure'));
                })
                .catch((error) => {
                    return Promise.reject(error)
                })
            }
        }
    }



### 请求优化
我们知道，这些异步请求的访问速度是很慢的。因此，我采用了几种方法来进行优化。

- 第一个方法就是加载时的loading美化。
我使用了[来自网络的图片](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1489644681768&di=2b78e7baf0749987d6bd1879b4d9f9f8&imgtype=0&src=http%3A%2F%2Fimg.zcool.cn%2Fcommunity%2F02dd4155c1c09a000001a5f0498dd6.gif)占位。
当我把控制台中网络环境人为的模拟为3G时，页面效果如下：
（请耐心等待GIF图加载）

![加载占为图](http://upload-images.jianshu.io/upload_images/4363003-c9da38782a2473d6.gif?imageMogr2/auto-orient/strip)


原谅我使用了这么粉嫩少女的加载图。。。

- 第二个方法其实是一个trick，我的全局图片在初始状态时opacity设置为0，在onload事件触发时设置一个fadeIn的效果：

      <img ref="image" src={imgSrc} onLoad=
      {this.handleImageLoaded.bind(this)}/>

      handleImageLoaded() {
          this.refs['image'].style.opacity = 1;
      }

这样的一个小技巧最初来自Facebook对用户体验的研究。如果您对此有兴趣，可以在我的[另外一篇文章](http://www.jianshu.com/p/2f3bc2598dc5)中找到相关内容。


- Web Storage来进行优化 
因为各大新闻媒体的headlines发布更新是不定时的，这个时间间隔可能较长。而我考虑到用户使用这个Web APP一般都是在碎片时间中。因此我采用了sessionStorage进行缓存内容。不要问我为什么不使用localStorage...，如果你存在疑问，建议对于Web Storage的特性再去回炉重修一下。

具体实现方式就是在发送请求时判断sessionStorage是否已经存在此新闻媒体（比如bbc）的数据。如果存在就使用缓存。否则就去进行AJAX请求，请求成功的回调函数里进行缓存的种植。
代码部分如下：

    if (sessionStorage.getItem(source)) {
        console.log('get from sessionStorage');
        let articles = JSON.parse(sessionStorage.getItem(source));
        dispatch({type: FETCH_CONTENTS_SUCCESS, contents: Object.assign(articles, {currentChanel: source.toUpperCase()})})
    }
    else {
        sendByGet({url}, dispatch)
        .then((json) => {
            if (json.status === 'ok') {
                sessionStorage.setItem(source, JSON.stringify(json.articles)); 
                return dispatch({type: FETCH_CONTENTS_SUCCESS, contents: Object.assign(json.articles, {currentChanel: source.toUpperCase()})})
            }
            return Promise.reject(new Error('FETCH_CONTENTS_SUCCESS failure'));
        })
        .catch((error) => {
            return Promise.reject(error)
        })
    }

当然，有种植缓存，就要有清除缓存。这个按钮我设置在里navBar组件的最右侧：

    const CLEAR_SESSIONSTORAGE = 'CLEAR_SESSIONSTORAGE';
    export const refresh = () => {
        sessionStorage.clear();
        return dispatch => dispatch({type: CLEAR_SESSIONSTORAGE});
    }



### 其他细节
为了使用先进的构建工具的需求，我使用了node最新版本。但是因为工作业务的需要，又要同时保留低版本node环境。为此，我使用了：n这个利器进行node版本管理。

同时，我使用了webPack一系列强大开发功能和构建功能。包括但不限于：
- 热更新
- Less编译插件
- 服务器构建，使用了8088端口
- jsx,es6编译
- 打包发布
- 彩色日志

...等等，但是我可不是webpack专家。在狼厂，当然使用更多的是FIS构建工具。关于FIS和webpack的比较，我的网红同事@颜大神有过[探索](https://www.zhihu.com/question/50829160)。


## 总结
这篇文章涉及到了较为前沿的前端开发技术栈。包括了React框架，Redux数据流框架以及函数式编程、异步action中间件，fetch异步请求，webpack配置等等。也无形中涉及到了一些成熟产品的设计理念思路。当然这个项目还远没有成熟。在代码仓库中，我会不间断进行更新。
希望本文对大家在各个维度都有所启发。也恳请业界大牛不吝赐教，进行斧正。

最后想跟大家谈一下对于框架和前端学习的一些感受。我记得我刚开始工作，在初次接触前端时，是使用ionic，即Angular框架和phoneGap开发hybrid移动APP。当时我是完全懵b的，只是感觉比利时同事用的超high，6到飞起。每次他用浓重的比利时口音法语给我讲解时，我听的云里雾里，不知所以。

现在想想当时那么菜的原因还是在于自己的JS基础不够牢固。当你面对迅速更新换代的前端技术踟蹰茫然时，唯一的捷径就是从基础抓起，从JS原型原型链，this，执行环境上下文等等看起。

觉得前端知识有欠缺的读者们，欢迎follow我。最近我会带大家“重读”JS经典书籍，以code demo的形式提炼知识点，并会同步到博客和个人Github上。

Happying code!





