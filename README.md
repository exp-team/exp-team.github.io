# exp team
百度经验研发团队博客，让我们团结一致，让我们的团队更强。

## 技术栈
我们的博客基于[hexo](https://hexo.io/zh-cn/)搭建，需要[node](https://nodejs.org/en/)和[git](https://git-scm.com/)环境的支持，具体安装过程可以看[这里](https://exp-team.github.io/)

第一步，安装hexo命令行工具

    npm install -g hexo-cli@1.0.2

第二步，下载本项目，并切换到source分支

    git clone git@github.com:exp-team/exp-team.github.io.git

    git checkout -b origin/source

第三步，安装node组件

    npm install

第四步，启动hexo服务器

    hexo server -p 4001

就可以在127.0.0.1:4001查看本博客的内容了

生成博客内容

    hexo generate

发布博客内容到github

    hexo deploy -g

## License
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="http://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a><br /><span xmlns:dct="http://purl.org/dc/terms/" href="http://purl.org/dc/dcmitype/Text" property="dct:title" rel="dct:type">百度经验研发团队博客</span> 由 <a xmlns:cc="http://creativecommons.org/ns#" href="https://exp-team.github.io/" property="cc:attributionName" rel="cc:attributionURL">百度经验研发团队</a> 创作，采用 <a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">知识共享 署名-非商业性使用-相同方式共享 4.0 国际 许可协议</a>进行许可。<br />基于<a xmlns:dct="http://purl.org/dc/terms/" href="https://exp-team.github.io/" rel="dct:source">https://exp-team.github.io/</a>上的作品创作。
