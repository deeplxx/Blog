---
title: Ubuntu16搭建个人博客(hexo+git+vps)
categories: 配置环境
tags:
- 配置环境
- Web
---

## 本地
1. 安装Hexo
    - hexo依赖于Nodejs和Git，需要先安装依赖
        - [安装nodejs](http://www.linuxdiyf.com/linux/31680.html)
        - 安装Git并配置:
        ```
        $ sudo apt install git
        $ git config --global user.name "example_name"
        $ git config --global user.email "email@example.com"
        ```
    - 安装hexo：
        ```
        $ npm install hexo-cli -g
        ```
    - 自定义一个Blog目录，hexo初始化并安装依赖：
        ```
        $ hexo init blog
        $ cd blog
        $ npm install
        ```
    - 完成后进行本地测试：
        ```
        $ hexo g # 渲染 Source 目录下文件问静态页面
        $ hexo s # 本地跑一个 server 来看博客效果
        ```
        然后可以在`http://localhost:4000/`上查看效果
    - 这里有必要提下Hexo常用的几个命令：
        - hexo generate (hexo g) 生成静态文件，会在当前目录下生成一个新的叫做public的文件夹
        - hexo server (hexo s) 启动本地web服务，用于博客的预览
        - hexo deploy (hexo d) 部署播客到远端（比如github, heroku等平台）
        - hexo new "**" 新建文章
        - hexo new page “**” 新建页面

## 服务器
1. [安装nginx并配置web站点](https://www.jianshu.com/p/998eeb56aa6c)
2. 部署hexo到服务器：
    - 在vps上安装git
    - 创建空git仓库并配置hook
        ```
        $ mkdir blog && cd blog
        $ git init --bare
        ```
        在 `/blog/hooks` 中添加脚本文件`post-receive`
        ```
        #!/bin/bash
        GIT_REPO=/root/blog  #git仓库目录
        TMP_GIT_CLONE=/tmp/blog
        PUBLIC_WWW=/path/to/web/root #网站目录
        rm -rf ${TMP_GIT_CLONE}
        git clone $GIT_REPO $TMP_GIT_CLONE
        rm -rf ${PUBLIC_WWW}/*
        cp -rf ${TMP_GIT_CLONE}/* ${PUBLIC_WWW}
        ```
        赋予脚本执行权限
        ```
        $ chmod +x post-receive
        ```

## 将本地渲染网页到服务器
1. 在`blog`目录下安装git部署工具：
    ```
    $ npm install hexo-deployer-git --save
    ```
2. 修改‘blog’的配置文件`_config.yml`：
    ```
    deploy:
    type: git
    message: update
    repo:
        s1: root@**.**.**.**:/root/blog.git/, master  # **为vps的ip地址
    ```
3. 执行部署
    ```
    $ hexo d
    ```

<font size=6>[发布文章，更换主题](https://linghucong.js.org/2016/04/15/2016-04-15-hexo-github-pages-blog/)</font>
