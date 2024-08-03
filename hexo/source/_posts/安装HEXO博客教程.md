---
layout: 'title:'
title: 安装Hexo博客教程
date: 2022-06-09 15:04:31
categories:
- 建站
tags:
- 博客
- blog
---

前言
一、准备GIthub仓库
二、本地安装git
三、本地安装node.js
四、本地安装Hexo
五、搭建博客
六、部署到Github
前言
去年就有在 Github 搭建博客的想法，但是因为工作太忙搁置了，昨天想起来这事儿，于是网上各种查阅资料，感觉虽然搭建方式比较多，但都不是很全，走了很多弯路，折腾了我一天，才终于搭建好了自己的 GIthub 博客，在此记录梳理一下，希望可以帮到大家，欢迎交流！

博客采用的是Hexo框架（因为支持Markdown语法），使用的是Butterfly主题，搭建过程中的参考链接如下：

Hexo 官方文档：https://hexo.io/zh-cn/docs/
Hexo 官方主题：https://hexo.io/themes/
Butterfly 主题 GIthub：https://github.com/jerryc127/hexo-theme-butterfly
Butterfly 主题doc（主要包含主题配置及一些自定义）：https://butterfly.js.org/archives/

最终效果：https://wuqiuxu.github.io

一、准备GIthub仓库
首先你需要在 Github 有一个自己的账号

进入个人主页可以看到Repositories，点击进入仓库


新建一个 Github 仓库，仓库名称填写github用户名.github.io（ps：因为我已经搭建过了，所以下图会出现仓库已存在的警告）


点击最下方的Create repository提交仓库


仓库生成后，复制自己的仓库地址，后续部署的时候需要（如果仓库里面没有文件，进入之后就是图一的样子，否则是图二，红框里就是地址）


二、本地安装git
Mac 系统：brew install git
其他系统可参考官网：https://git-scm.com/downloads
三、本地安装node.js
安装 nvm
Mac系统：brew install nvm
其他系统可参考官网：https://nodejs.org/zh-cn/download

配置环境变量：vim ~/.zshrc
export NVM_DIR=~/.nvm
source $(brew --prefix nvm)/nvm.sh
生效环境变量：source ~/.zshrc
查看 node.js 的可用版本：nvm ls-remote
安装 node.js（版本需不低于 10.13，官方建议使用 Node.js 12.0 及以上版本）：nvm install v16.20.0

使用指定版本：nvm use v16.20.0
指定每次启动终端所使用的版本：nvm alias default v16.20.0
四、本地安装Hexo
安装 Hexo：npm install -g hexo-cli（这里提示可以更新npm版本，可以更新一下：npm install -g npm@9.7.1）

五、搭建博客
准备一个文件夹：mkdir blog
初始化：hexo init blog

进入文件夹：cd blog
安装：npm install

查看生成的文件：ls

克隆 butterfly 主题仓库：git clone -b master ``https://github.com/jerryc127/hexo-theme-butterfly.git`` themes/butterfly，然后在 themes 下就会多一个主题目录

修改blog目录下的 _config.yml（注意不是主题目录下的_config.yml），把 theme 的值改为 butterfly

配置主题，可以参考这篇文章：https://butterfly.js.org/posts/4aa8abbe/（这一步可以先跳过，等搭建好之后再修改）
生成文章：hexo new post "文章标题"，然后就可以在 blog/source/_posts 下生成的md文件编写了（这一步也可以先跳过，等搭建好之后再来写）
生成静态文件 public（每次修改主题配置后都需要重新生成）：hexo generate，缩写hexo g

启动本地服务器：hexo server，缩写hexo s（访问地址为：http://localhost:4000/，可以在本地查看页面，ctrl + c可以结束服务）

清除缓存文件 db.json 和已生成的静态文件 public ：hexo clean，缩写hexo cl
有时候对站点的更改不生效，可能需要先运行该命令再重新生成文件

六、部署到Github
修改blog目录下的 _config.yml（注意不是主题下的_config.yml）
a. type 类型配置为 git
b. repo 那里配置为自己的Github仓库地址

按照yml文件格式要求，:后面必须留有一个空格

安装一键部署插件：npm install hexo-deployer-git --save

一键部署到Github：hexo deploy ，缩写hexo d



PS：

更新butterfly主题后报错extends includes/layout.pug block content（网上方法没解决的可以来看看）
百度出来的解决方法都是一样的

```
npm install --save hexo-renderer-jade hexo-generator-feed hexo-generator-sitemap hexo-browsersync hexo-generator-archive
hexo clean
hexo g
hexo s
```

就很隔路，别人都解决了就我没解决，甚至错误都没有变，一点反馈都没有 = _ =
化悲愤为食欲，吃了顿饭，心情好了，回来 重新看了一遍终端输出信息，果然发现一些可以操作的细节

作为一名合格的LSP 码农，之前本能地无视风险继续安装了
反正也没有什么办法了，fix一下试试吧

```
npm aduit fix
npm aduit fix --force
```



接下来就是常规的清除缓存，生成文件，“开机”

```
hexo clean
hexo g
hexo s
```

