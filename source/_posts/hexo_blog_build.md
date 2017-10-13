---
title: 如何搭建 hexo 博客
date: 2017-06-09 15:20:08
tags:
  - hexo
  - blog
categories:
  - hexo
keywords:
  - blog
description:
  - hexo 建立博客
---

**摘要**:本文主要介绍如何使用hexo建立自己的博客。

<!--more-->

## 1 安装node.js

```shell
wget -qO- https://raw.github.com/creationix/nvm/master/install.sh | sh
```

安装完成后执行：`nvm install stable`, 即可安装node.js。

使用npm命令安装hexo

```shell
npm install -g hexo-cli
```

安装完成之后，使用`hexo --version`查看hexo是否正确安装。

![hexo 安装](http://i.imgur.com/OsK3uSC.jpg)

【PS】目前npm官方源在国内访问并不稳定，如果无法直接安装，请更换国内npm源。执行以下命令更换淘宝npm源

```shell
npm config set registry https://registry.npm.taobao.org
```

## 2 安装git

已经安装过git跳过此步骤

debian/ubuntu使用：

```shell
sudo apt-get install git-core
```

## 3 hexo使用

### 3.1 初始化一个站点

```shell
hexo init louie_blog
```

此命令用于执行站点的初始化。执行后，folder文件夹会成为一个Hexo站点文件夹，执行过程中涉及安装多个nodejs模块包以及git clone操作。

![hexo初始化站点](http://i.imgur.com/zzPkjDZ.jpg)

在初始化一个hexo站点文件夹之后，该文件夹的目录结构如下：

![hexo目录结构](http://i.imgur.com/72R8bGc.jpg)

详细说明如下：

- 1、`_config.yml`是YAML格式文件，也是Hexo的站点配置文件（敲黑板！重点重点！）
- 2、node_modules包含使用Hexo需要的其他node.js模块，以后安装的hexo相关模块也放在这里
- 3、package.json配置hexo运行需要的node.js包，不用手动更改（PS：通常不需要干预它，不过其中有一条"name": "hexo-site"起着告诉hexo该文件夹是hexo站点的作用，因此更加不要修改该文件内容，安装hexo其他模块也依赖该文件）
- 4、scaffolds是模板文件夹，不过这里的“模板”概念没有那么高端。这个“模板”就是指新建的markdown文件的模板，每新建一个markdown文件（由于Hexo使用markdown语法，在渲染生成静态HTML页面之前，源文件都是markdown文件），就会包含对应模板的内容。该文件夹内有三个模板：draft.md，草稿的模板page.md，页面的模板post.md，文章的模板
- 5、source是源文件文件夹，此处存有渲染生成静态页面需要的所有源文件，包括markdown文件、图片文件。默认此文件夹下只有一个`_post`文件夹，存放文章的markdown源文件。每个页面有一个以该页面命名的文件夹，也存放在source文件夹下。该文件夹下除了`_post`外，所有以下划线开头的或以.开头的文件夹都会被忽略。
- 6、themes是主题文件夹，Hexo的主题作用与WordPress相同。
- 7、public文件夹，默认没有，存放生成的静态文件。

打开`_config.yml` 修改配置：

```yaml
title: Hexo #站点的标题
subtitle: #站点的副标题
description: #站点的描述，写一段话来介绍你的博客吧:)，主要为SEO使用
author: John Doe #显示的文章作者名字，例如我配置的是fourstring
language: #语言。简体中文是zh-Hans
timezone: #时区，可以不配置，默认以本地时区为准
url: http://yoursite.com #你的站点地址，如果是全站HTTPS记得写成https://domain.com
root: / #如果您的网站存放在子目录中，例如 http://yoursite.com/blog，则请将您的 url 设为 http://yoursite.com/blog 并把 root 设为 /blog/。（引用自官方文档）
permalink: :year/:month/:day/:title/ #固定链接格式。这项配置的格式为：变量1/变量2/变量3...，其中合法的变量格式为“:变量名”（注意，:是变量的组成部分！）这样生成的效果为/2016/08/10/文章标题。默认的固定链接格式存在一些问题，下文讲解
per_page: 10 #设置每页文章篇数，设为0可以关闭分页功能
theme: #使用的主题。下文讲解
deploy: #部署配置，其值是一个杂凑表，注意缩进，下文详细讲解
```

### 3.2 更换主题

Hexo的主题存储在`louie_blog/themes`目录下

hexo主题星级排名[https://www.zhihu.com/question/24422335](https://www.zhihu.com/question/24422335)

推荐n4l.pw使用的Hexo主题：Next，功能极其强大，是目前github上star第一的Hexo主题：[https://github.com/iissnan/hexo-theme-next](https://github.com/iissnan/hexo-theme-next)。官方文档讲解非常详细，鉴于篇幅，这里只提一个小技巧，在文章中加入`<!--more-->`标签，主题会自动将标签之前的内容截取作为文章摘要输出在首页。（可见下图效果，点击放大）。

![hexo_theme](http://i.imgur.com/uvNNid6.jpg)


### 3.3 写作

```shell
hexo new post "Test"
```

可以新建一篇文章。post参数可以省略，_config.yml中的default_layout:设置了默认类型，默认值是post，你可以改成draft来默认存储为草稿。

然后用任意你喜欢的编辑器打开home/source/_post/标题.md文件就可以写作了。（PS：Windows下markdown编辑器可以使用MakdownPad2，OS X下可以使用MWeb，功能非常强大）。

【MarkdownPad2】
下载地址：[http://markdownpad.com/](http://markdownpad.com/)
注册码：
[http://www.jianshu.com/p/9e5cd946696d](http://www.jianshu.com/p/9e5cd946696d)

	邮箱：
	Soar360@live.com
	
	授权秘钥：
	GBPduHjWfJU1mZqcPM3BikjYKF6xKhlKIys3i1MU2eJHqWGImDHzWdD6xhMNLGVpbP2M5SN6bnxn2kSE8qHqNY5QaaRxmO3YSMHxlv2EYpjdwLcPwfeTG7kUdnhKE0vVy4RidP6Y2wZ0q74f47fzsZo45JE2hfQBFi2O9Jldjp1mW8HUpTtLA2a5/sQytXJUQl/QKO0jUQY4pa5CCx20sV1ClOTZtAGngSOJtIOFXK599sBr5aIEFyH0K7H4BoNMiiDMnxt1rD8Vb/ikJdhGMMQr0R4B+L3nWU97eaVPTRKfWGDE8/eAgKzpGwrQQoDh+nzX1xoVQ8NAuH+s4UcSeQ==

**自定义链接格式太蠢 **

可能语言不是很严谨，不过给我的第一感觉就是这样。由于链接最后没有带上.html后缀名，而且生成文件的MIME类型似乎不太对，导致用默认链接格式的话，nginx web server会直接进行文件下载。。。能不能像WordPress那样，为每篇文章自定义一个简短的英文名称作为链接呢？

我们需要用到Hexo的permalinkFront-matter选项。先编辑模板文件home/scaffolds/post.md，在其Front-matter中加入permalink:即可。

**分类和标签**

默认的主题菜单栏是没有标签和分类两个页面的。而且默认的模板中Front-matter也只有tags选项，没有分类选项。是不是Hexo没有这些功能呢？答案是否定的。

PS：这两个选项的值都是一个清单，注意缩进。

编辑模板文件home/scaffolds/post.md，加入categories:，如图所示：

![hexo format](http://i.imgur.com/AHaOcCJ.jpg)

然后执行：

```shell
hexo new page categories
hexo new page tags
```

![hexo format 设置](http://i.imgur.com/bUOdTY8.jpg)

创建标签和分类页面，如果你的主题支持，它们不需要填充任何内容，主题会自动生成这两个页面的内容，你只需要将它们加入菜单栏即可。（这并不意味着不用生成这两个页面）
默认主题菜单栏修改方法如下：
编辑`home/themes/landscape/_config.yml`文件，在`menu:`项下加入显示名称: 路径即可，如下图所示：

** 评论功能 **

这个主要看主题是否支持。例如我使用的next主题，支持多说和disqus两套系统。特别提醒，由于本身是静态化的，所以必须依靠第三方服务提供评论功能。

如果想让某篇文章禁用评论功能，next主题需要在Front-matter中加入：

```yaml
comment: false
```

一般来说页面都不需要评论功能，可以编辑home/scaffolds/page.md，在Front-matter中加入

```yaml
comment: false
```

编辑`source/_posts/Test.md`文件，使用markdown语法撰写自己的博客。

### 3.4 预览

执行`hexo generate`生成静态文件，执行`hexo server`，本地开启服务器，然后浏览器访问`http://localhost:4000`即可看到预览效果了。

![](http://i.imgur.com/AvHMXXg.jpg)

## 4 部署

最后一个重点难点内容，如何将public文件夹下的内容发布到服务器上。

- 1.Github Pages
  这个服务允许github用户发布静态页面，无限空间流量，适合轻度用户。我没有采用这种方法，限于篇幅也不会介绍，如果你需要，Google之。

- 2.直接复制
  理论上可行的方法。毕竟public下的文件到哪里都能直接变成一个可运行的站点。但是这种方法太蠢了，看似可行，其实蕴含着一大堆问题，比如，无效流量、重复文件……

- 3.Git版本控制系统
  官方支持的部署方式之一。利用Git版本控制系统的强大功能，通过ssh上传文件。这也是我采用并且下文讲解的方法。

- 4.Heroku
  没有了解过这个方式。

- 5.Openshift
  Openshift是著名厂商Red Hat的PaaS服务，功能十分强大。关于如何使用该服务的教程网上已经很多，限于篇幅不再介绍。（如果各位有需要，我会考虑另开文章讲解，毕竟这个平台是相当好用的，请留言）

- 6.rsync
  利用强大的同步工具rsync进行同步，这种同步方式只需要用户提供一个能访问bash的Linux用户，并且服务器上安装了rsync软件包，其余一切涉及rsync命令的操作都由hexo自动完成，更加简单。推荐新手使用这种方式。

- 7.FTPsync
  直接通过FTP协议进行同步。如果购买了虚拟主机，可以考虑这种方式。然而这个插件写得也很蠢。第一次使用必须自己手动上传所有文件，否则会无限报连接被重置错误（我就不配图了，工作量有点大）。

基于上述方式的优缺点，本文将讲解如何使用Git和rsync进行服务端部署。

### 4.1 Git版本控制系统

服务端配置

4.1.1 编译安装nginx

** 这里跳过 **

执行`sudo nginx -V`测试是否安装成功。

![nginx 版本](http://i.imgur.com/lEEJxHH.jpg)

4.1.2 配置Git仓库

选择合适的路径，建立文件夹：

这里选择根目录下的`/web`
​	
```shell
sudo mkdir /web
```

并更改所有者为自己
```shell
sudo chown -R louie:louie /web	
```

建立目录
```shell
mkdir -p /web/blog/hexo  #Git仓库，不存储网站文件
mkdir -p /web/blog/hexo_blog.git #实际存储网站文件目录
```

执行如下命令，初始化空的Git仓库：

```shell
git init --bare hexo_blog.git
```

![hexo_blog git仓库初始化](http://i.imgur.com/E9XzzzZ.jpg)

然后进入该仓库，配置`post-receive hooks`。

	钩子(hooks)是一些在”$GIT-DIR/hooks”目录的脚本, 在被特定的事件(certain points)触发后被调用 。当”git init”命令被调用后, 一些非常有用的示例钩子文件(hooks)被拷到新仓库的hooks目录中; 但是在默认情况下这些钩子(hooks)是不生效的 。 把这些子文件(hooks)的”.sample”文件名后缀去掉就可以使它们生效了。
	需要关注的是post-receive的钩子，当push操作完成之后这个钩子就会被调用。

进入到hooks目录

建立`post-receive`文件，输入

```shell
#!/bin/sh
git --work-tree=/web/blog/hexo --git-dir=/web/blog/hexo.git checkout -f
```

赋予可执行权限：

```shell
chmod +x post-receive
```

![git hooks](http://i.imgur.com/0UwN6HX.jpg)

本文使用的是自己搭建的Gitolite，已经添加了无密码登陆权限。

4.1.3 nginx web server配置
首先建立虚拟主机配置文件夹：

```shell
npm install hexo-deployer-git --save
```


