---
title: 做出Uber移动网页版还不够 极致性能打造才见真章
categories: js
tags: js
date: 2017-04-20
author: 侯策
---
之前分享过两篇关于React技术栈的原创文章：
- [React+Redux打造“NEWS EARLY”单页应用 一个项目理解最前沿技术栈真谛](http://www.jianshu.com/p/cde3cf7e2760)
- [一个react+redux工程实例](http://www.jianshu.com/p/8e28be0e7ab1)。

今天进一步剖析一个实际案例：**Uber APP 移动网页版。**

如果你对React技术栈没有多大兴趣，或者不是很了解，也没有关系。因为读下来，你会发现，这篇文章的真谛其实在于**性能优化**上。

本文灵感和主体内容翻译自Narendra N Shetty的[文章：How I built a super fast Uber clone for mobile web](https://hackernoon.com/how-i-built-a-super-fast-uber-clone-for-mobile-web-863680d2100f)，同时进行了大量扩充以及深挖。

## 出发点和产品雏形
很早以来，相信大家都会认同一个观点：**移动端流量超越PC端是不争的事实。**对于前端开发者来说，移动端web的开发同样非常有趣，也充满挑战。

这不，Uber最近发布了最新版本APP，全新样式，体验超棒。于是，笔者决定使用React来从零开始构建一个新的属于自己的Uber。

开发期间，笔者花费了很多时间在基础组件和样式搭建上。这环节中，主要应用了[Uber官方开放的React地图库](https://github.com/uber/react-map-gl)，并在地图上“目的地”和“起始点”之间采用svg-overlay和html-overlay去绘制路线。

最终的基本交互可以参考下面Gif图：



![uber.gif](http://upload-images.jianshu.io/upload_images/4363003-59027620881f3e91.gif?imageMogr2/auto-orient/strip)





## 走上优化之路
现在，我们有基本的产品形态了。目前面临的问题在于提高产品的各方面性能体验。我使用了Chrome Lighthouse去检验产品的性能表现。最终得到的结果为：




![结果1.png](http://upload-images.jianshu.io/upload_images/4363003-ea0f448ffd3ae4f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



wow...
第一次绘制时间就已经**接近2秒**，后面的时间惨不忍睹就不要看了吧。
想象一下，一个用户拿出手机，企图叫车。主屏时间的绘制就超过了19189.9ms，这是极其不能忍受的。

接下来，什么也不说了，撸起袖子，想办法去优化吧。

## 优化方法1－代码分离（Code Splitting）
我最开始想到并使用的方法就是：Code Splitting（代码分离），正好我们可以借助webpack来实现这项技术。
什么是webpack code splitting呢? 您可以参考[这里](https://webpack.github.io/docs/code-splitting.html)，如果英语阅读吃力，可以参考下面引文：

>code splitting就是指将文件分割为块(chunk)，webpack使我们可以定义一些分割点(split point)，根据这些分割点对文件进行分块，并实现按需加载。

因为笔者使用了React技术栈，并采用了react-router，所以代码的划分（split）就可以按照路由和加载时机进行。具体操作可以使用react-router的getComponent api来实现：

    <Route path="home" name="home" getComponent={(nextState, cb) => {
        require.ensure([], (require) => {
            cb(null, require('../components/Home').default);
        }, 'HomeView');
    }}> 

只有当对应路由被请求时，相应的组件才会被加载呈现。

同时，笔者使用了webpack的CommonChunkPlugin插件提取第三方代码。这是出于什么考虑呢？

细心的读者可能会发现上面的code splitting也许会存在一个问题：
**按需（按路由）引入资源后，这些资源可能存在大量重复代码。尤其是我们使用的第三方资源。**
想明白这个问题，这时候，你应该就会明白CommonChunkPlugin这个插件的意义了。关于这个插件配置方法有多种，这里我们采用了：有选择性的提取（对象方式传参）：

    {
        'entry': {
            'app': './src/index.js',
            'vendor': [
                'react',
                'react-redux',
                'redux',
                'react-router',
                'redux-thunk'
            ]
        },
        'output': {
            'path': path.resolve(__dirname, './dist'),
            'publicPath': '/',
            'filename': 'static/js/[name].[hash].js',
            'chunkFilename': 'static/js/[name].[hash].js'
        },
        'plugins': [
            new webpack.optimize.CommonsChunkPlugin({
                name: ['vendor'], // 公共块的块名称
                minChunks: Infinity, // 最小被引用次数，最小是2。传递Infinity只是创建公共块，但不移动模块。 
                filename: 'static/js/[name].[hash].js', // 公共块的文件名
            }),
        ]
    }

这样子，我们把公共代码（react、react-redux、redux、react-router、redux-thunk）专门抽取到vendor模块中。

通过上述方法，笔者欣喜地发现：
First meaningful paint时间由19189.9ms缩短到4584.3ms：



![结果2](http://upload-images.jianshu.io/upload_images/4363003-b7384bf2a9c9bb5a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



这无疑是激动人心的。



## 优化方法2－Server side rendering（服务端直出）
也许你一直在听说过“服务端渲染”或者“服务端直出”这样的名词。但是从未实践过，也从来没有了解过他的意义。好吧，这里我先描述一下，到底什么是服务端直出。

服务端直出，其实简单总结为服务器在接到来自浏览器第一次请求时，便返回一个“初步最终”HTML文档。这个HTML文档已经进行了数据拼接。这样用户能以最快的时间看到首屏的效果，当然这个效果是“阉割版”的，非最终版本。

这种方式主要是针对“前后分离”的传统模式。传统模式中，服务器返回HTML文档，之后浏览器解析文档标签，拉取CSS，之后拉取JS文件。JS文件加载完成之后，执行JS内容，并发送请求获取数据。最终，将数据渲染在页面上。

由此，Server side rendering方式将JS请求数据的过程放在了服务器上，甚至对于数据与HTML结合处理也可以在服务器上做。

这样一来，**主要就是加快了首屏渲染时间。**当然，使用服务端渲染，还能够优化前端渲染难以克服的SEO问题。

理论理解起来很简单，难处就在于服务器端环境的前端脚本如何处理，如何与客户端保持一致。

在这个项目中，我使用了Express作为nodeJS框架，结合react－router完成：

    server.use((req, res)=> {
        match({
        'routes': routes,
        'location': req.url
        }, (error, redirectLocation, renderProps) => {
            if (error) {
                res.status(500).send(error.message);
            } 
            else if (redirectLocation) {
                res.redirect(302, redirectLocation.pathname + redirectLocation.search);
            } 
            else if (renderProps) {
                // Create a new Redux store instance
                const store = configureStore();

                // Render the component to a string
                const html = renderToString(<Provider store={store}><RouterContext {...renderProps} /></Provider>);

                const preloadedState = store.getState();

                fs.readFile('./dist/index.html', 'utf8', function (err, file) {
                    if (err) {
                        return console.log(err);
                    }
                    let document = file.replace(/<div id="app"><\/div>/, `<div id="app">${html}</div>`);
                    document = document.replace(/'preloadedState'/, `'${JSON.stringify(preloadedState)}'`);
                    res.setHeader('Cache-Control', 'public, max-age=31536000');
                    res.setHeader("Expires", new Date(Date.now() + 2592000000).toUTCString());
                    res.send(document);
                });
            } 
            else {
                res.status(404).send('Not found')
            }
        });
    });


通过上述方法，我们欣喜地发现：
First meaningful paint时间已经缩短到921.5ms：



![结果3](http://upload-images.jianshu.io/upload_images/4363003-b4c9aa1eff18344c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



这无疑是令人振奋的。



## 优化方法3－Compressed static assets（压缩静态文件）
压缩文件，当然是一个容易想到而且行之有效的措施。为此，我使用了webpack的CompressionPlugin插件：

    {
        'plugins': [
            new CompressionPlugin({
                test: /\.js$|\.css$|\.html$/
            })
        ]
    }

同时，使用express-static-gzip来对服务端进行配置：

    server.use('/static', expressStaticGzip('./dist/static', {
        'maxAge': 31536000,
        setHeaders: function(res, path, stat) {
        res.setHeader("Expires", new Date(Date.now() + 2592000000).toUTCString());
            return res;
        }
    }));

express-static-gzip是一个处于express.static之上的中间件。如果对于指定路径的文件没有找到压缩版本，就使用为压缩版本进行返回。

经过此处理，我们缩短了400ms时间，OK，现在First meaningful paint时间为546.6ms.



![结果4](http://upload-images.jianshu.io/upload_images/4363003-29234133ac7f9edd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## 优化方法4－Caching（缓存）
截止到此，我们已经从最初的19189.9ms已经优化到546ms，我们当然继续可以在客户端进行**静态文件缓存**来使得加载时间变得更短。

笔者使用了[sw-toolbox](https://github.com/GoogleChrome/sw-toolbox)搭配service workers进行。

>sw-toolbox：A collection of service worker tools for offlining runtime requests.
Service Worker Toolbox provides some simple helpers for use in creating your own service workers. Specifically, it provides common caching strategies for dynamic content, such as API calls, third-party resources, and large or infrequently used local resources that you don't want precached.

简单翻译下：
Service Worker实现常见运行时缓存模式，例如动态内容、API调用以及第三方资源，实现方法就像编写README一样简单。

也许到这里你一头雾水，没关系，我们从最初开始，了解一下什么是service worker:

>在2014年，W3C公布了service worker的草案，service worker提供了很多新的能力，使得web app拥有与native app相同的离线体验、消息推送体验。
service worker是一段脚本，与web worker一样，也是在后台运行。
作为一个独立的线程，运行环境与普通脚本不同，所以不能直接参与web交互行为。native app可以做到离线使用、消息推送、后台自动更新，service worker的出现是正是为了使得web app也可以具有类似的能力。

而sw-toolbox，顾名思义，就是service worker一个toolbox，具体我们看代码：

    toolbox.router.get('(.*).js', toolbox.fastest, {
        'origin':/.herokuapp.com|localhost|maps.googleapis.com/,
        'mode':'cors',
        'cache': {
            'name': `js-assets-${VERSION}`,
            'maxEntries': 50,
            'maxAgeSeconds': 2592e3
        }
    });

上面代码的意思是，我们对于get类型的请求，当请求内容为js脚本时，应用toolbox.fastest handler处理。
toolbox.fastest指示：对于这个请求，我们既从缓存中获取，也同时通过正常的请求network获取。**这两种方式哪个返回快，就应用哪一个。**
另外，toolbox.router.get的第三个参数表示配置项。

考虑周到的读者可能会想，上面是对于支持Service worker的浏览器，那么对于不支持的浏览器呢？我们干脆设置：

    res.setHeader("Expires", new Date(Date.now() + 2592000000).toUTCString());

通过这样处理，我们来直观感受一下页面加载瀑布流：





![使用Service worker](http://upload-images.jianshu.io/upload_images/4363003-1a5ca5dd910efece.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![不使用Service worker](http://upload-images.jianshu.io/upload_images/4363003-456cb5a07f1acd0b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)





## 优化方法5－Preload and then load（预加载／延后加载）
如果你还没听说过“Preload”，不要紧。我们这就来了解一下：

>Preload作为一个新的web标准，旨在提高性能和为web开发人员提供更细粒度的加载控制。Preload使开发者能够自定义资源的加载逻辑，且无需忍受基于脚本的资源加载器带来的性能损失。

换成你能听明白的话来说：
**preload建议允许始终预加载某些资源，浏览器必须请求preload标记的资源。**

这样子，究竟有什么意义呢？
举个例子：比如一些隐藏在CSS和Javascript中的资源。
当浏览器发现自己需要这些资源时已经为时已晚，所以大多数情况，这些资源的加载都会对页面渲染造成延迟。

preload的出现就是为了优化这个过程。
对于preload的兼容性，可以参考[这里。](http://caniuse.com/#search=preload)

对于不支持preload的浏览器，笔者使用了prefetch来处理。
但于preload不同，prefetch的作用是告诉浏览器加载下一页面可能会用到的资源，注意，是下一页面，而不是当前页面。因此该方法的加载优先级非常低。

这些新标准其实很有意思，里面的内容远不止这些。有兴趣的同学可以自行了解，也欢迎与我讨论。

回到正题，我在head标签中使用：

    <link rel="preload" ... as="script">

最终优化的结果如图：

![最终结果](http://upload-images.jianshu.io/upload_images/4363003-8823ee4b69444430.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 总结
其实，使用React＋Webpack做出一个Uber已经不是重点了。真正激动人心的是整套流程的优化之路。我们使用了大量成熟的、未成熟（新技术），希望对读者有所启发！