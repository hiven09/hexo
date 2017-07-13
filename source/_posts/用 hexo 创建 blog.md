---
title: 用 hexo 创建 blog
date: 2017-02-18 12:00:00
tags: "hexo"
categories: "技术"
---


## 安装Hexo

### 准备工作
hexo 的官方文档 ,[地址](https://hexo.io/zh-cn/docs/index.html)

安装 Hexo 相当简单。然而在安装前，您必须检查电脑中是否已安装下列应用程序：
 - [git](http://git-scm.com/)
 - [Node.js](http://nodejs.org/)
 
### Hexo 安装部署

- 打开hexo 执行安装命令

``` bash
$ npm install hexo-cli -g
$ hexo init blog
$ cd blog
$ npm install
$ hexo clean
$ hexo server
$ hexo generate
```
- 部署到github上

``` bash
$ hexo deploy
```

### 遇到的坑
- 配置文件中“：”后面要带空格,否则报错
- 搜索开启本地搜索才会出现搜索框
``` bash
	local_search:
		enable: true
```
- 修改空心变实心

``` bash
source/css/_common/components/post/post-expand.styl
  ul li { list-style: disc; }
```
## 参考文章
[官方文档:](https://hexo.io/zh-cn/docs/server.html)https://hexo.io/zh-cn/docs/index.html
[配置相关:](http://blog.aiyouweiya.xyz/posts/3077/)