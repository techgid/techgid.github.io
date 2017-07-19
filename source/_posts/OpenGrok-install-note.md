---
title: Ubuntu 中配置 OpenGrok 服务
date: 2017-07-18 11:01:09
catalog: true
categories:
  - 学习笔记
tags:
  - OpenGrok
---

> OpenGrok作为代码检索服务器，十分方便我们阅读代码，搜索速度也是很快。作为浏览代码的服务还有[wobog](https://woboq.com/codebrowser.html),以下首先介绍OpenGrok的服务搭建，它是部署在tomcat服务器中，因此需要配置基本的tomcat环境。

<!-- more -->

### 安装jdk
* [Download jdk](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
* 配置全局变量

```
export PATH=$PATH:/home/jiajia/soft/android-sdk-linux/platform-tools
export JAVA_HOME=/home/jiajia/bin/java-8-openjdk-amd64
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH
```

### 安装tomcat

* [Download tomcat](http://tomcat.apache.org/download-80.cgi)
* 测试环境

```
sudo sh ./apache-tomcat-8.5.16/bin/configtest.sh
```
* 运行tomcat

```
sudo sh ./apache-tomcat-8.5.16/bin/startup.sh
```
* 测试配置

```
http://localhost:8080
```
如过出现tomcat首页，说明tomcat环境配置成功，下面我们需要部署OpenGrok.

### 部署openGrok
* [DownLoad openGrok](https://github.com/OpenGrok/OpenGrok/releases)

{% note warning %} 提示：
建议下载的是<font color=red bgcolor=orange>0.12.1.6 (stable)</font>，其他版本配置中可能会有问题，也可以去尝试最新版本。
{% endnote %}
* 解压

```
tar -xvzf ~/Downloads/opengrok-0.13-rc10.tar.gz -C ~/soft/
```

* 部署

```
sudo OPENGROK_TOMCAT_BASE=～/bin/apache-tomcat-8.5.16 ~/soft/opengrok-0.13-rc10/bin/OpenGrok deploy
```
此步骤会将source.war包放入tomcat webapps目录中。

* 创建project索引

```
OPENGROK_VERBOSE=true OPENGROK_INSTANCE_BASE=~/soft/opengrok-0.12.1.6 ./OpenGrok index ~/project/
```
其中**OPENGROK_INSTANCE_BASE**为存放数据库路径， ～/project 为需要建立索引的源码路径。
{% note primary %} 提示：如果有下面错误，需要安装 sudo apt-get install CTags
ERROR: Unable to determine Exuberant CTags command name for Linux 4.4.0-83-generic
FATAL ERROR: Missing Dependent Application - Exuberant CTags - Aborting!
{% endnote %}

* 修改etc/configuration.xml 路径

```
vi apache-tomcat-8.5.16/webapps/source/WEB-INF/web.xml

<context-param>
  <param-name>CONFIGURATION</param-name>
  <param-value>/home/jiajia/soft/opengrok-0.12.1.6/etc/configuration.xml</param-value>
  <description>Full path to the configuration file where OpenGrok can read it's configuration</description>
</context-param>
```

### 配置tomcat局域网访问

``` html
//name=“localhost” 改成ip
vi apache-tomcat-8.5.16/conf/server.xml

<Host name="192.168.64.245"  appBase="webapps"
      unpackWARs="true" autoDeploy="true">
```
