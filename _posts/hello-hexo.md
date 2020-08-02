---
title: hello-hexo
date: 2019-11-18 00:27:11
tags:
 - hexo
 - github.io

categories:
 - hexo

---

## 前言
> 之前陆陆续续又在写博客，最近几个月以来是越来越懒了，博客也许久未写了，技术相关的东西也没有学多少。应该是到了醒悟的时候，同时这次我把每周一篇博客写进我个人成长的OKR中。

> 作为一枚理工直男，在文字写作方面的能力确实会差了一些。每每在B站看UP主的电影解说的时候，是真的羡慕他们的文字和语言表达能力。就像是以前上学的时候，他们就是经常作文满分的同学，而我就是在及格线上徘徊的渣渣。
但是写写技术博客与个人心得，应该是勉强够用了。

> 鉴于折腾这个博客网上花了一些时间，也说不上折腾吧，其实就是自己的脑瓜子不太好使，所以这周的博客决定以弄这个博客网站水上一篇。见谅见谅。

<!-- more -->

## 正题

考虑到自己建站的麻烦和维护，所以决定把博客到挂到github.io。操作简单方便，而且hexo也有git插件可以一键部署到github.ios上。

### 所用到的工具

- github.io
- hexo
- hexo主题next
- 以及hexo插件

--- 

### github.io

- 这玩意是啥，百度一下就有大票答案了。总而言之就是在你的github的仓库上new一个跟你的username同名的{username}.github.io。
- 然后把静态的html文件提交到这个仓库中，然后就是访问了。贼简单。
- 建好这个仓库先放着，等待后续hexo生成的静态文件提交到这即可。
- 提交了html静态文件之后，就可以通过{username}.github.io 例如我的地址：wang007.github.io


### hexo
首先来说hexo是什么。这个可以直接从hexo白嫖过来。
```
快速、简洁且高效的博客框架 - hexo官网
```
很显然，这个官网没有说人话。
```
hexo 可以理解为是基于node.js制作的一个博客工具，不是我们理解的一个开源的博客系统。其中的差别，有点意思。
hexo 正常来说，不需要部署到我们的服务器上，我们的服务器上保存的，其实是基于在hexo通过markdown编写的文章，然后hexo帮我们生成静态的html页面，然后，将生成的html上传到我们的服务器。简而言之：hexo是个静态页面生成、上传的工具。
白嫖自：https://www.jianshu.com/p/1c888a6b8297
```
上面这段描述是很准确的了。
- hexo只是一个基于node.js的博客制作工具，而不是第一影响理解的blog server。
- 用户通过hexo编写md文件，然后生成html相关的静态文件上传到自己的静态服务器上。

#### 安装hexo
hexo基于node.js，同时又需要git来拉取代码，所以先安装node.js,npm,git
node.js,npm,git安装过程自行搜索，网上大堆资料。

安装好node.js,npm,git之后，安装hexo-cli，没有安装淘宝镜像源的话 =>   cnpm -> npm
```
cnpm install -g hexo-cli
```
然后创建一个blog（目录名随意）目录，位置随意。例如：自己的home目录
```
mkdir blog
```
然后进行目录
```
cd blog
```
执行hexo init命令
```
hexo init
```
不知道是不是只有我这的问题，会报一些超时的错误，当时这里卡了好久。当时网上搜了很久，都没有找到答案。然后直接操作下一个命令。cnpm install，直接就行了，接就行了，就行了，行了，了。
所以这里的错误可以先不用理会，直接操作下一个命令。我当时脑瓜子有毛病，在这一步卡了很久。哎，脑子真是个好东西。
```
hexo init
INFO  Cloning hexo-starter https://github.com/hexojs/hexo-starter.git
正克隆到 '/Users/see/bl'...
remote: Enumerating objects: 22, done.
remote: Counting objects: 100% (22/22), done.
remote: Compressing objects: 100% (17/17), done.
remote: Total 153 (delta 8), reused 9 (delta 3), pack-reused 131
接收对象中: 100% (153/153), 29.67 KiB | 394.00 KiB/s, 完成.
处理 delta 中: 100% (70/70), 完成.
子模组 'themes/landscape'（https://github.com/hexojs/hexo-theme-landscape.git）已对路径 'themes/landscape' 注册
正克隆到 '/Users/see/bl/themes/landscape'...
remote: Enumerating objects: 32, done.        
remote: Counting objects: 100% (32/32), done.        
remote: Compressing objects: 100% (25/25), done.        
remote: Total 1054 (delta 20), reused 10 (delta 7), pack-reused 1022        
接收对象中: 100% (1054/1054), 3.21 MiB | 97.00 KiB/s, 完成.
处理 delta 中: 100% (578/578), 完成.
Submodule path 'themes/landscape': checked out '73a23c51f8487cfcd7c6deec96ccc7543960d350'
INFO  Install dependencies
yarn install v1.19.0
info No lockfile found.
[1/4] 🔍  Resolving packages...
warning hexo > nunjucks > chokidar > fsevents@1.2.9: One of your dependencies needs to upgrade to fsevents v2: 1) Proper nodejs v10+ support 2) No more fetching binaries from AWS, smaller package size
[2/4] 🚚  Fetching packages...
error Incorrect integrity when fetching from the cache
info Visit https://yarnpkg.com/en/docs/cli/install for documentation about this command.
WARN  Failed to install dependencies. Please run 'npm install' manually!
```
前面说了，不管这里报错，直接先下一个命令试试。
```
npm install
```
没问题，提示成功之后，然后测试hexo是否安装成功，启动hexo s
```
hexo s
INFO  Start processing
INFO  Hexo is running at http://localhost:4000 . Press Ctrl+C to stop.
```
如上提示信息，说明hexo已经安装好了，可以访问http://localhost:4000 看看效果。


#### hexo部署git插件
直接命令行执行
```
npm install hexo-deployer-git --save
```
然后进入到blog目录（hexo init所在的目录），编辑_config.yml文件，deploy在文件最后头。
```yaml
deploy:
  type: git  
  repo: git@github.com:wang007/wang007.github.io.git //your blog repos
  branch: master  //
```
然后ok搞定。
此时确保你的git和ssh是配置正确的。否则会失败。


先执行hexo g，
最后执行hexo d，发布到github.io上。
```
hexo d
```

关于网站的标题，描述等。也在_config.yml文件中。默认在文件开头这里
```yaml
# Site
title:  标题
subtitle: '小标题'
description: '网站简述'
keywords: java,asynchronous,high concurrency //网站关键字，可以有多个且必须用英文逗号间隔。
author: wang007 //作者
language: zh-CN //
timezone: 'Asia/Shanghai' //时区
```

### hexo主题
最后来说说hexo主题，主题这种东西，属于萝卜青菜，各有所爱。这里我选用的使用比较广泛的nexT主题。
```
# 在blog目录下执行
git clone https://github.com/iissnan/hexo-theme-next themes/next
# 打开blog目录下的 _config.yml，找到theme
theme: next
```
然后hexo clean，重新发布一下就能看到效果了。

--- 
next自己也有几个小主题，样式都差不多。next的主题叫做scheme。
```
# 进入themes/next目录下，也有一个叫_config.yml的文件。里面找scheme，然后进行设置
```

```
# Schemes
scheme: Muse
#scheme: Mist
#scheme: Pisces
#scheme: Gemini
```

默认情况下，首页标签路径只有Home和Archives。
```
menu:
  home: / || home
  #about: /about/ || user
  #tags: /tags/ || tags
  #categories: /categories/ || th
  archives: /archives/ || archive
  #schedule: /schedule/ || calendar
  #sitemap: /sitemap.xml || sitemap
  #commonweal: /404/ || heartbeat
```
可以把相应的注释打开。about，tags，categories。这几个最常用。
但是光这样来不行。还在blog目录下的sources创建对应的目录文件才行。

```
hexo new page "categories"
```
然后~/blog/source/categories目录下就会多一个index.md文件。在文件头添加如下所示。

```
---
title: categories
date: 2019-11-17 23:56:32
type: "categories" //一定添加这个
---
```
一定要添加这个type属性，不然就没有categories（分类）的效果。


tags，about也是如此。
```
hexo new page "tags"
hexo new page "about"
```

```
---
title: categories
date: 2019-11-17 23:56:32
type: "tags" //一定添加这个，才是标签的功能效果。
---
```

about也是差不多，就是描述关于我的内容。

至此，好像基本差不多了。
还有一个建议，就是把blog目录sources也放到github上，这样就能做一个备份。

=== 
update: 后续使用hexo图片遇到了图片不显示的问题，最后发现是插件有bug。
        这里记录一下：https://www.jianshu.com/p/3db6a61d3782


--- 
### 好了，时间不早了， 晚安。





--- 
参考链接：
  -  https://www.jianshu.com/p/3d2e7b3ec182
  











































