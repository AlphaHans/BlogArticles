---
title: 【从零开始的Weex世界】一：Hello,Weex!
date: 2017-04-19 22:09
tag: Weex
category: Weex
---
## 安装weex开发环境

* 参照官网文档安装即可


我的环境是：

```

node:v6.10.2

weex:v1.0.5

weex-devtool:v0.2.80

weexpack:v0.4.0

```
<!-- more -->

## 创建weex工程

* 选定好一个目录，打开终端

* 初始化weex工程：

```

weex init

```

* 安装node_module依赖：

```

npm install

```



## 编译Weex工程

* 运行

```

npm run build

```

这个命令是什么意思呢？

可以参考：[阮一峰的网络日志 - npm](http://www.ruanyifeng.com/blog/2016/10/npm_scripts.html)

实际上这个命令就是npm在package.json文件里面，定义的脚本

```

"scripts": {

    "build": "webpack",

    "dev": "webpack --watch",

    "serve": "node build/init.js && serve -p 8080",

    "debug": "weex-devtool"

  }

```

其实

```

npm run build

```

等价于

```

webpack

```



此外，可以看到这里四个脚本：

* build：通过webpack打包工具进行打包

* dev：使用webpack --watch进行网页动态刷新呢

* serve：开启本地服务

* debug：开启weex调试



注：

* webpack是前端新一代的打包工具，具体可以参考：[imooc幕课网 - webpack深入与实战](http://www.imooc.com/learn/802) 进行学习

* 强烈推荐先看一下上面的视频，能够更好理解webpack.config.json



## 启动Weex工程

* 经过上面编译

```

npm run build

```

之后可以看到在/dist目录下输出了两个文件：app.web.js和app.weex.js（具体的内容的意思，通过上面慕课网webpack视频之后即可大致了解）

* 开启动态编译代码

```

npm run dev //等于 webpack --watch

```

会显示：

![npm run dev](http://ww1.sinaimg.cn/large/7cbc163dgy1feul4uqwpvj20jd0da0up.jpg)

这样一来，我们修改代码；就会实时编译成app.web.js和app.weex.js，而不需要我们每次都通过`npm run build`命令进行编译

* 启动Weex工程

开启另一个终端，然后运行

```

npm run serve

```

![npm run serve](http://ww1.sinaimg.cn/large/7cbc163dgy1feul7zsu52j20jd0datad.jpg)

* 在浏览器访问`localhost:8080`即可

![](http://ww1.sinaimg.cn/large/7cbc163dgy1feulmmsh1fj21hc0qp0uw.jpg)

说明启动成功了





## 小插曲

一开始我直接

```

npm run serve 

```

是这样的

![](http://ww1.sinaimg.cn/large/7cbc163dgy1feulnvif1ij21hc0qqac8.jpg)

一片空白，我们打开控制台可以看到：

![](http://ww1.sinaimg.cn/large/7cbc163dgy1feulowdm83j21hc0b0jw7.jpg)



Google发现解决方法：[Segmentfault - weex-vue-render 如何查看更新日志](https://segmentfault.com/q/1010000008829493)

问题原因：

```

http://localhost:8080/node_modules/weex-vue-render/index.js这个在最新版本已经去掉了。 要换成http://localhost:8080/node_modules/weex-vue-render/dist/index.js. 注意多了个dist

```

**我的weex-vue-render版本是0.11.3，产生这个问题的原因是weex工具更新没有跟上weex-vue-render的更新**

修改措施：

找到工程weex.html文件中

```

  <script src="./node_modules/weex-vue-render/index.js"></script>

```

修改为

```

  <script src="./node_modules/weex-vue-render/dist/index.js"></script>

```