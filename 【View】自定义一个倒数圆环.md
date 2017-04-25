---
title: 【View】自定义一个倒数圆环
date: 2016-11-5 17:22
tag: View
category: Android进阶
---

<!-- more -->
## 源码

[Github-CountingDonutView](https://github.com/AlphaHans/CustomView/blob/master/app/src/main/java/xyz/hans/customview/view/CountingDonutView.java)



## 知识点

* `RectF`的left、top、right、bottom是相对于View本身的，而不是屏幕的坐标系

* 通过`canvas.drawArc`方法可以绘制一段圆弧。

    * 第一个参数：RectF。

    * 第二个参数：起始角度。 如果为0则为三点钟；比如-45是一点半方向开始。

    * 第三个参数：结束角度。 正顺时针，负为逆时针；比如第二和第三参数分别为0和45，那么这段圆弧是从三点到六点的一段1/4弧。

    * 第四个参数：boolean。 是否绘制原点。

    * 第五个参数：Paint画笔



## 扩展

在此之上，基于[CircleImageView](https://github.com/hdodenhof/CircleImageView) 实现了圆形头像外部有一条倒数的ProgressBar的CircleImageView效果![](http://i1.piimg.com/567571/ee50ff8d1b6c1897.png)



源码地址：[Github-CircleProgressImageView](https://github.com/AlphaHans/CustomView/blob/master/app/src/main/java/xyz/hans/customview/view/CircleProgressImageView.java)