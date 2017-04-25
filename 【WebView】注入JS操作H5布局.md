---
title: 【WebView】注入JS操作H5布局
tag: WebView 
category: Android进阶
date: 2016-05-23 13:41
---
# 前言
**本文用来记录一次WebView开发，没有详细讲解。 
需要有一定的WebView开发经验**

<!-- more -->
如何通过WebView加载JS代码来操控H5的布局呢？

需要明白这个，首先要知道前端工程师是如何调试代码的。

# 前端Chrome浏览器注入JS过程
打开Chrome浏览器，然后输入m.taobao.com，然后F12
![1](http://img.blog.csdn.net/20160523133723140)
就可以看到这样的一个页面。

假设要隐藏最上面的搜索框
![2](http://img.blog.csdn.net/20160523133735031)
首先通过Chrome的Elements查看元素，找到搜索框所在的div
![3](http://img.blog.csdn.net/20160523133746500)
可以看到这个div的class名字是 top-fixed
![4](http://img.blog.csdn.net/20160523133800235)
然后借助Chrome的console面板，进行调试
![5](http://img.blog.csdn.net/20160523133807923)
打开Console面板，输入（这行代码暂时不解释）：
```
document.getElementsByClassName('top-fixed')[0].style.display='none';
```
![6](http://img.blog.csdn.net/20160523133817861)
回车
![7](http://img.blog.csdn.net/20160523133826314)
现在发现原来的搜索框已经不见了。

实际上我们通过这种方式就可以使用JS来操作H5布局了，那么就可以同理到Android WebView中去了。

# Android WebView 注入JS过程
通过上面的过程我们已经确定JS函数，可以直接操作H5的布局。那么现在我们只需要找到WebView如何注入一个JS并且调用它了。

直接上代码：
```
mWebView.loadUrl("javascript:(function() " +
        "{ " +
        "document.getElementsByClassName('top-fixed')[0].style.display='none'; " +
        "})()");
```

现在解释一下这行代码：
```
document.getElementsByClassName('top-fixed')[0].style.display='none';
```
* document:这个是在JS中用来获取整个网页Page的doc树的，类似于Andoird 中View布局
* getElementsByClassName:类似于findViewById\findViewByTag
* 通过这种方式可能找到多个class名字都为top-fixed的，所以返回的是数组。这里我们取第一个元素，即[0]
* style.display='none':修改这个div标签的样式

然后解释一下外面的这个“壳”：
**javascript:(funcation())() 
语法解析：通过JavaScript来定义一个函数 然后再通过使用()表示立刻执行**

# 小结
实际上，JS能做的很多，只要Chrome浏览器Console面板可以做到，在Android WebView里面也可找到相应的办法。比如对某个div的监听事件等等。

**-Hans 2016.05.23 13:42**