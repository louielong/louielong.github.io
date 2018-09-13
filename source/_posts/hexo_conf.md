---
title: Hexo 博客美化配置
date: 2018-09-13 16:44:38
tags:
  - Hexo
  - Blog
keywords:
  - Hexo
categories:
  - Hexo
description:
  - Hexo 博客自定义配置
summary_img:
  - https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1525777662294&di=bc13570a74bfd4138f597d701e923b13&imgtype=0&src=http%3A%2F%2Ffile.itnpc.com%2F201608%2Fbd8b3313886100de04663eb3aacdd068.jpg
---

## 1 前言

使用Hexo一年多了，页面样式主题都是好久之前的一直没有更改过。闲来想换换新货，体验下新的代码，同时加入一些羡慕已久的页面特效，本文记录一下升级的过程。以一个新建立的blog开始记录，至于Hexo的安装则跳过了。

## 2 初始blog

为了记录自定义的修改这里预先建立的一个blog的git仓库，方便记录修改的地方。

使用Hexo命令建立一个初始blog，随后加入到仓库中。

```shell
louie@ubuntu:~/workplace$ hexo init blog
```

下载主题，这里使用next主题，使用v6.1.0的版本方便后续升级也便于记录修改起始的版本。

![Next Themes](https://raw.githubusercontent.com/theme-next/hexo-theme-next/master/source/images/logo.svg?sanitize=true)

```shell
louie@ubuntu:~/workplace/blog/themes$ git clone --branch v6.1.0 https://github.com/theme-next/hexo-theme-next.git next
```

Next 主题官网更新后给出了一波自定义教程，总结的非常全面，可以直接参考官方的配置：[传送门](http://theme-next.iissnan.com/getting-started.html#third-party-services)

## 3 配置、美化blog

### 3.1 配置`_config.yml`

修改默认主题

```yaml
# Site

title: Louie's Blog
subtitle: O ever youthful, O ever powerful.
description: 一只会说666的程序媛鼓励师
author: Louie Long
language: zh-Hans
timezone: Asia/Shanghai

# Extensions

 theme: next
```




设置网站图标，在souce目录下创建一个uploads目录(因为next搜索指定了[目录路径)](https://github.com/theme-next/hexo-theme-next/blob/master/_config.yml#L192-L195)，将图片等都放置在这个目录同一管理。修改next主题的配置文件`themes/next/_config.yml`，avatar为图像图片，favicon为网站icon图片 ，推荐一个好用的[ico 制作网站](https://tool.lu/favicon/)。apple和safari的像素要高许多可以参考[原主题ico](https://github.com/theme-next/hexo-theme-next/tree/master/source/images)大小。

> favicon:
>   small: /uploads/favicon-16x16.ico
>   medium: /uploads/favicon-32x32.ico
>   apple_touch_icon: /uploads/favicon-48x48.ico
>   safari_pinned_tab: /uploads/favicon-48x48.ico
>
> avatar: /uploads/shadian.png

blog首页底部的版权信息，直接修改`themes/next/_config.yml`的`footer`内容

```yaml
footer:
  # Specify the date when the site was setup.
  # If not defined, current year will be used.
  since: 2015

  # Icon between year and copyright info.
  icon:
    # Icon name in fontawesome, see: https://fontawesome.com/v4.7.0/icons
    # `heart` is recommended with animation in red (#ff0000).
    name: user
    # If you want to animate the icon, set it to true.
     animated: true
     # Change the color of icon, using Hex Code.
     color: "#808080"

  # If not defined, will be used `author` from Hexo main config.
   copyright:
  # -------------------------------------------------------------

  # Hexo link (Powered by Hexo).

   powered: true

   theme:
     # Theme & scheme info link (Theme - NexT.scheme).
     enable: true
     # Version info of NexT after scheme info (vX.X.X).
     version: true
```

根据需求自行修改，英文也很简单易懂^-^。

### 3.2 创建标签等页面

创建标签、关于、分类、归档等页面，只需要调用hexo的命令在`blog/source`目录下生成指定的目录文件即可

```shell
hexo new page "tags"
hexo new page "about"
hexo new page "categories"
```

随后即会在source目录下生成相应的目录及文件，修改对应的文件如`about/index.md`，将comments设为`false`，禁止在该页面进行评论。

```shell
cat categories/index.md
 ---
title: 分类
date: 日期
type: "categories"
comments: false
---
```

设置完成后可以通过`hexo g`和`hexo s`进行效果查看，这里不再赘述了。同时需要在主题中打开相应的开关`themes/next/_config.yml`

> about: /about/ || user
> tags: /tags/ || tags
> categories: /categories/ || th
> archives: /archives/ || archive
> commonweal: /404/ || heartbeat

### 3.3 设置首页文章显示篇数

安装相关插件，可以看到`package.json`中的信息有修改，显示新加入的插件

```shell
npm install --save hexo-generator-index
npm install --save hexo-generator-archive
npm install --save hexo-generator-tag
```

修改全局配置文件`blog/_config.yml`，增加以下内容

> index_generator:
>   per_page: 10
>
> archive_generator:
>   per_page: 20
>   yearly: true
>   monthly: true
>
> tag_generator:
>   per_page: 10

其中`per_page`字段是期望每页显示的篇数。`index`, `archive`及`tag`开头分表代表主页，归档页面和标签页面。

### 3.3 增加字数统计功能

Next主题集成了[Word Count 功能](https://github.com/theme-next/hexo-symbols-count-time)，首先安装插件

```shell
npm install hexo-symbols-count-time --save
```

插件主要功能：

- 字数统计:WordCount
- 阅读时长预计:Min2Read

安装完插件后仅需要修改根配置文件`blog/_config.yml`即可

> symbols_count_time:
>   symbols: true
>   time: true
>   total_symbols: true
>   total_time: true

![hexo word count](https://i.imgur.com/CA9U4yU.jpg)



### 3.4 增加站内搜索功能

Next主题支持Google、Algolia、Swiftype三种搜索，这里选用Algolia搜索。设置方式也很多，随便搜一下就可以找到。首先需要到[Algolia官网](https://www.algolia.com/)进行注册，然后获得API Key填入根配置文件中。

参考[Algolia GitHub](https://github.com/theme-next/hexo-theme-next/blob/master/docs/ALGOLIA-SEARCH.md)链接，安装algolia插件

```shell
npm install --save hexo-algolia
```

在`blog/_config.yml`配置文件添加如下信息

> algolia:
>   applicationID: 'Application ID'
>   apiKey: 'Search-only API key'
>   indexName: 'indexName'
>   chunkSize: 5000

将`Application ID`、`Search-only API key`以及`indexName`都替换成自己的信息。同时需要将next主题中algolia搜索开关打开

> algolia_search:
>
>  enable: true

执行下述命令

```shell
$ export HEXO_ALGOLIA_INDEXING_KEY=Search-Only API key # Use Git Bash
# set HEXO_ALGOLIA_INDEXING_KEY=Search-Only API key # Use Windows command line
$ hexo clean
$ hexo algolia
```

### 3.5 添加首页文章以摘要显示

开启主题的摘要功能，`length`表示显示摘要的截取字符串长度。

> auto_excerpt:
>   enable: true
>   length: 150

### 3.6 添加统计

#### 3.6.1 文章阅读统计及热度

如图所示的效果，看起来很酷炫

![文章阅读统计](https://i.imgur.com/QjU61uL.jpg)

参考[leanclond next配置指南](https://github.com/theme-next/hexo-leancloud-counter-security)，首先安装`leanclound`插件

```shell
npm install hexo-leancloud-counter-security --save
```

这里需要用到leancloud，注册及配置查考链接：[传送门](https://www.jianshu.com/p/702a7aec4d00)，首先注册账号，经过注册创建应用等过程后拿到key填入到next主题中

> leancloud_visitors:
>   enable: true
>   app_id: M8d5yPXQ5l0kwFz1bO6cF***********
>   app_key: QEarlHm5c8XcuF22*****

将统计结果修改为热度，修改`themes/next/languages/zh-CN.yml`文件中`post`段中`view`中文翻译为`热度`。

![view字段](https://i.imgur.com/bABWCvv.jpg)

打开`themes/next/layout/_macro/post.swig`搜索`leancloud-visitors-count`字段，添加`<span>sheshid</span>`

![摄氏度符号添加](https://i.imgur.com/TBcABwW.jpg)

后续还有一些关于Leanclound的设置，为节省篇幅，请直接访问：[传送门](https://leaferx.online/2018/02/11/lc-security/)



#### 3.6.2 添加全站访问量统计

全站访客统计使用[不蒜子](http://ibruce.info/2015/04/04/busuanzi/)，在`themes/next/layout/_partials/footer.swig`中添加如下内容。

```html
<script async src="//dn-lbstatics.qbox.me/busuanzi/2.3/busuanzi.pure.mini.js">
</script>
<span id="busuanzi_container_site_pv">
| 本站总访问量<span style="color:#FA8072" id="busuanzi_value_site_pv"></span>次
</span>
<span id="busuanzi_container_site_uv">
| 本站访客数<span style="color:#FA8072" id="busuanzi_value_site_uv"></span>人次
</span>
```

### 3.7 修改文章底部的那个带#号的标签

![](https://i.imgur.com/5cleWRI.jpg)

修改文件`themes/next/layout/_macro/post.swig`，找到`rel="tag">#`，将`#`替换为`<i class="fa fa-tag"></i>`即可。

### 3.8 文章末尾效果

1)添加“本文结束”标记

效果图：

![](https://i.imgur.com/wIS1EFZ.jpg)

创建文件`themes/next/layout/_macro/passage-end-tag.swig`，填入一下内容

```html
{% if not is_index %}
        <div blockquote class="blockquote-center" style="color: ##32CD32;font-size:18px;">
------ 本文结束 ------</div>
{% endif %}
```

随后修改`themes/next/layout/_macro/post.swig`，在`{### END POST BODY ###}`之前添加

```html
{% if not is_index %}
	<div>
    	{% include 'passage-end-tag.swig' %}
    </div>
{% endif %}
```

2）添加文末版权声明

效果如下

![](https://i.imgur.com/h2tADWv.jpg)

在`themes/next/layout/_macro/`生成一个`custome-cc.swig`文件，填入一下内容

```jinja2
{% if not is_index %}
<br/>
<div style="border: 1px solid black">
<div style="margin-left:10px">
<span style="font-weight:blod">版权声明</span>
<img src="/uploads/cc.png" >
<br/>
<p style="font-size: 10px;line-height: 30px"><a href="http://ylong.net.cn" style="color:#258FC6">Louie's Blog</a> by <a href="http://ylong.net.cn" style="color:#258FC6">louie long</a> is licensed under a <a href="https://creativecommons.org/licenses/by-nc-nd/4.0/" style="color:#258FC6">Creative Commons BY-NC-ND 4.0 International License</a>.<br/>
由<a href="http://ylong.net.cn" style="color:#258FC6">龙雨</a>创作并维护的<a href="ylong.net.cn" style="color:#258FC6">Louie's Blog</a>博客采用<a href="https://creativecommons.org/licenses/by-nc-nd/4.0/" style="color:#258FC6">创作共用保留署名-非商业-禁止演绎4.0国际许可证</a>。<br/>
本文首发于<a href="http://ylong.net.cn" style="color:#258FC6">Louie's Blog</a> 博客（ <a href="http://ylong.net.cn" style="color:#258FC6">http://ylong.net.cn</a> ），版权所有，侵权必究。
<br/>
转载请注明作者和链接地址<a href="http://ylong.net.cn" style="color:#258FC6">http://ylong.net.cn</a>, 如对文章内容有疑问请联系邮箱（ <a style="color:#258FC6">longyu805@163.com</a> ）。</p>
</div>
</div>
{% endif %}
```

需要在`source/uploads`下上传图片如下

![cc image](https://i.imgur.com/ARwLVLp.png)

同样修改`themes/next/layout/_macro/post.swig`，在`post-footer`之前添加

```html
<div>
  {% if not is_index %}
    {% include 'custome-cc.swig' %}
  {% endif %}
</div>
```

### 3.9 动态背景

next主题集成了集中动态背景效果，如下所示

1) canvas_nest

next 5.1.1以上可以直接在主题配置文件中将`canvas_nest: false`改成`canvas_nest: true`，随后下载动态背景js即可

参考链接：[传送门](https://github.com/theme-next/theme-next-canvas-nest)

下载代码带主题的lib库中

```shell
git clone https://github.com/theme-next/theme-next-canvas-nest themes/next/source/lib/canvas-nest
```

随后使能`canvas_nest`即可，随后生成看效果，有几个参数可以自己修改`themes/next/source/lib/canvas-nest/canvas-nest.min.js`

| 参数    | 含义                                                     |
| :------ | :------------------------------------------------------- |
| zIndex  | 背景的z-index属性，css属性用于控制所在层的位置，默认：-1 |
| opacity | 线条透明度（0~1）, 默认：0.5                             |
| color   | 线条颜色, 默认: ‘0,0,0’；三个数字分别为(R,G,B)           |
| count   | 线条的总数量, 默认: 99                                   |

2) 3-D 效果

参考链接：[传送门](https://github.com/theme-next/theme-next-three)

同样是下载相应的js脚本开启开关即可，非常简单

```shell
git clone https://github.com/theme-next/theme-next-three themes/next/source/lib/three
```

随后只需要开启单个效果开关

```yaml
# JavaScript 3D library.
# Dependencies: https://github.com/theme-next/theme-next-three
# three_waves
three_waves: false
# canvas_lines
canvas_lines: true
# canvas_sphere
canvas_sphere: false
```

### 3.10 自定义样式修改

#### 3.10.1 文章内链接样式

修改`themes/next/source/css/_custom/custom.styl`，在末尾添加如下css样式

```css
// 文章内链接文本样式
.post-body p a{
  color: #0593d3;
  border-bottom: none;
  border-bottom: 2px solid #0593d3;
  &:hover {
    color: #fc6423;
    border-bottom: none;
    border-bottom: 1px solid #fc6423;
  }
}
```

颜色可以自定义修改。

#### 3.10.2 文章内``样式

修改`themes/next/source/css/_custom/custom.styl`，添加以下内容

```css
// Custom styles.
code {
    color: #ff7600;
    background: #fbf7f8;
    margin: 2px;
}
// 大代码块的自定义样式
.highlight, pre {
    margin: 5px 0;
    padding: 5px;
    border-radius: 3px;
}
.highlight, code, pre {
    border: 1px solid #d6d6d6;
}
```

#### 3.10.3 主页文章添加阴影效果

修改`themes/next/source/css/_custom/custom.styl`，添加以下内容

```css
// 主页文章添加阴影效果
 .post {
   margin-top: 60px;
   margin-bottom: 60px;
   padding: 25px;
   -webkit-box-shadow: 0 0 5px rgba(202, 203, 203, .5);
   -moz-box-shadow: 0 0 5px rgba(202, 203, 204, .5);
  }
```

### 3.11 侧边栏社交小图标设置

打开主题配置文件`themes/next/_config.yml`，搜索`Social`，在[图标库](http://fontawesome.io/icons/)找自己喜欢的小图标，并将名字复制在`||`之后即可。

> GitHub: https://github.com/louielong || github

### 3.12 顶部加载条特效

next的主题中已经预置的相关配置，仅仅只需要打开相应的开关并下载lib库即可，方法参见：[pace特效传送门](https://github.com/theme-next/theme-next-pace)

克隆代码带主题的lib库中

```shell
git clone https://github.com/theme-next/theme-next-pace themes/next/source/lib/pace
```

随后修改`themes/next/_config.yml`，将`pace: false`改为`pace: true`，然后选择满意的特效选项即可。

> pace_theme: pace-theme-flash

### 3.13 开启打赏功能

仅需要开启打赏功能并上传二维码即可

修改`themes/next/_config.yml`中的`Reward`字段下开启一个标签，并上传图二维码至`source/uploads`或者`themes/next/source/images`目录下，并相应的修改路径即可。

开启后发现打赏字体闪动很频繁碍眼，可以注释掉，`themes/next/source/css/_common/components/post/post-reward.styl`，如下注释掉或者修改 roll后的参数值，降低闪动频率。

```css
/* 注释文字闪动函数
#wechat:hover p{
    animation: roll 0.1s infinite linear;
    -webkit-animation: roll 0.1s infinite linear;
    -moz-animation: roll 0.1s infinite linear;
}
#alipay:hover p{
    animation: roll 0.1s infinite linear;
    -webkit-animation: roll 0.1s infinite linear;
    -moz-animation: roll 0.1s infinite linear;
}
#bitcoin:hover p {
    animation: roll 0.1s infinite linear;
    -webkit-animation: roll 0.1s infinite linear;
    -moz-animation: roll 0.1s infinite linear;
}
*/
```

### 3.14 添加RSS订阅

安装RSS订阅插件

```shell
npm install --save hexo-generator-feed
```

搜索`rss`字样，添加`rss: /atom.xml`，重新`hexo g`生成一遍即可。

### 3.15 添加SEO搜索

本来想添加SEO加速搜索的，但是登录到**百毒**上一看需要这么多信息，果断放弃了，顺带给一个鄙视的眼神。

![百度SEO注册](https://i.imgur.com/Vq98g8w.jpg)

### 3.16 自定义默认生成文章头部模板

当新创建文章是希望可以默认生成一些头部字段，修改`scaffolds/post.md`文件增加一些自定义填充字段。

```markdown
---
title: {{ title }}
date: {{ date }}
tags:
keywords:
categories:
description:
summary_img:
---
```

使用`hexo new test`创建文章时则会自动填充相应信息，如下图所示

![文章模板](https://i.imgur.com/NSYFNAp.jpg)

若想在主页显示文章的总结图片，在`themes/next/layout/_macro/post.swig `中的`post.description`前加入如下内容

```jinja2
{% if post.summary_img  %}
  <div class="out-img-topic">
   <img src={{ post.summary_img }} class="img-topic">
  </div>
{% endif %}
```

重新生成博客即可看到文章缩略图。

### 3.17 博客字体修改

博客的字体配置在`themes/next/_config.yml `找到`Font Settings`，修改对应字段可以配置相应的字体效果。默认使用的是[谷歌的字体](https://www.google.com/fonts)。

如我想修改代码段的字体格式

```yaml
# Font settings for <code> and code blocks.
codes:
  external: true
  family: PT Mono
  size: 14
```

### 3.18 显示网站副标题

Next的Mist主题默认隐藏了副标题的显示，通过修改`themes/next/source/css/_schemes/Mist/_logo.styl`可以打开副标题的显示。

```css
.site-subtitle { display: yes; }
```

### 3.19 文章图片居中

修改Next主题文件`themes/next/source/css/_schemes/Mist/_posts-expanded.styl`，找到` .posts-expand`中的`.post-body img`加入`auto`字段。

```css
  .post-body img { margin: 0, auto; }
```

### 3.20 点击爱心效果

在`themes/next/source/js/src`下新建文件`clicklove.js`，随后将[链接](http://7u2ss1.com1.z0.glb.clouddn.com/love.js)下的代码拷贝粘贴到`clicklove.js`文件中

在`themes/next/layout/_layout.swig`文件末尾添加：

```
<!-- 页面点击小红心 -->
<script type="text/javascript" src="/js/src/clicklove.js"></script>
```

![点击爱心](https://i.imgur.com/7hCt0FX.jpg)

### 3.21 博客背景图片

统一修改next预留的自定义文件`themes/next/source/css/_custom/custom.styl`

1)静态图片

```javascript
body {
    background:url(https://XXX);
}
```

设置静态背景图片

2)动态图片

**unsplash**网站提供大量高清图片，可随机选择

```css
body {
    background:url(https://source.unsplash.com/random/1600x900);
    background-repeat: no-repeat;
    background-attachment:fixed;
    background-position:50% 50%;
}
```

不透明度设定

```css
.main-inner {
    margin-top: 60px;
    padding: 60px 60px 60px 60px;
    background: #fff;
    opacity: 0.8;
    min-height: 500px;
}
```

- background: #FFF  代表白色背景色
- opacity： 0.8  代表不透明度

### 3.22 侧边栏背景修改

给侧边栏添加背景图片，只需要修改`themes/next/source/css/_common/components/sidebar/sidebar.styl`的`background`的值即可，同时为了让图片平铺可以增加一些设置。同样的，图片可以是在线的也可以是本地。

```css
background: url('/uploads/sidebar.jpg');
box-shadow: none;
background-repeat: round;
```

【Note】

由于使用的黑色的背景，这里将侧边栏的字体设置为白色，需要修改一些配置文件

1)`themes/next/source/css/_common/components/sidebar/sidebar-toc.styl`的Line 23，此处影响的是`文章目录`字体颜色

```css
color: $whitesmoke;
```

2)修改`themes/next/source/css/_variables/base.styl`的 line 265，此处影响的是`站点概览`的字体颜色

```css
$sidebar-nav-color                    = $whitesmoke
```

