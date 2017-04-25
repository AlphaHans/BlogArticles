---
title: 【Weex】Weex环境搭建
date: 2017-01-18 21:54
tag: Weex
category: Weex
---

## 安装Nodejs与npm

<!-- more -->

```

node -v

npm -v

```

可以用于查看当面nodejs和npm的版本



淘宝镜像cnpm:

```

npm install -g cnpm

```



## 安装Weex ToolKit

```

npm install -g weex-toolkit

```

![](http://p1.bpimg.com/567571/72b9d8d509c03318.png)

如果是macOS可能需要添加权限：

```

sudo npm install -g weex-toolkit

```

安装成功后：

```

weex --version

```

可以看到当前版本

![](http://i1.piimg.com/567571/f17285c21e83bd5e.png)



## Hello

创建文件`hello.we`:

```

<template>

  <div>

    <text>Hello</text>

  </div>

</template>

<style></style>

<script></script>

```

然后使用Weex命令编译

```

weex hello.we

```

即可看到：

![](http://i1.piimg.com/567571/ab6db1a614221c34.png)



## Weex编译后的html文件

在相同目录下会生成一个weex_temp的文件夹，里面存放了真正的html文件