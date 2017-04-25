---
title: 【Android】Android中的ClassLoader小结.md
date: 2017-1-10 17:22
tag: ClassLoader
category: 热修复
---
## 参考资料

* [热修复入门：Android中的ClassLoader](http://jaeger.itscoder.com/android/2016/08/27/android-classloader.html)
<!-- more -->
## 概述

ClassLoader分为一下两大类

* 系统相关

    * BootClassLoader

* 应用相关

    * BaseDexClassLoader

        * PathDexClassLoader

        * DexClassLoader

    * SecureClassLoader

        * URLClassLoader



`SecureClassLoader` 的子类是 `URLClassLoader` ，其只能用来加载 `jar 文件`，这在 Android 的 Dalvik/ART 上没法使用的。







## BootClassLoader

* PathClassLoader的父类加载器



流程：

* 其在系统启动时创建，在 App 启动时会将该对象传进来

* 具体的调用在 `com.android.internal.os.ZygoteInit` 的 `main()` 方法中调用了` preload() `

* 然后调用 `preloadClasses()` 方法

* 在该方法内部调用了 Class 的` forName()` 方法



源代码如下：

```

public static Class<?> forName(String className, boolean shouldInitialize,
        ClassLoader classLoader) throws ClassNotFoundException {
    if (classLoader == null) {
        classLoader = BootClassLoader.getInstance();
    }
    // Catch an Exception thrown by the underlying native code. It wraps
    // up everything inside a ClassNotFoundException, even if e.g. an
    // Error occurred during initialization. This as a workaround for
    // an ExceptionInInitializerError that's also wrapped. It is actually
    // expected to be thrown. Maybe the same goes for other errors.
    // Not wrapping up all the errors will break android though.
    Class<?> result;
    try {
        result = classForName(className, shouldInitialize, classLoader);
    } catch (ClassNotFoundException e) {
        Throwable cause = e.getCause();
        if (cause instanceof LinkageError) {
            throw (LinkageError) cause;
        }
        throw e;
    }
    return result;
}
```

### PathClassLoader

* BaseDexClassLoader的子类

* PathClassLoader 只能加载**已经安装**应用的 dex 或 apk 文件

* PathClassLoader 在应用启动时创建，从 data/app/… 安装目录下加载 apk 文件

#### PathClassLoader的实例化

![](http://p1.bqimg.com/567571/051901505cb64a3a.jpg)

* 在 ZygoteInit 中的调用是用来启动相关的系统服务

* 在 ApplicationLoaders 中用来加载系统安装过的 apk，用来加载 apk 内的 class ，其调用是在 LoadApk 类中的 getClassLoader() 方法中调用的，得到的就是 PathClassLoader：

```

mClassLoader = ApplicationLoaders.getDefault().getClassLoader(zip, lib,mBaseClassLoader);
```
### DexClassLoader

* BaseDexClassLoader的子类

* 没有PathClassLoader的限制，允许从任何地方加载dex或者apk文件

* **热修复的关键**



**-Hans**
