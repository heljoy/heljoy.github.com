---
layout: post
title: "Install and configure Hexo&Next for blog"
description: "Record another simple style blog system setup"
category: blog 
tags: [blog, setup, hexo]
---
{% include heljoy/setup %}

昨天在查找Linux内存管理方面的文章时碰到一篇介绍[伙伴算法的分析](http://labrick.xyz/2015/10/12/buddy-system-algorithm/)的blog, 文章主要讲解了算法的原理与用途, 后面分析了一个[简洁版本](https://github.com/wuwenbin/buddy2)代码实现, 写得很紧凑易懂, 但我更喜欢他的blog风格.


找到这个blog是[Hexo](https://hexo.io/)框架搭建的, [Next](https://github.com/iissnan/hexo-theme-next)主题风格, 后面自己根据网络上的一些资料与框架和主题的文档尝试再新建一个blog, 本文记录一下具体的过程备忘.

<!-- more -->

### 安装Hexo框架

Hexo是基于[Node.js](http://nodejs.cn/)实现的一个轻量级快速静态网页生成工具, 安装Hexo要先有Node.js环境. 我是Ubuntu14.04系统, 在Node.js的官网直接下载Ubuntu的稳定版或LTS版都可以, 我用的是[Node-4.4.3](https://nodejs.org/dist/v4.4.2/node-v4.4.2-linux-x64.tar.gz),没有使用源码包编译.

```bash

$ tar xvf node-v4.4.3-linux-x64.tar.xz
$ cd node-v4.4.3-linux-x64/
$ cd lib/node_modules/npm/
$ ls
AUTHORS       cli.js           doc   LICENSE   man           README.md
bin           configure        html  make.bat  node_modules  scripts
CHANGELOG.md  CONTRIBUTING.md  lib   Makefile  package.json  wercker.yml
$ sudo make install

```

由于node安装包是编译过的, 直接`make install`安装就行了, 默认安装目录在/usr/local, 需要加上root权限才行. `npm`相当于Ubuntu的apt-get工具, 管理基于Node.js开发的各种框架, Hexo就是其中一种,安装好后确认一下.

```bash

$ node -v
v4.4.3
$ npm -v
2.15.1

```

使用包管理工具npm安装hexo.

```bash

$ npm install -g hexo-cli
$ npm install hexo --save

```
hexo安装很简单, 后面使用就更简单了, 哈哈, 这么简单的工具, 以后再也没有理由偷懒不写blog了. 确认hexo安装成功.

```bash

$ hexo -v
hexo-cli: 1.0.1
os: Linux 3.13.0-24-generic linux x64
http_parser: 2.5.2
node: 4.4.3
v8: 4.5.103.35
uv: 1.8.0
zlib: 1.2.8
ares: 1.10.1-DEV
icu: 56.1
modules: 46
openssl: 1.0.2g

```

### 安装Next主题

前面只说到安装hexo框架工具, 并没有创建blog, 不急, 一行命令的事, 所以就放到与主题安装一起介绍了. 首先新建一个blog系统工作目录`hexo`并初始化.

```bash

$ mkdir hexo
$ cd hexo
$ hexo init
$ ls
_config.yml  node_modules  package.json  scaffolds  source  themes

```

上面`hexo init`在当前目录下创建了blog的默认文件与目录结构,我们重点关注下面几个目录与文件.

* `_config.yml`: 站点配置文件, 用于配置整个站点参数, 比如主题,标题,作者信息,文章标题等等
* `node_modules`: 包含用于生成站点静态文件的node模块, 后面会给出必要模块的安装命令
* `scaffolds`: 模板文件目录, 创建post/page页面时参考的模板, 一些公用的设置可以放在模板文件里
* `source`: 源文件目录, 我们用markdown写的文件都在这里, 也是主要的更新目录
* `themes`: 主题目录, 后面使用的Next主题也是放在这个目录下面

在安装主题前再准备一些必要的hexo工具, 包含生成RSS文件功能/支持GIT发布等等, 部分工具在`hexo init`时有安装过, 都在`node_modules`目录下, 这里可以不用再安装.

```bash

$ npm install hexo-generator-index --save
$ npm install hexo-generator-archive --save
$ npm install hexo-generator-category --save
$ npm install hexo-generator-tag --save
$ npm install hexo-server --save
$ npm install hexo-deployer-git --save
$ npm install hexo-renderer-marked@0.2 --save
$ npm install hexo-renderer-stylus@0.2 --save
$ npm install hexo-generator-feed@1 --save
$ npm install hexo-generator-sitemap@1 --save

```

建立好了hexo基本的结构后, 使用下面命令安装Next主题.

```bash

git clone https://github.com/iissnan/hexo-theme-next themes/next

```

替换hexo的默认主题为next.

```bash

$ sed -i '/^theme/s/landscape/next/g' _config.yml
$ sed -n '/^theme/p' _config.yml
theme: next

```

### 配置站点

新建tags/categories/about页面, 分别对应标签/分类/关于. 

```bash

$ hexo new page categories
$ hexo new page tags
$ hexo new page about
$ tree source 
source
├── about
│   └── index.md
├── categories
│   └── index.md
├── _posts
│   └── hello-world.md
└── tags
    └── index.md

    4 directories, 4 files

```

分别在`source`目录新建了三个对应的目录, 并且各自下面还新建了一个`index.md`文件, 该文件的layout以`scaffolds/page.md`为模板, 一些公用的属性可以加在模板里, 比如是否允许评论.

```bash

$ cat scaffolds/page.md 
---
title: {{ title }}
date: {{ date }}
comments: false
---

```

根据[next主题配置教程]()设置其他参数,比如代码高亮方案\侧边栏\评论\站点统计等第三方服务.


### 新建blog页面


我们使用站点与主题的默认配置, 为简单也不修改基本信息.新建一篇blog测试.

```bash

$ hexo new "my first hexo blog"
$ cat source/_posts/my-first-hexo-blog.md 
---
title: my first hexo blog
date: 2016-04-26 15:39:41
tags:
---

$ cat <<EOF >> source/_posts/my-first-hexo-blog.md 

Just say:"hello!"
more info at [my home](heljoy.github.com)
Bye!!
EOF

```

启动服务器预览我们的站点, 在浏览器中输入[local](http://localhost:4000/)访问站点.

```bash

$ hexo s --debug

```

生成站点的静态网页文件, 目录`public`就是我们推送到远程服务器的网站主目录.

```bash
$ hexo g
$ ls public 
2016  about  archives  categories  css  images  index.html  js  tags  vendors

```

### End

* Hexo是一个轻量级的框架, 生成blog静态文件快速方便
* Next这个主题我比较喜欢, 是因为这个主题我才使用这个框架的
* Hexo环境配置,文章生成/预览都很方便,包括还能一条命令发布到github上.
