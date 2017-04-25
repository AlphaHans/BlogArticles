---
title: 【WebView】（二）JavaScript调用Java小结
tag: WebView
category: Android进阶
date: 2016-12-27 17:43
---

## 四种基本通讯方法

JS调用Java原生方法有四种方式：

* JavaInterface

* WebViewClinent#onShouldOverrideUrlLoading

* WebViewChromeClient#onConsoleMessage

* WebViewChormeClient#onJsConfirm


<!-- more -->

第一种是Android WebView提供的一个通过注入JS Object来实现的，相当于网页加载了一个JS文件

优点：简单粗暴

缺点：4.1及以下存在安全漏洞 如：通过JS函数反射获取类，来发送信息等，导致恶意扣费。



其他三种是通过“消息传递”的方式：1.URL携带信息 2.控制面板输出信息 3.弹窗携带信息

原理就是：客户端从URL\面板\弹窗拿到消息，这个消息是前端和移动端规定好的一个消息协议（自由规定）



## 以WebViewClinent#onShouldOverrideUrlLoading为例

以WebViewClinent#onShouldOverrideUrlLoading为例，其他情况是类似的

前端代码：

```

<body>

      <a href="test://methodName?params1&params2&params3">触发JS</a>

 </body>

```

Android：

```

webView.setWebViewClient(new WebViewClient() {
    @Override
    public boolean shouldOverrideUrlLoading(WebView view, String url) {
        Log.i(TAG, "shouldOverrideUrlLoading: " + url);
        return super.shouldOverrideUrlLoading(view,url);
    }
});
```

会输出下面的消息：

```

test://methodName?params1&params2&params3

```

就可以根据methodName和后面的参数 来具体调用方法

### 特殊情况

不过注意 协议名（test）不要使用下划线等特殊字符，可能会出现特殊情况；

如协议名为：test_sb，输出的会是这样的（我点击触发的页面是：`http://10.1.0.63:8080/test/index.html'）：

```

http://10.1.0.63:8080/test/test_sb://methodName?params1&params2&params3

```

**-Hans 2016.12.27**
