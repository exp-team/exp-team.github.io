---
title: 人工智能离前端并不远 一步步教你开发一个机器学习APP（附源码）
categories: js
tags: js
date: 2017-11-04
author: 侯策
---

最近HBO电视网推出的美剧《硅谷Silicon Valley》席卷全球，里面有一个桥段介绍了超级有趣的iOS app－ [Not Hotdog。](https://www.theverge.com/tldr/2017/5/14/15639784/hbo-silicon-valley-not-hotdog-app-download)你甚至可以在APP Store上[下载](https://itunes.apple.com/app/not-hotdog/id1212457521)到它。


![是不是热狗？](http://upload-images.jianshu.io/upload_images/4363003-b6c49a02a1b0d3ba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


受启发于此，我打算开发一个实现同样功能的“机器人”：用户只需要上传任何一张图片，马上就可以得到反馈，告诉你这张图片的内容是不是一个热狗。最重要的是，我的代码**全部以JS实现**，是时候让前端工程师们在人工智能／机器学习领域大展身手了。

## 实现细节
这个APP以Twitter为宿主，基于Twitter Bot机器人：任何Twitter用户都可以发布一张图片，并且在上传描述文字中加入“@IsItAHotdog”，就能立即得到回复。就像大陆常用的微博加入"＃"描述符一样简单。


![热狗探测APP](http://upload-images.jianshu.io/upload_images/4363003-12400866446d9728.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


千万不要被表象所困扰，更不要被“人工智能／机器学习”的标签所迷惑。其实实现方式和原理非常简单。

首先，我forked [@BryanEBraun’s ](http://twitter.com/BryanEBraun)的开源作品[Twitter bot](https://github.com/bryanbraun/twitter-listbot)，它基于NodeJS，Twitter Bot译为机器人：会定时发推，或随机回复。

官方介绍内容也非常简洁明了：

>This is a simple twitter bot, designed to retweet the contents of a twitter list.

借助这个工具，接下来我的工作就是对提到"IsItAHotdog"的推文（即含有IsItAHotdog标签），作出回应。

在安装 [tuiter](https://www.npmjs.com/package/tuiter) NPM包之后，代码中引入依赖，并加入：

     var tu = require('tuiter')(config.keys);
     function listen() {    
        tu.filter({        
            track: 'isitahotdog'    
        }, function(stream) {        
            console.log("listening to stream");
            stream.on('tweet', onTweet);
        })
    }

当然，我们只对含有图片的推文进行处理：

      if(tweet.entities.hasOwnProperty('media') && tweet.entities.media.length > 0)

最后，我们把结果写进推文回复中：

    tu.update({
        status: "@" + tweet.user.screen_name + message,        
        in_reply_to_status_id: tweet.id_str    
    }, onReTweet);

## 训练模型
以上只是介绍了劫持推文，发布回复的内容。那么回复的结果应该怎么获得呢？我们怎么知道图片是不是热狗呢？这就到了最重要的一步。

熟悉深度学习的朋友可能会了解，接下来我们可能需要收集图片，并用Keras搭建CNN常用神经网络。其中Keras是一个兼容Theano和Tensorflow的神经网络高级包, 高度模块化，用他来组建一个神经网络非常快速便捷。

这些内容可能中文资料并不多，仅有的一些如果大家感兴趣的话，我推荐：

- [对比学习用 Keras 搭建 CNN RNN 等常用神经网络](http://www.jianshu.com/p/9efae7a20493)
- [Basic Machine Learning and Deep Learning](https://github.com/wepe/MachineLearning)

**但是这些深度学习的内容，可能很多前端工程师并不是太了解，那么我们就得重新修炼才能玩转这一切吗？**

别急，现在就可以开始！这里我给大家安利一下[AWS Rekognition，](https://aws.amazon.com/rekognition/)我们的APP也是基于AWS Rekognition来完成。

>Amazon Rekognition 是一种让您能够轻松为应用程序添加图像分析功能的服务。利用 Rekognition，您可以检测对象、场景和面孔，可以搜索和比较面孔，还可以识别图像中的不当内容。借助 Rekognition 的 API，您可以快速为应用程序添加基于深度学习的复杂视觉搜索和图像分类功能。

换句话说，“不了解机器学习，简单的调用几个API都应该会吧。”

Amazon Rekognition基于同样由Amazon计算机视觉科学家开发的成熟且高度可扩展的深度学习技术，每天能够分析数十亿张 Prime Photos 图像。

说到这里可能有些绕，其实来看下代码，非常的简单：

     var params = {
         Image: { 
             Bytes: body
         },
         MaxLabels: 20,
         MinConfidence: 70
     };
      
     rekognition.detectLabels(params, function(err, data) {
        if (err) console.log(err, err.stack); // an error occurred
        else {
            console.log(data);           // successful response
            var isItAHotdog = false;
            for (var label_index in data.Labels) {
                var label = data.Labels[label_index];
                if(label['Name'] == "Hot Dog") {
                   if(label['Confidence'] > 85) {
                        isItAHotdog = true;
                        tweetBasedOnCategorization(tweet, true);
                    }
                }
            }
            if(isItAHotdog == false) {
                tweetBasedOnCategorization(tweet, false);
            }
        }
    });

我把推文附带的图片下载到自己的服务器机器上，然后通过AWS Node SDK传递给Rekognition，并设置相应的参数，包括置信区间等。
最后，在回调中获得最终结果。

## 最终结果
让我们来看一组测试结果吧：


![测试](http://upload-images.jianshu.io/upload_images/4363003-f236923571a6f99f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![测试](http://upload-images.jianshu.io/upload_images/4363003-c388ddadb880d566.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这一切的开发过程都是非常的简单，如果你想看到源码，我fork了一份，并加入了中文注解。请点击[这里查看源码。](https://github.com/HOUCe/hot-dog-bot)

本文翻译自[Building Silicon Valley’s Hot Dog App in One Night](https://hackernoon.com/building-silicon-valleys-hot-dog-app-in-one-night-aab8969cef0b)，对于原文进行了部分扩展。

Happy Coding!

**最后，可耻地做一波广告：**

**受到gitChat的邀请，我要开分享了。**形式类似知乎Live，但是这个平台我感觉少了浮躁而更加专业。
主题内容为：**面对前端六年历史代码，如何接入并应用ES6解放开发效率**。

**我邀请了资深前端专家，社区网红[@颜海镜](https://zhuanlan.zhihu.com/p/26929765?group_id=847800456846639104)同我一起，详情介绍点击[这里。](https://zhuanlan.zhihu.com/p/26929765?group_id=847800456846639104)**

微信扫描下方二维码，即可参加：


![扫描此二维码](http://upload-images.jianshu.io/upload_images/4363003-5058b054ba03b9bc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![内容介绍](http://upload-images.jianshu.io/upload_images/4363003-ecf3f61112d467b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)





PS: 作者[Github仓库](https://github.com/HOUCe)，欢迎通过代码各种形式交流。