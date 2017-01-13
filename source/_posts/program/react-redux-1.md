---
title: 一个简单的react+redux工程实例
categories: program
tags: program
date: 2016-11-26
author: 侯策
---

最近一直在学习研究react.js。前前后后做了几次分享。并在我参与的百度经验私信项目也使用了这套技术栈 ，目前也进入了最后提测上线阶段。

学习过程期间，感觉react+redux初级DEMO不多，社区上多是用烂了的todolist教程，未免乏味。
这篇文章主要实现一个简单的例子（并不是百度经验私信项目），难度不大，但是贯穿了react+redux基本思想。

他将会是一个连续教程，这只是第一篇，不涉及redux中间件，redux处理异步等内容，也不涉及react性能优化，不可变数据immutable.js等内容。读者可以根据自己的层次来选读。但这些不涉及到的内容，都会随着这个demo复杂度一步一步提升，在后续章节有分析和使用。
另外，他并不适合初学者阅读，如果你还不太清楚react.js、ES6、redux相关知识，建议先做了解。

<!-- more -->

## 简介
百度经验个人中心（WAP端）是经验流量较多的页面群，其中的个人定制页面是重要的页面之一，请用手机[点击这里](http://jingyan.baidu.com/user/nuc/feed)查看效果，页面的截图如下：

![](/bimg/c1.png)

页面大体上功能一目了然。主要分为两大块：
1）可以在『选择分类』区块中选择自己喜欢的经验分类条目来订阅。在该区块中，我们可以点击『换一换』按钮来切换分类条目。
2）已选结果会再『已选分类』区块里面展示。在『已选分类』区块里，我们可以点击相关经验分类条目来取消订阅。

![](/bimg/c5.png)

在现在线上版本中，我们采用了传统的操作DOM方式（zepto类库）来实现这一系列交互。顺便提一下，对于前端不断更迭的框架或者技术，本人的态度是不盲目跟风吹捧。建议理性地看待前端技术，重要的是学习新框架、新技术的思想。没有必要大刀阔斧地采用新技术框架重构。一定要分清适合不适合以及合理不合理。窃以为，其实这个页面采用传统技术来开发也是不错的。但是使用react，又是一种全新的思想。孰优孰劣，可以在结尾处大家自己总结。
好了，废话不多说，我们马上进入正题。

## 架构
整体目录如截图，为了和我们线上代码保持一致，我采用了fisp来做工程化组织。现在社区上大多都是采用webpack，其实这些工具用哪个都一样，解决的问题也都类似，这里不再展开。即使你不懂FIS，可不妨碍继续阅读。

![](/bimg/c2.png)

### app/ app文件夹下

action.es定义了页面交互中dispatch的所有action；

![](/bimg/c6.png)

app.jsx是页面入口脚本；

![](/bimg/c8.png)

component.jsx定义了页面的组件；

![](/bimg/c7.png)

reducer.jsx接收action，该文件定义了所有用到的reducer。

### lib + lib-nomod 文件夹下
这两个文件夹是我们要用的框架源码，比如react.js+redux.js等等；
该项目用到的是react15.3.1版本未压缩版，
这个版本比较稳定。采用未压缩版的原因是想使用react addons 的perf，在后续章节中，会有性能优化部分的分析；
我们知道，react和redux其实独立存在，我们使用流行的react-redux.js库来实现两者的连接。

### 其他相关文件
其他还有 fis-conf.js文件：这是用来做fis配置的，比如打包规则，发布规则，编译配置等；
同时，我们配置了babel来编译es6和jsx等，还配置了autoprefixer；
server.conf是fis的附属文件，用来做数据mock；
build.sh和BCLOUD是上线脚本相关，这里我们并不上线，只是学习react的用法。


## 页面数据
我们部门后端是PHP，采用Smarty模板。这个页面会在请求时同步给出一些数据，比如用户信息等。项目里，我们的同步数据如截图：

![](/bimg/c3.png)

我们关心selectList和likedList：
1）likedList给出当前用户已经选则的订阅分类条目；
2）selectList给出所有可选的分类条目，一共从1-127，数据格式如上。
我们的脚本里使用F.context('likedList')，和F.context('selectList')取出相应同步数据，用于页面初始化展示。
F.context()没有什么可神奇的，其实就是一个全局的数据中心，实现就是在window.F下存取数据。


## 具体实现
说了这么多，终于可以进入具体代码层面了。如果上边的内容你似懂非懂，也没有关系。因为涉及了一些项目组织上的内容。下边的内容，就是具体的代码分析了。


### 数据设计
react+redux开发前端的思想是页面由数据驱动。
上边已经分析到，这里再赘述一下。我们的页面主要由两种数据：
1）一个是F.context('selectList')，我们姑且叫做选择池数据；
2）另一个是已选数据F.context('likedList')。
这两处数据初始由reducer拿到，设置为容器组件的初始状态，并由容器组件传递给相应展示组件。

### 组件设计
组件设计如下截图：

![](/bimg/c4.jpg)

按照react-redux思想，组件分为
1）容器组件，负责接收数据；
2）展示（木偶）组件负责向上接收数据，根据数据展现组件UI。

其实很明显，我们主要就是两个展示组件：
1）一个SelectedBlock，
2）一个SelectListBlock。
前者负责展示用户已选已订阅内容，后者展示页面选择池可供选择的内容。他们一起被套在叫做DemoApp的父组件里面。

有了以上划分，我们可以想到：
SelectedBlock组件需要关心已选数据；
SelectListBlock则选择池数据和已选数据都需要关心。你可能会问『SelectListBlock
关心选择池数据不就够了吗？』但是，产品经理要求在选择池里，当渲染用户已选条目时，需要样式置灰，并且在点击已选分类条目时不在触发action。所以选择池SelectListBlock组件也要依赖已选数据，进而来做出相应的变化。

这两项数据有react-redux派分给容器组件，并由容器组件按需分给展示组件；也就是说：F.context('likedList')向下传递给SelectedBlock；F.context('likedList')和F.context('selectList')向下传递给SelectListBlock

有了以上基础，我们看最外层的DemoApp组件全部代码：

    class DemoApp extends React.Component {
        constructor(props) {
            super(props);
        }
        render() {
            const { dispatch } = this.props;
            return (
                <div>
                    <SelectedBlock 
                        likedList={this.props.likedList} 
                        onDeleteLikeItem={(item)=>{dispatch(action.deleteLikeItem(item))}}>
                    </SelectedBlock>

                    <SelectListBlock 
                        selectList={this.props.selectList} 
                        likedList={this.props.likedList} 
                        onAddLikeItem={(index, item) =>{dispatch(action.addLikeItem(index, item))}}>
                    </SelectListBlock>
                </div>
            )
        }
    }

我们看他的render()部分，很明显，他平行嵌套了
1）SelectedBlock组件，并把likedList数据作为属性向其传递；
2）同时，包含了SelectListBlock，并把selectList，likedList数据作为属性向其传递。

那么SelectedBlock设计如下：

    class SelectedBlock extends React.Component {
        constructor(props) {
            super(props);
        }
        deleteItem(event, index) {
            this.props.onDeleteLikeItem(index);
        }
        render() {
            let likedList = this.props.likedList;
            let likedListArray = [];
            let likedListKey = Object.keys(likedList);
            likedListKey.forEach(function(index){
                likedListArray.push(likedList[index]);
            })
            return (
                <div>
                    <h2>已选分类(<em id="f-num">{likedListArray.length}</em>)</h2>
                    <div className="selected-list" style={{overflow: 'auto'}}>
                        <ul className="feed-list">
                            {
                                likedListArray.length > 0 ?
                                likedListArray.map((item, index) => {
                                    return (
                                        <li style={{position: 'relative'}}>
                                            <span>{item}</span>
                                            <a style={deleteIconStyle} 
                                                onClick={event=>{this.deleteItem(event, likedListKey[index])}}>
                                            </a>
                                        </li>
                                    )
                                })
                                :
                                <li className="empty-list">还没有任何订阅<br />请从下方选择订阅</li>
                            }
                        </ul>
                    </div>
                </div>
            )
        }
    }

因为likedList我们获取到的是个对象，我们需要使用：

    let likedList = this.props.likedList;
    let likedListArray = [];
    let likedListKey = Object.keys(likedList);
    likedListKey.forEach(function(index){
        likedListArray.push(likedList[index]);
    })

来把likedList转换成likedListArray数组，在render()里面，直接使用map循环输出；
当用户删除某一条目时，触发deleteItem(event, index)方法，该方法向上传递。并在DemoApp父组件中，触发相应action。这个删除过程并不是一个组件内行为，因为这个action会使得已选数据发生变化，进而影响SelectListBlock组件。所以一系列逻辑需要在组件外，即reducer中处理，处理完后重置已选数据，进而页面更新。


SelectListBlock组件也很好理解：

    class SelectListBlock extends React.Component {
        constructor(props) {
            super(props);
            this.state = {
                flag: 0
            }
        }
        onChangeGroup(event) {
            event.stopPropagation();
            let flagNow = this.state.flag;
            if (flagNow == 117) {
                this.setState({
                    flag: 0
                });
            }
            else {
                this.setState({
                    flag: flagNow + 9
                });
            }
        }
        onSelectItem(index, item) {
            var likedList = this.props.likedList;
            var likedListKey = Object.keys(likedList);
            if ( likedListKey.indexOf(index.toString()) >= 0 ) {
                return;
            }
            this.props.onAddLikeItem(index, item);
        }
        render() {
            let selectListArray = [];
            for (var i in this.props.selectList) {
                selectListArray.push(this.props.selectList[i])
            }
            let likedList = this.props.likedList;
            let likedListKey = Object.keys(likedList);
            return (
                <div>
                    <h2 className="clr">
                        <span onClick={event=>{this.onChangeGroup(event)}}>换一换</span>
                    选择分类</h2>
                    <ul className="feed-list clr">
                        {
                            selectListArray.slice(this.state.flag, this.state.flag+9).map((item, index)=>{
                                return (
                                    <li onClick={event=>{this.onSelectItem((index + this.state.flag), item)}} 
                                        key={index + this.state.flag}>
                                        {(likedListKey.indexOf((index + this.state.flag).toString()) >= 0 ?
                                            <span className='disable'>{item}</span>
                                            :
                                            <span>{item}</span>
                                        )}
                                    </li>
                                )
                            })
                        }
                    </ul>
                </div>
            )
        }
    }

我们首先把后端打过来的127条可供选择的数据转换为selectListArray数组。这127个分类内容都对应一个index（1-127）。之后我们获得用户已经选择喜欢条目的index，将这些index组成likedListKey数组，用于存储当前用户已选条目。在render()时候，因为页面一次只展示9项待选项，所以我们把selectListArray用slice方法按顺序切割出来9项输出。点击『换一换』按钮时，触发onChangeGroup（）方法，这个方法是个组件内方法，他负责将slice参数+9，当到127（一共127项分类）时，还原回0。如下：

    onChangeGroup(event) {
        event.stopPropagation();
        let flagNow = this.state.flag;
        if (flagNow == 117) {
            this.setState({
                flag: 0
            });
        }
        else {
            this.setState({
                flag: flagNow + 9
            });
        }
    }

我们知道，点击『换一换』触发的onChangeGroup方法改变flag时候，因为flag为该组件内部state。他的变化，将会引起该组件重新render()，所以数据池就会毫无压力的切换了。
同时，我们给数据池里每一项分类都绑定onSelectItem方法，该方法会向上传递给父组件，由父组件发出相应action。因为这个动作将会改变已选数据，影响平行的SelectedBlock组件。需要在reducer中处理。
关于onSelectItem参数做的调整，我想聪明的你一定看得出来是因为什么。其实是因为map遍历时每一项index都是依次递增的，永远是从1-9。而当我们传递用户选中的参数时，要传递的是要添加的某一项所对应的index时，他会是1-127某一数字。


### action设计
有了以上组件的设计，很明显我们需要定义两个action：
1）第一个是添加某一条目到已选分类

    export const ADD_LIKE_ITEM = 'ADD_LIKE_ITEM';

对应action creator:

    export function addLikeItem (index, item) {
        return {
            type: ADD_LIKE_ITEM,
            obj: {
                index: index,
                item: item
            }
        }
    }

返回action对象，包括type命名为ADD_LIKE_ITEM和负载数据：条目名item及其index。

另一个是在已选分类删除某一条目：

    export const DELETE_LIKE_ITEM = 'DELETE_LIKE_ITEM';

对应action creator:

    export function deleteLikeItem (index) {
        return {
            type: DELETE_LIKE_ITEM,
            index
        }
    }

返回action对象，包括type和负载数据。
到此为止，action脚本只需要定义action，不需要进一步处理，对所有action的处理都会由reducer接受。


### reducer设计
再次强调reducer是一个纯函数，他接受两个参数，一个是state，一个是action；并对相应的action，返回一个新的state，从而促使页面里订阅相关state的组件再次render（）；
我们把同步模板数据initialLikeBlockState设为初始state

    var initialLikeBlockState = F.context('likedList');
    function likeBlockReducer (state = initialLikeBlockState, action) {
        switch (action.type) {
            case actionType.ADD_LIKE_ITEM: {
                var addIndex = action.obj.index;
                var newLikedList = Object.assign({}, state, {
                    [addIndex]: action.obj.item
                })
                return newLikedList;
            }
            case actionType.DELETE_LIKE_ITEM: {
                var newLikedList =  {};
                for (var key in state) {  
                    var val = state[key];  
                    newLikedList[key] = val;  
                }  
                var index = action.index;
                delete newLikedList[index];
                return newLikedList;
            }
            default: {
                return state;
            }
        }
    }

当匹配ADD_LIKE_ITEM action时，我们把当前的state和action带来的数据（item，index）进行merge，从而return 一个新的已选数据状态，即添加了新分类item的state；
当匹配DELETE_LIKE_ITEM action时，我们把action负载带来要删除item的index删除掉。返回删除该条目之后的新state。



## 总结
截至目前，我们介绍了基本设计和开发思路。教程里面已经基本包含了全部代码。

### 对比线上已有代码
1）和线上的zepto实现对比完全是两种思路，经过比较，用react设计的代码代码量上有着明显的优势。
2）开发思路上，是个萝卜青菜各有所爱的问题。但是对于写惯了$()的我来说，这种全新的开发方式还是带来了极大的惊喜。
3）线上实现这一套逻辑，可能一个简单的UI交互我们需要选取很多dom元素，进行处理。整体上看，比较复杂且凌乱，不是很容易进行维护。


### 接下来...
当然，这只是第一步。后边还有更多的路要走。比如：
1）我们在选择或删除一个条目时，按道理讲真正实战上，如何给后端发异步请求并没有涉及。因此，redux异步流程并没有展现。后续章节会进一步讲解。
2）我们的数据都是后端模板通过同步的方式传递过来的，数据量也不大，结构也不复杂，因此这一章为了简单并未使用immutable.js。当然，后续章节会进一步讲解。
3）这里我并没有介绍使用redux dev tool，这真的是一个很漂亮的利器。尤其在数据复杂时候，对于调试能帮上很大作用。后面我会单独介绍一下关于这个工具的使用。
4）最后，这么简单的交互还并不会涉及页面性能的问题。在后续章节，我会构造出极端CASE进行一些边缘测试，并使用一些方法结合chrome dev tool进行性能优化，请进一步关注。




author: LucasHC