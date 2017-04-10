---
title: React大会上我们能学到什么？React + ES next = ♥
categories: js
tags: js
date: 2017-04-14
author: 侯策
---

React Conf 2017在加利福尼亚州的圣克拉拉万豪酒店圆满落幕，这已经是Facebook举办的第三届React官方大会了。
虽然不能参会，但是作为前端开发者，我们当然不能错过这个绝佳的学习契机。

笔者利用清明假期，参看了YouTube上关于这次大会的记录。一共34个精彩演讲，对应34个视频。获益匪浅。

这里，将会作为一个系列，对其中的几篇演讲进行翻译和分析。并辅助以code demo，帮助大家理解。
欢迎关注我的[简书](http://www.jianshu.com/u/452568260db5)或[掘金](https://juejin.im/user/58305395a22b9d006b83d214)账号，也欢迎在Github上floow，相信你一定也会有所收获。

今天为大家介绍的是Ben Ilegbodu的主题：React + ES next = ♥ 

Ben Ilegbodu是“为数不多的”参会有色程序员，黑人程序员如同女性程序员一样凤毛麟角，同样，本次分享主题也很有爱，很有营养。如果你不了解React也不要紧，因为这次讲的是ES6、ES7在React中的应用。所以，其实是对下一代ES的普及和介绍。

本文将以Destructuring、Spread Opetator、Arrow Function、Promises、Async Functions展开。并通过nodeJS实现一个兼具前端和后端的小型“评论／留言发布阅读系统”。

## 实现预览
如图，我们实现了如下的页面。当然，样式是极其简陋的。作为“粗糙”的程序员，实在懒得在页面上花时间。
我们可以在输入框内输入姓名和留言内容。并点击按钮提交。后台使用nodeJS express框架，实现对文件的读写更新。

    app.post('/api/comments', function(req, res) {
        fs.readFile(COMMENTS_FILE, function(err, data) {
            if (err) {
                console.error(err);
                process.exit(1);
            }
            var comments = JSON.parse(data);
            var newComment = {
                id: Date.now(),
                author: req.body.author,
                text: req.body.text,
            };
            comments.push(newComment);
            fs.writeFile(COMMENTS_FILE, JSON.stringify(comments, null, 4), function(err) {
                if (err) {
                    console.error(err);
                    process.exit(1);
                }
                res.json(comments);
            });
        });
    });

这是nodeJS的后端代码，是为了介绍实现的这个project前后端协作。
接下来我们继续回到演讲。

## 解构Destructuring
在我们的项目中，最初版存在这样的一段代码：

    _handleCommentSubmit(comment) {
        let comments = this.state.comments;
        ...
        // remaining code
    }

请务必记住这个_handleCommentSubmit函数，接下来的全文都是围绕他展开，并进行一步步拓展。
这个函数的逻辑是对新提交的comment进行处理，首先他需要从state中读取现有的comments数组。
在使用解构赋值的情况下，我们重构为：

    _handleCommentSubmit(comment) {
        let {comments} = this.state;
        ...
        // remaining code
    }

也许这还看不出来解构到底有什么作用，但是在逻辑多的时候，他是很有必要的。比如：

    let author = this.state.author;
    let text = this.state.text;

就可以写为：

    let {author, text} = this.state;

如果需要改变变量名时，就可以从：

    let authorName = this.state.author;
    let fullText = this.state.text;
    
改为：

    let {author: authorName, text: fullText} = this.state;

再举一个例子，在function component情况下：

    function MyComponent(props) {
        return (
            <div style={props.style}>{props.children}</div>
        )
    }
    <MyComponent style="dark">Stateless function!</MyComponent>

我们可以改写为：

    function MyComponent({children, style}) {
        return (
            <div style={style}>{children}</div>
        )
    }
    <MyComponent style="dark">Stateless function!</MyComponent>


## 展开符Spread Opetator
还记得上面那个_handleCommentSubmit函数吗？接下来我们要进行扩充。首先我们将新提交的comment（即函数参数），添加一个时间戳作为id。接下来，
我们拿到旧的state.comments之后，就要将新的comment（包含id）加入到state.comments当中。

初版做法是：

    _handleCommentSubmit(comment) {
        let {comments} = this.state;
        let newComment = comment;
        
        newComment.id = Date.now();

        let newComments = comments.concat([newComment]);

        ...
        // setState + ajax stuffs
    }

在使用展开符后，我们可以重构为：

    _handleCommentSubmit(comment) {
        let {comments} = this.state;
        let newComment = {...comment, id: Date.now()};
        let newComments = [...comments, newComment];
        ...
        // setState + ajax stuffs
    }

当然，展开符还有很多其他benefits，比如，平时我们可以使用Math.max方法对数组求出最大值：

    var arrayOfValue = [33, 2, 9];
    var maxValueFromArray = Math.max.apply(null, arrayOfValue)

这个思路利用了apply接受一个数组作为函数参数的特性。
使用展开符，我们就可以：

    var arrayOfValue = [33, 2, 9];
    var maxValueFromArray = Math.max(...arrayOfValue);

同理，我们可以这样扩充一个数组：

    let values = [2, 3, 4];
    let verbose = [1, ...values, 5];

当然，以上都是对数组的展开。对对象属性的展开符的使用，也已经到了Stage3阶段。今后，我们可以这样写代码：

    let warriors = {Steph: 95, Klay: 82, Draymond: 79};
    let newWarriors = {
        ...warriors,
        Kevin: 97
    }

而不必再使用Object.assign进行对象的扩展。

## 箭头函数Arrow Function
箭头函数带来的好处无疑是对this的绑定。
现在再回到我们的_handleCommentSubmit函数，做完了初始化工作，我们需要将新的comment通过ajax，异步发送给后端。在成功的回调函数中，进行setState处理：

    $.ajax({
        url: this.props.url,
        data: comment,
        success: function(resJson) {
            this.setState({comments: resJson})
        }.bind(this)
    })

有经验的同学注意到了我使用bind更改this指向的处理。在ES6箭头函数下，我们可以直接：

    $.ajax({
        url: this.props.url,
        data: comment,
        success: (resJson) => {
            this.setState({comments: resJson})
        }
    })

我们再展开看看箭头函数的几种用法：
1）匿名函数，单参数情况，比如实现一个数组的各项平方：

    let squares = [1, 2, 3].map(value => value * value);

2) 匿名函数，多参数情况，比如实现一个数组的各项累加：

    let sum = [9, 8, 7].reduce((prev, value) => prev + value, 0)

这时候，需要把参数放在括号之中。

3) 非匿名函数，无返回值情况：

    const alertUser = (message) => {
        alert(message)
    }

这时候，函数体用花括号围起来。

4）更复杂的情况，比如上面我们用到过的：
    
    const MyComponent = ({children, style}) => (
        <div style={style}>{children}</div>
    )



## Promises
上面的代码我们使用了jquery的ajax方法发送异步请求，这可能在某些情况下出现“回调地狱”的情况。为此，我们使用基于Promise的fetch方法，进行重构：

    _handleCommentSubmit(comment) {
        // ...
        fetch(this.props.url, {
            method: 'POST',
            body: Json.stringify(comment)
        })
            .then((res)=>res.json())
            .then((resJson)=>{
                this.setState({comments: resJson})
            })
            .catch((ex)=>{
                console.error(this.props.url, ex)
            })
    }

当然，我们可以把上述代码做的更抽象：

    const fetchJson = (path, options) => {
        fetch(`${DOMAIN}${path}`, options)
            .then((res)=>res.json())
    }

我们在拉取评论时，就可以：

    const fetchComments = () => fetchJson('api/comments')

基于promises的fetch，让异步请求变的灵活可靠。

同时，我再安利一个基于promises的sleep函数：

    const sleep = (delay = 0) => {
        new Promise((resolve)=>{
            setTimeout(resolve, delay)
        })
    }

    sleep(3000)
        .then(()=>getUniqueCommentAuthors())
        .then((uniqueAuthors)=>{this.state({uniqueAuthors})})

回到我们的Project中，在后端nodeJS实现里，我们把所有的评论存在data/comments.json文件当中。
在渲染视图时，就不可避免地要进行对data/comments.json文件读取，这个过程也应该是异步完成的：

    const readFile = (filePath) => (
        new Promise((resolve, reject)=>{
            fs.readFile(filePath, (err, data)=>{
                if (err) {reject(err)}
                resolve(data)
            })
        })
    )

    readFile('data/comments.json')
        .then((data)=>console.log('Here is the data', data))
        .catch((ex)=>console.log('Arg!', ex))

## Async Functions
当然，Promises实现也有缺点。关于Promises和Async的对比，具体可以看我的一篇文章：[]()

更先进的方式，就是使用Async，回到我们的_handleCommentSubmit方法，我们可以重构为：

    async _handleCommentSubmit(comment) {
        try {
            let res = await fetch(this.props.url, {
                method: 'POST',
                body: JSON.stringify(comment)
            });
            newComments = await res.json();
        }
        catch (ex) {
            console.error(this.props.url, ex);
            newComments = comments;
        }

        this.setState({comments: newComments});
    }


## 总结
这篇演讲生动形象地阐释了React是怎样与ES next融合天衣无缝的。不论是React也好，还是ES也好，其实笔者认为说到底，都是为了更大限度地解放生产力。

欢迎读者与我交流，有任何问题可以留言。今后几天，将有更多新鲜的react conf视频翻译奉献给大家。



PS: 作者[Github仓库](https://github.com/HOUCe)，欢迎通过代码各种形式交流。
