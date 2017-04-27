---
title: 一个有趣的react demo 教你如何处理树形结构
categories: js
tags: js
date: 2017-04-05
author: 侯策
---

本文介绍一个简单的react demo，通过此实例，剖析react的经典用法。并演示如何使用react思想，处理复杂的数据结构。

项目地址建议参考我的[Github仓库。](https://github.com/HOUCe/react-handle-tree-data)

## 项目简介
这个项目通过树形可操纵列表，展示了美国典型的“三权分立”政治体制，截图如下：


![三权分立展示](http://upload-images.jianshu.io/upload_images/4363003-5f0b97d7ffd6e270.gif?imageMogr2/auto-orient/strip)


如果实现这样一个view，其实并不难。我们使用原生的javascript或者jquery都能简单的实现。关键在于，这里的树形可操纵列表是可以实现定制化的。比如，今天我们剖析了“三权分立”政治体制，明天我们想实现一个公司管理层级结构，后天我们想表述一个文件系统的嵌套关系。通过这个项目，我们只需要更改数据表（配置表），便可以一键生成我们想要的结果。


## 数据设计
关于配置表，我们使用一个js对象实现。比如，“三权分立”的tree如下：

    var tree = {
        title: "American Government System",
        childNodes: [
            {title: "Legislative", childNodes: [
                {title: "Congress", childNodes: [
                    {title: "Agencies"}
                ]}
            ]},
            {title: "Executive", childNodes: [
                {title: "President", childNodes: [
                    {title: "Cabinet"},
                    {title: "Exec Office Of The President"},
                    {title: "Vice-president"},
                    {title: "Independent Agencies", childNodes: [
                        {title: "Agriculture"},
                        {title: "Commerce"},
                        {title: "Defense"},
                        {title: "Education"},
                        {title: "......"}
                    ]}
                ]}
            ]},
            {title: "Judicial", childNodes: [
                {title: "Supreme Court", childNodes: [
                    {title: "Lower Courts"}
                ]}
            ]}
        ]
    };

这样的一个数据结构中，title表示当前节点展示名称，与他同级的是一个childNodes数组。

这个数组的每一项又是一个类似“tree”结构的节点：这个节点包括了一个title表示当前节点展示名称，和一个childNodes数组。
如此递归，无穷匮也。。。

## 设计思路
设计思想上，其实上文已经说的很清楚了，就是递归。同时需要react作为视图层框架，对每一个Node节点配置以state。
好吧，我们来看一下代码：

    class Component1 extends React.Component {
        constructor(props) {
            super(props);
            this.state = {
                visible: true,
            };
        }
        toggle() {
            this.setState({visible: !this.state.visible});
        };

        render() {
            ....

            return (
                ....
            )
        }
    }

上面是一个Node节点的组件设计情况。他通过props继承了父节点传递过来的tree属性，该属性值既为上文中的tree数据结构。
这个组件的状态只有一个：visible，该值为true，表示节点展开可见；否则不可见。对应的样式集合为：style = {display: "none"}。

同时，最重要的是需要判断此Node节点是否存在子节点，既node.childNodes是否为空：

        if (this.props.node.childNodes != null) {
            ....
        }

我们首先分析此节点存在子节点的情况，既node.childNodes!= null时，

    childNodes = this.props.node.childNodes.map(function(node, index){
        return <li key={index}><Component1 node={node}/></li>
    })

    let className1 = 'togglable';
    let className2 = this.state.visible ? 'togglable-down' : 'togglable-up';
    var classNameFinal = className1 + ' ' + className2;

我们对node.childNodes的每一项子节点进行遍历，并返回childNodes变量。
同时根据组件的状态，选择合适的class进行拼接。togglable类名是必有的，同时
togglable-down/togglable-up根据状态，选择其一。

需要注意的是我们针对包含子节点的情况，进行了递归调用：<Component1 node={node}/>，能理解这一点至关重要。


## 总结
这篇文章虽然简短，但是涉及到了react思想的核心部分：组件间的通信和组件状态的选择切换。同时还涉及到了一些基本的算法，比如递归调用组件和数据结构的设计。
可谓，麻雀虽小，五脏具全。

Happy Coding!

PS: 作者[Github仓库](https://github.com/HOUCe)，欢迎通过代码各种形式交流。

