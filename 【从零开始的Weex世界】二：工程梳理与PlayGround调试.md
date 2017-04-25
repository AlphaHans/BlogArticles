---
title: 【从零开始的Weex世界】二：工程梳理与PlayGround调试
date: 2017-04-20 22:09
tag: Weex
category: Weex
---
## 目标
* 了解工程目录
* 理解Weex工作原理

<!-- more -->

## 工程目录说明

![](http://ww1.sinaimg.cn/large/7cbc163dgy1feulualy9aj20a40ebmxl.jpg)

* assets 存放的是一些工程开发中需要用到的文件代码，如qrcode.js就是用于生成二维码的js

* build 我也不清楚，一般不会用到也不会修改

* dist app.web.js和app.weex.js输出目录，即输出的JSBundle代码

* node_module 依赖模块

* src 我们的代码存放文件夹

* app.js 入口entry，当我们启动工程的时候，这个文件是入口

* config.js 不需要修改

* index.html PC端网页的html文件，我们可以看到里面有qrcode的二维码代码。对应声明app.web.js的JSBundle

* package.json npm需要的配置文件

* webpack.config.json webpack的配置文件

* weex.html 移动端的入口文件，对应生成app.weex.js的JSBundle

## 梳理工程

* 我们打算将Hello world改成Hello，Weex世界！

![](http://ww1.sinaimg.cn/large/7cbc163dgy1feum1jwaunj20sw0kqjsh.jpg)

* 找到src文件下的foo.vue（自带的）
```

<text class="title">Hello {{target}}</text>
```


两个大括号是Vue自带的语法糖，表明这里是使用target变量中的字符串。

这种语法很奇怪，最终会经过webpack的vue-loader生成浏览器可以识别的html、css、js文件

那么修改其中的变量即可：
```

<script>

  export default {

    data: {

      logoUrl: 'https://alibaba.github.io/weex/img/weex_logo_blue@3x.png',

      target: 'Weex世界！！'

    },

    methods: {

      update: function (e) {

        this.target = 'Weex'

        console.log('target:', this.target)

      }

    }

  }

</script>

```

* 开启了`npm run dev`，我们直接刷新网页即可看到已经成功修改了



## 这是怎么做到的呢？

* 上文说了，app.js是我们打包的入口

* 查看app.js文件

```

import foo from './src/foo.vue' //导入foo.vue组件

foo.el = '#root' //挂载到root div下

export default new Vue(foo); //导出

```

* 查看weex.html

```

...

<body>

  <div id="root"></div> //foo.vue被包裹在此

  <script src="./dist/app.web.js"></script>

</body>

...

```

* 从上面步骤我们可以知道了：修改foo.vue如何导致weex.html被修改



## 使用Playground

* 从官网下载playground安装到手机

* 通过app内二维码扫描网页的二维码，那么我们就可以看到app.weex.js渲染在我们的手机上了（注意要在一个wifi下）

![](http://ww1.sinaimg.cn/large/7cbc163dgy1feumc9r8b4j218g1z4n0i.jpg)



## Debug调试

* 使用weex的debug模式

* 输入命令

```

npm run debug

```

* 然后参考官方文档学习如何使用playground进行调试

* 注意这个很重要，可以少走很多弯路