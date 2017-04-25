---
title: 【从零开始的Weex世界】三：添加一个AppBar
date: 2017-04-21 22:09
tag: Weex
category: Weex
---

## 目标

* 通过.vue组件化写一个AppBar

* 添加到我们第二节的页面中去

<!-- more -->

## AppBar.vue

代码如下：

```

<template>

    <div class="bar">

        <text class="bar_lr_text" v-if="vLeft" @click="onLeftClick()">返回</text>

        <text class="bar_title">{{title}}</text>

        <text class="bar_lr_text" @click="onRightClick()" v-if="vRight">分享</text>

    </div>

</template>

<style>

.bar {

    height: 100px;

    flex-direction: row;

    background-color: #41B883;

}



.bar_title {

    flex: 1;

    color: #ffffff;

    height: 100px;

    font-size: 45px;

    padding-top: 25px;

    text-align: center;

    text-overflow: ellipsis;

    lines: 1;

    line-height: 100px;

    padding-left: 50px;

    padding-right: 50px;

}



.bar_lr_text {

    height: 100px;

    width: 100px;

    color: #fff;

    text-align: center;

    font-size: 30px;

    line-height: 100px;

    background-color: lightseagreen;

}

</style>

<script>

export default {

    props: ['title', 'vLeft', 'vRight'],

    data() {

        return {}

    },

    methods: {

        onLeftClick: function() {},

        onRightClick: function() {



        }

    }

}

</script>

```

* template style script 是vue的写法

* v-if状态绑定，用于决定是否显示左右两侧的text

根据表达式的值的真假条件渲染元素。在切换时元素及它的数据绑定 / 组件被销毁并重建。([Vuejs - v-if](https://cn.vuejs.org/v2/api/#v-if))

注意他和v-show的区别

根据表达式之真假值，切换元素的 display CSS 属性。([Vuejs - v-show](https://cn.vuejs.org/v2/api/#v-show))

* @click写法

实际上是v-on的缩写([Vuejs - v-on](https://cn.vuejs.org/v2/api/#v-on))

等价于

```

v-on:click=''

```

绑定事件监听器。事件类型由参数指定。表达式可以是一个方法的名字或一个内联语句，如果没有修饰符也可以省略。



## 添加到页面中去

* 修改foo.vue

```

<template>

  <div class="wrapper" @click="update">

    <app-bar title='标题' vRight='true' vLeft='true'></app-bar>

    <image :src="logoUrl" class="logo"></image>

    <text class="title">Hello {{target}}</text>

  </div>

</template>



<style>

  .wrapper { align-items: center; margin-top: 120px; width: auto }

  .title { font-size: 48px; }

  .logo { width: 360px; height: 82px; }

</style>



<script>

  import AppBar from './AppBar.vue'



  export default {

    components:{

      AppBar

    },

    data: {

      logoUrl: 'https://alibaba.github.io/weex/img/weex_logo_blue@3x.png',

      target: 'Weex世界！'

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

* 注意

 * 通过import语句 导入 AppBar.vue

 * 在export中导出 组件AppBar

 * 在template中使用`<app-bar>`添加组件（这个也是vue的约定写法，一定要这样写 大写字母变小写，前面加-）

 * 可以发现AppBar的状态：title、是否显示左右等是从外部传入的，通过props

[Vuejs - 使用props传递数据](https://cn.vuejs.org/v2/guide/components.html#使用-Prop-传递数据)

* 效果图：

![](http://ww1.sinaimg.cn/large/7cbc163dgy1feunc9bcs0j20900h3jrs.jpg)