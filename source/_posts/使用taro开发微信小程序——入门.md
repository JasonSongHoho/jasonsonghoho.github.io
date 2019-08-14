---
title: 使用taro开发微信小程序——入门
date: 2019-08-13 18:30:32
tags:
- taro
- 微信小程序
---

## 简介
### taro 是什么
Taro 是一套遵循 React 语法规范的 多端开发 解决方案。现如今市面上端的形态多种多样，Web、React-Native、微信小程序等各种端大行其道，当业务要求同时在不同的端都要求有所表现的时候，针对不同的端去编写多套代码的成本显然非常高，这时候只编写一套代码就能够适配到多端的能力就显得极为需要。

使用 Taro，我们可以只书写一套代码，再通过 Taro 的编译工具，将源代码分别编译出可以在不同端（微信/百度/支付宝/字节跳动/QQ小程序、快应用、H5、React-Native 等）运行的代码。

### 支持多端开发转化
Taro 方案的初心就是为了打造一个多端开发的解决方案。目前 Taro 代码可以支持转换到 微信/百度/支付宝/字节跳动/QQ小程序 、快应用、 H5 端 以及 移动端（React Native）。

{% asset_img  platforms.jpg %}


## quick start
### 安装 taro cli

```
# 使用 npm 安装 CLI
$ npm install -g @tarojs/cli
# OR 使用 yarn 安装 CLI
$ yarn global add @tarojs/cli
# OR 安装了 cnpm，使用 cnpm 安装 CLI
$ cnpm install -g @tarojs/cli

```

### demo 工程
可以网上下载一个 taro 的 demo 工程，我们以 [taro-library](https://github.com/imageslr/taro-library) 为例。

```
$ git clone https://github.com/imageslr/taro-library.git

$ cd taro-library

$ npm install 或者 yarn

$ npm run dev:weapp

// 新建一个终端，在项目根目录下执行
$ gulp mock

```

需要注意的是，最好使工程的依赖与 cli 版本一致，否则运行时可能会出现一些奇怪的错误。

{% asset_img  taro-version.jpg %}


### 导入 微信开发者工具

执行完 ``npm run dev:weapp`` 之后，将会在项目的 dist 文件夹中生成相应的小程序工程。

安装好 [微信开发者工具](https://developers.weixin.qq.com/miniprogram/dev/devtools/download.html) 之后启动，选择导入项目。**目录**选择 taro-library 的 dist 文件夹，**AppID**选择测试号。

{% asset_img  new-weapp.jpg %}

### 调试

{% asset_img  debug-weapp.jpg %}



### easy-mock
数据是在本机mock的，如果想要真机调试。可以使用我司的 [easy-mock](https://easy-mock.com/) 来 mock 在线数据，使用方便。

{% asset_img  easy-mock.jpg %}




















