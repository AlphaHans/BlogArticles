---
title: 【WebView】（三）第三方链接出现BUG调试.md
tag: WebView
category: Android进阶
date: 2017-1-2 22:15
---
## 问题描述

项目中嵌套了WebView进行文章展示，难免会有外链，但是第三方的外链可能在本地的WebView会有水图不服的现象，按我们Android开发者可以通过一些方法进行WebView的调试


<!-- more -->

## 必要工具

* Android 4.4以上手机 （4.4以上内核为chromium）

* PC端Chorme浏览器



注意不是所有Android4.4以上都可以，手里有一台三星并不行，最后通过Nexus模拟器来调试



## 调试步骤

* PC Chrome打开网页：chrome://inpect

* 选择Device 如图所示可以看到已经连接的设备

![](http://p1.bpimg.com/567571/2b41ddc138e8833d.png)

* 在Android手机通过WebView打开怀疑有问题的第三方网站页面，点击inspect

![](http://i1.piimg.com/567571/1f06085ac0547ece.png)

* 在Console面板中查看错误信息

![](http://i1.piimg.com/567571/abe80d7ae2a80c3c.png)
红字是错误信息，左侧是错误所在的JavaScript文件

* 找到错误

ctrl+f呼出搜索面板，输入关键词找到两处错误：

![](http://i1.piimg.com/567571/1ab3bc3f78036a55.png)
![](http://i1.piimg.com/567571/358268f815471acc.png)

* 定性错误

发现错误是localStorage出现，如果没有前端编程经验的话可能要找前端伙伴帮忙解决了。

localStorage是什么可以参考下面的文章：

[CSDN - Android WebView缓存分析](http://blog.csdn.net/a345017062/article/details/8703221)
这样就可以找到思路去解决问题了，而不用一头脑在找问题



## 知识补充

### WebView相关的缓存

* AppCache：html、css、js存储

* DOM Storage：存储kv数据非常不错

    * Session Storage：会话级别，关了页面就清除

    * Local Storage：永久保存



[CSDN - Android WebView缓存分析](http://blog.csdn.net/a345017062/article/details/8703221)

[简书 - Android WebView性能优化](http://www.jianshu.com/p/95d4d73be3d1)



### 浏览器内核

[JOB BOLE - 主流浏览器内核](http://web.jobbole.com/84826/)

* WebKit

    * fork自KHTML

    * Apple Safari是鼻祖

    * 2008 年Google发布 Chrome 浏览器，采用的 chromium 内核便fork 自 Webkit。



* Chromium/Bink

  * 2008 年Google发布 Chrome 浏览器，采用的 chromium 内核便fork 自 Webkit。

  * 精简WebKit，开发Javascript V8引擎，极大提高JavaScript执行速度

  * 国产几乎所有浏览器都是在Chromuim再次包装而来

  * 2013年Google宣布与Apple分道扬镳，并且Apple发布的WebKit2与Chromium的沙箱机制存在冲突，而且Chromium与WebKit2的兼容有很大困难

  * 此后Google开始研发Blink（WebKit的另外一个分支）

**-Hans 2017.1.2**
