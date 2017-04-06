---
title: Algolia Serach
date: 2017-03-27 18:48:18
categories:
  - 学习笔记
tags:
  - Hexo
---
静态网页的站点如果需要评论或搜索相关的功能，需要借助第三方服务。常用的评论服务有多说、网易云跟帖。由于多说服务平台六月将要关闭，因此推荐网易云跟帖服务。至于搜索服务，之前用的Swiftype站内搜索，一直有问题，索性直接换个平台Algolia。
使用起来还可以界面也挺搭配，这里就主要介绍下Algolia搜索服务的应用。
<!-- more -->
>注：该文章Algolia的使用是基于主题nexT5.1.0版本,5.1以下版本不适合此方式，其他hexo主题请参照Algolia官网文档。

- 效果如下

<img src="http://ofywot861.bkt.clouddn.com/image/algolia/algolia.png" class="full-image" />

### 第一步 注册algolia账号

- [官网链接](https://www.algolia.com/)注册账号

可以使用 GitHub 或者 Google 账户直接登录，注册后的 14 天内拥有所有功能（包括收费类别的）。之后若未续费会自动降级为免费账户，免费账户 总共有 10,000 条记录，每月有 100,000 的可以操作数。注册完成后，创建一个新的 Index，这个 Index 将在后面使用。

- 新建个INDEX如图

![Algolia INDEX](http://ofywot861.bkt.clouddn.com/image/algolia/algolia-step-2.png)

- 获取key

在 Algolia 服务站点上找到需要使用的一些配置的值，包括 ApplicationID、Search API Key、 Admin API Key。注意，Admin API Key 需要保密保存。

![Algolia KEY](http://ofywot861.bkt.clouddn.com/image/algolia/algolia-result.png)


### 第二步 本地配置algolia

- 安装  hexo-algolia，在项目根目录执行：

```
npm install hexo-algolia --save
```

- 在__站点根目录__的_config.yml中新增如下配置，改成前面第一步获取key数据

```
algolia:
  applicationID: 'your applicationID'
  apiKey: 'your apiKey'
  adminApiKey: 'your adminApiKey'
  indexName: 'your indexName'
  chunkSize: 5000
```

### 第三步 更新Index

```
hexo algolia
```
![hexo algolia](http://ofywot861.bkt.clouddn.com/image/algolia/algolia-step-4.png)

***

{% note primary %} ### 后记 {% endnote %}

- 如果在执行hexo algolia出现下面错误时 *Plugin load failed: hexo-algolia*

建议删除hexo-algolia module 使用指定版本命令安装：

```
npm install hexo-algolia@0.2.0
```

- 如果出现 *Cannot find module XXX 错误*

版本 > 3 依然报错： 请先删除 node_module 目录，然后使用 npm install 重新安装一下模块。

版本 < 3： 您可以选择升级您的 NPM； 或者在站点目录下明确指定模块依赖 npm install --save hexo-util。 其中 `hexo-util` 仅是示例，请替换成错误中提示的模块名称。
