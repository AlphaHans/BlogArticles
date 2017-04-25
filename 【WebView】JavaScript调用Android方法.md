---
title: 【WebView】JavaScript调用Android方法
tag: WebView 
category: Android进阶
date: 2016-03-20 18:05
---
<h1>Error</h1>
今天开发的时候第一次接触HTML 5通过WebView与Android Naive交互！

<!-- more -->
然后当我按照网上教程搞了一下的时候发现一直会有这个BUG
```
I/chromium: [INFO:CONSOLE(9)] "Uncaught TypeError: android.success is not a
function", source: file:///android_asset/index.html (9)
```

当时我的代码是这样的：

```
 mWebView.loadUrl("file:///android_asset/index.html");//这里方便 直接加载assets里面的html文件~

 mWebView.getSettings().setJavaScriptEnabled(true);
        mWebView.addJavascriptInterface(new Object() {

            public void success() {
                Log.i("addJavascriptInterface", "success ");
            }

            public void error() {
                Log.i("addJavascriptInterface", "error");
            }

        }, "android");
```

网页端的代码是这样的：
```
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"
        "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="zh-CN" dir="ltr">
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8"/>

    <script type="text/javascript">
function showAndroidToast(toast) {       
    javascript:android.success(toast);
}


    </script>

</head>
<body>
<input type="button" value="Say hello"
       onClick="showAndroidToast('Hello Android!')"/>
</body>
</html>
```

<h1>解决方法</h1>
查了好多文章 都没有提及到这个BUG。

所以查阅了一下官方文档：

>public void addJavascriptInterface (Object object, String name)

>Added in API level 1
Injects the supplied Java object into this WebView. The object is injected into the JavaScript context of the main frame, using the supplied name. This allows the Java object's methods to be accessed from JavaScript. For applications targeted to API level JELLY_BEAN_MR1 and above, only public methods that are annotated with JavascriptInterface can be accessed from JavaScript. For applications targeted to API level JELLY_BEAN or below, all public methods (including the inherited ones) can be accessed, see the important security note below for implications.

>Note that injected objects will not appear in JavaScript until the page is next (re)loaded. For example:

>class JsObject {
    @JavascriptInterface
    public String toString() { return "injectedObject"; }
 }
 webView.addJavascriptInterface(new JsObject(), "injectedObject");
 webView.loadData("", "text/html", null);
 webView.loadUrl("javascript:alert(injectedObject.toString())");


 这点是这一句话：


 >For applications targeted to API level JELLY_BEAN_MR1 and above, only public methods that are annotated with JavascriptInterface can be accessed from JavaScript. For applications targeted to API level JELLY_BEAN or below, all public methods (including the inherited ones) can be accessed


意思是：对于Android版本在JELLY_BEAN_MR1和以上的版本需要增加注解@JavascriptInterface

加上这一个就搞定啦~~


**-Hans 2016.3.20 18:05**



 