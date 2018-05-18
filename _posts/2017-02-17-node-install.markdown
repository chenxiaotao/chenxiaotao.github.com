---
layout:     post
title:      Node.js 环境搭建
subtitle:   "node的基础环境搭建"
date:       2017-02-16
author:     "Sheldon"
header-img: "img/pic/post/new-nodejs.jpg"
tags:       
  - javascript
  - 环境搭建
---

### 一、前言
近期由于工作需要，开始需求完成一些nodejs的工作，公司也慢慢转型Java + nodejs的技术工作模式，在此记录下自己node环境的搭建过程，后续会慢慢添加一些自己在nodejs上学习到的东西和问题

### 二、版本管理
我所使用的node版本管理是 [n](https://github.com/tj/n) ，当然也有很多人使用 [nvm](https://github.com/creationix/nvm) ，网上也有很多关于 nvm 和 n 的对比。我也没有纠结太多，感觉n更加的简洁，故直接使用了 n ，下面简单的介绍一下 n 的用法

n 是一个需要全局安装的 npm package `npm install -g n`

这意味着，我们在使用 n 管理 node 版本前，首先需要一个 node 环境。我们或者用 Homebrew 来安装一个 node `brew install node`，因为n本身是没法给你装的

安装完成之后，直接输入n后输出当前已经安装的node版本以及正在使用的版本（前面有一个o），你可以通过移动上下方向键来选择要使用的版本，最后按回车生效

~~~
$ n
    node/4.3.0
  ο node/6.9.5
    node/7.4.0
~~~

* 如果你要安装其他的版本（比如4.4.1），那么使用：`n 4.4.1`
* 安装最新的版本：`n latest`
* 安装稳定版本：`n stable`
* 删除某个版本：`n rm 4.4.1`
* 以指定的版本来执行脚本：`n use 4.4.1 demo.js`

### 三、npm源管理
由于公司内部搭建了一套 NPM 源，为了灵活控制各个源，发现了 [nrm](https://github.com/Pana/nrm) 这个东西，nrm 是一个 NPM 源管理器，允许你快速地在如下 NPM 源间切换。使用 `npm install -g nrm` 全局安装一个 nrm
* 列出可选源 `nrm ls` 带 * 号的是当前使用的源，如下都是官方源

~~~
$ nrm ls
* npm ---- https://registry.npmjs.org/
  cnpm --- http://r.cnpmjs.org/
  taobao - http://registry.npm.taobao.org/
  eu ----- http://registry.npmjs.eu/
  au ----- http://registry.npmjs.org.au/
  sl ----- http://npm.strongloop.com/
  nj ----- https://registry.nodejitsu.com/
~~~
* 切换 `nrm use taobao`
* 增加源 `nrm add  <registry> <url>` 例： `nrm add  test 'http://npm.test.com'`
* 删除源 `nrm del test`
* 还可以测试速度，测试所有源: `nrm test`， 测试单个源： `nrm test npm`
* 详细内容可见 `nrm --help`
