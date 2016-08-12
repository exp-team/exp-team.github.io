# exp team
百度经验研发团队博客，让我们团结一致，让我们的团队更强。

## 技术栈
我们的博客基于[hexo](https://hexo.io/zh-cn/)搭建，需要[node](https://nodejs.org/en/)和[git](https://git-scm.com/)环境的支持，具体安装过程可以看[这里](https://exp-team.github.io/)

第一步，安装git，可参考[这里](https://git-scm.com/book/zh/v2/%E8%B5%B7%E6%AD%A5-%E5%AE%89%E8%A3%85-Git)；附赠[常用git命令](http://yanhaijing.com/git/2014/11/01/my-git-note/)

第二步，安装node，可参考[这里](http://nodejs.cn/download/package-manager/)；附赠[npm使用指南](http://yanhaijing.com/tool/2015/09/01/my-npm-note/)

第三步，安装hexo命令行工具

	npm install -g hexo-cli@1.0.2 --registry=https://registry.npm.taobao.org


## 如何写博客
第一步，下载项目，并切换到source分支

	git clone git@github.com:exp-team/exp-team.github.io.git
	
	cd exp-team.github.io
	
	git fetch origin
	
	git checkout -b source origin/source

第二步，安装npm插件，**仅需安装一次**

	npm install --registry=https://registry.npm.taobao.org

第三步，在source/_posts目录下编写博文；附赠[markdown语法说明](http://wowubuntu.com/markdown/)

第四步，本地预览，127.0.0.1:4001

	hexo server -p 4001

## 如何发布
发布源文件，将源文件发布到github

	git add .
	git commit -m "blog: add"
	git push origin source

如果远程有更新的提交就会发生冲突，请先把远程新的比较合并到本地在提交

	git fetch
	git rebase origin/source

发布博文,确认没问题即可发布博文，搞定收工

	hexo deploy -g

*还可以用下面的命令查看生成的文件，默认在public目录*

    hexo generate

## License
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="http://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a><br /><span xmlns:dct="http://purl.org/dc/terms/" href="http://purl.org/dc/dcmitype/Text" property="dct:title" rel="dct:type">百度经验研发团队博客</span> 由 <a xmlns:cc="http://creativecommons.org/ns#" href="https://exp-team.github.io/" property="cc:attributionName" rel="cc:attributionURL">百度经验研发团队</a> 创作，采用 <a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">知识共享 署名-非商业性使用-相同方式共享 4.0 国际 许可协议</a>进行许可。<br />基于<a xmlns:dct="http://purl.org/dc/terms/" href="https://exp-team.github.io/" rel="dct:source">https://exp-team.github.io/</a>上的作品创作。
