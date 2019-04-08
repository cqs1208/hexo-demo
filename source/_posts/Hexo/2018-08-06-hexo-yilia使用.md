---
layout: post
title: hexo-yilia使用
tags:
- Hexo
categories: Hexo
description: hexo-yilia使用
---

本文介绍hexo的搭建，并下载更新yilia主题示例

<!-- more --> 

**注：确保安装了node和git**

### 1 安装hexo

1. 先创建一个文件夹（用来存放所有blog的东西），然后`cd`到该文件夹下。
2. 安装hexo命令：npm i -g hexo
3. 初始化命令：`hexo init` ，初始化完成之后打开所在的文件夹可以看到以下文件： 

文件解释：

- node_modules：是依赖包
- public：存放的是生成的页面
- scaffolds：命令生成文章等的模板
- source：用命令创建的各种文章
- themes：主题
- _config.yml：整个博客的配置
- db.json：source解析所得到的
- package.json：项目所需模块项目的配置信息

### 2 搭桥到github

- 搭建好自己的远程git仓库并和本地git客户端联通

- 用编辑器打开你的blog项目，修改`_config.yml`文件的一些配置(冒号之后都是有一个半角空格的)：

  ```java
  deploy:
    type: git
    repo: https://github.com/YourgithubName/YourgithubName.github.io.git
    branch: master
  ```

- 回到gitbash中，进入你的blog目录，分别执行以下命令：

  ```java
  hexo clean
  hexo generate
  hexo server
  ```

  注：hexo 3.0把服务器独立成个别模块，需要单独安装：`npm i hexo-server`。 

- 打开浏览器输入：`http://localhost:4000`  。 至此本地ok

### 3 修改及配置主题

- hexo初始化之后默认的主题是`landscape` , 然后你可以去[这个地址](https://hexo.io/themes/)里面找到你想要的主题。在github中搜索你要的主题名称，里面都会有该主题的如何使用的介绍  本文选的是`yilia`

- 在当前项目页,输入以下命令: 

  ```java
  git clone https://github.com/litten/hexo-theme-yilia.git themes/yilia
  ```

- 回到gitbash中，进入你的blog目录， 编辑_config.yml,  找到theme项，theme后面的内容修改为yilia（初始化为：landscape） （不是themes/yilia下的__config.yml文件）

  ```java
  # Extensions
  ## Plugins: https://hexo.io/plugins/
  ## Themes: https://hexo.io/themes/
  theme: yilia
  
  # Deployment
  ## Docs: https://hexo.io/docs/deployment.html
  #coding: git@git.coding.net:JansZeng/hexoblog.git,master
  ```

- 然后，重新编译，启动： 

  ```java
  hexo clean
  hexo g
  hexo s  # 本地启动
  hexo d  # 推到远程git服务
  ```

  注：如果出现：

  ```java
  ERROR Deployer not found: git
  ```

  执行以下命令: 

  ```java
  npm install hexo-deployer-git --save
  ```

  然后在执行：

  ```java
  hexo d
  ```

### 4 Hexo yilia 主题一揽子使用方案

#### 4.1 查看所有文件，提示缺失模块

`yilia` 在首次使用时，点击`所有文章` 时，会出现模块找不到的错误，可按照提示操作即可  注意一下，_config.yml 路径是指 根目录下的，而非 `yilia` 主题下的 config文件 

![hexo缺失模块](/images/Hexo/Hexo_hexo.png)

模块缺失：

1. 请确保node版本大于6.2

2. 回到gitbash中，进入你的blog目录，执行

   ```java
   npm i hexo-generator-json-content --save
   ```

3. 在blog目录下_config.yml里添加配置

   ```java
   jsonContent:
     meta: false
     pages: false
     posts:
       title: true
       date: true
       path: true
       text: false
       raw: false
       content: false
       slug: false
       updated: false
       comments: false
       link: false
       permalink: false
       excerpt: false
       categories: false
       tags: true
   ```

#### 4.2 配置图片资源

- **添加图片资源文件夹**。 路径为 `themes/yilia/source/`下，可添加一个 `assets` 文件夹，里面存放图片资源即可 

- **配置文件中直接引用即可**。路径为 `themes/yilia/_config.yml`，找到如下即可 

  ```java
  # 微信二维码图片
  weixin:  /assets/img/wechat.png
  
  # 头像图片
  avatar:  /assets/img/head.jpg
  
  # 网页图标
  favicon:  /assets/img/head.jpg
  ```

#### 4.3 文章如何显示摘要

- **问题**。点击主页时，发现所有文章都是全文显示，不利于查找，可控制显示的字数

- **解决办法**。 在你 MD 格式文章正文插入 `<!-- more -->`即可，只会显示它之前的，此后的就不显示，点击文章标题，全文阅读才可看到，同时注释掉以下 `themes/yilia/_config.yml`，重复 

  ```java
  # excerpt_link: more
  ```

  ![3_2_yilia_摘要](/images/Hexo/Hexo_hexo2.png)

#### 4.4 文章显示目录

增加文章目录 TOC(table of content )，方便阅读文章, 在 `themes/yilia/_config.ym`中进行配置 `toc: 2`即可，它会将你 Markdown 语法的标题，生成目录，目录查看在右下角。 

![3_3_yilia_目录](/images/Hexo/Hexo_hexo3.png)

#### 4.5 增加归档菜单

修改 `themes/yilia/_config.yml` 

```java
menu:
    主页:  /
    归档:  /archives/index.html
```



