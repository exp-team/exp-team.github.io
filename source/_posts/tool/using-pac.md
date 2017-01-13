---
title: 如何使用PAC文件“科学上网”
categories: tool
tags: [tool,pac]
date: 2017-01-13
author: Wayne C.
---
万里长城是我们中华民族的瑰宝，是我们民族的骄傲，是世界八大奇迹之一，是我们中华的代表，它让我们避免了外族的侵略！

嗯，不过对我们现代中华儿女，我们要做的就是！翻越它去征服世界上所有的蛮夷之地！

<!-- more -->

## GFW

令人敬畏的Great Fire Wall，说起他的由来。。。等下，我有个顺丰快递。

## 为什么要“科学上网”

由于GFW的存在，我们失去了不少与友善外族沟通交流的机会，例如Facebook，Twitter，Instagram。我们只能靠那些处于水深火热环境中的朋友给我们搬运回来外族的消息。不过据说这些东西也不好玩，毕竟上面大部分都是外族百姓，我们不玩也罢。

然而！对我们技术宅们，不能与外族高端人士沟通，实属悲哀之事。既然州官不让，那百姓们就自己想办法咯。

“科学上网”方法其实有很多，今天介绍一个我个人比较认同的一个方法吧。

## 通过PAC文件“科学上网”

首先，你要有一个可用的代理。没有代理的话就自己想办法搞一个吧。我不教╭(╯^╰)╮

PAC(Proxy Auto Config)实际上就是一个脚本(Script)，通过这个脚本，我们能够让系统判断在怎么样的情形下，要利用哪一台Proxy来进行联机。

在“科学上网”方面呢，我们可以让系统确认，哪些网站通过代理访问，哪些直接通过本机网络访问，这样一来避免了直接整机挂VPN而导致国内不少网站访问不了的尴尬情况。

**PAC文件采用JavaScript编写，想要实现高级规则，最好有点JavaScript基础^_^**

### 基本函数

先新建一个*.pac文件，然后输(fu)入(zhi)以下代码。

~~~js
function FindProxyForURL(url, host) {
    return 'DIRECT';
}
~~~

FindProxyForURL是PAC文件的“主函数”，PAC文件一定要定义它，所有的请求都会进入这个方法，然后匹配规则。

其中 *return 'DIRECT';* 表示直接使用本机网络直接访问，这一段目前的含义是所有请求通过本机网络直接访问。

PAC一共支持三种访问方式

- DIRECT *直接联机而不透过 Proxy*
- PROXY host:port *使用指定的 Proxy 伺服机*
- SOCKS host:port *使用指定的 Socks 伺服机*

比如将代码改成

~~~js
function FindProxyForURL(url, host) {
    return 'PROXY 127.0.0.1:7070';
}
~~~

则表示所有的请求，以HTTP方式，通过本机的7070端口访问。

### 通过域名匹配规则

我这里就介绍一个比较常用的规则，通过域名匹配，如果一个请求在一个域名下，我们就走代理访问。

直接上代码吧。

~~~js
var autoproxy_host = {
    "google.com": 1,
    "twitter.com": 1
};
function FindProxyForURL(url, host) {
    var lastPos;
    do {
        if (autoproxy_host.hasOwnProperty(host)) {
            return 'PROXY 127.0.0.1:7070';
        }
        
        lastPos = host.indexOf('.') + 1;
        host = host.slice(lastPos);
    } while (lastPos >= 1);
    return 'DIRECT';
}
~~~

其实会Js的朋友应该很容易就能看懂了，不断分隔域名，然后去匹配autoproxy_host中设定好了的域名，如果匹配上了，我们就通过本机7070端口代理访问，否则就直接通过本机网络访问。比如这里，访问google.com和twitter.com的时候，就通过代理访问。

实际上用的时候，记得把127.0.0.1:7070换成你自己代理，如果是SSH的代理，就用SOCKS就好了。

### 使用PAC文件

#### Windows

Windows上面使用PAC文件很简单，新建一个你的PAC文件，放在一个固定的位置，比如

D:\学习资料\国外学习资料\中外文化交流\跨越\别看\说了别看\搞毛啊\setting.pac

然后，Internet设置 -> 连接 -> 局域网设置 

![](/bimg/pac-win-1.jpg)

勾选“自动检测设置”以及“使用自动配置脚本”，在“地址”里面填写

file:\\\\\\ 加 文件路径，如下

![](/bimg/pac-win-2.jpg)

然后多确认几次就好了，打开浏览器或者IE（没错我就是来黑IE的）试试看吧！

#### Mac OSX

Mac上面比较麻烦，因为最新的OSX是不支持本地文件设置的，你需要填写一个网络地址。比较好的办法是你现在本地建立一个服务器，然后把你的pac文件丢进去，然后通过http能访问就好了。

在Mac上建立本地服务器的方法很多，比如自带的apache。直接在命令行输入

~~~sh
sudo apachectl start
~~~

一般默认的目录都是 /Library/WebServer/Documents，你也可以修改/etc/apache2/httpd.conf里面的DocumentRoot配置项，修改服务器的默认路径。

把pac文件放进根目录，然后就可以直接通过 http://localhost/file.pac 来访问了。

接下来是配置网络，系统偏好 -> 网络 -> 高级 -> 代理

勾选“Automatic Proxy Configuration”，在右侧填写pac文件路径就好了。

![](/bimg/pac-mac-1.png)

至此，大伙儿就开开心心地来“科学上网”吧！！！\\(^o^)/！！

## 结束语

然后，没错，我就是被颜大大拜托过来耍宝的！=￣ω￣=
