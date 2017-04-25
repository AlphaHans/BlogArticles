---
title: 【设计模式】Java静态代理与动态代理
date: 2017-1-5 16:43
tag: 设计模式
category: Android进阶
---
## 参考文章


* [Android沉思录 - Android公共技术点之四-Java动态代理](http://yeungeek.com/2016/05/19/Android%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%E5%9B%9B-Java%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86/)

* [codeKK - 公共技术点之 Java 动态代理](http://p.codekk.com/blogs/detail/54cfab086c4761e5001b253d)

<!-- more -->

## 为什么要用代理

代理分为：静态代理和动态代理

用代理的优点：

* 隐藏委托类的实现解耦

* 不改变委托类代码情况下做一些额外处理，比如添加初始判断及其他公共操作、数据库的权限判断等



## 动态代理优点

* 免除了静态代理中：修改接口则所有实现类以及其代理类都要修改的缺点。 在动态代理中，只需要在实现类中添加具体实现，而不需要修改代理类

* 动态代理能够对相关特殊方法进行统一处理，如：数据库编程中，对于增删查改函数都需要进行权限验证。如果使用动态代理，能够只在动态代理中的invoke函数进行集中式处理，避免逻辑重复和耦合度。



## 实践

### 第一步 说明代理的好处

接口类：

```

public interface TimeInterface {//获取时间接口
    String getTime();
}
```

实现类：

```

public class TimeImpl implements TimeInterface {
    @Override
    public String getTime() {
        return String.valueOf(System.currentTimeMillis());
    }
}
```

代理类（静态代理）：

```

public class StaticTimeProxy implements TimeInterface {
    private TimeInterface mDelegate;

    public StaticTimeProxy() {
        mDelegate = new TimeImpl();
    }

    @Override
    public String getTime() {
        return mDelegate.getTime();
    }
}
```

测试代码：

```

public static void main(String args[]) {
    TimeInterface timeInterface = new StaticTimeProxy();
    String time = timeInterface.getTime();
    System.out.println(time);
}
```

输出结果：

```

1483603974906

```

那么如果现在我这样一个场景：

* 1.当是时间等于2017年时，才返回正确的毫秒值

* 2.如果不是2017年，则返回空时间

那么我可能需要这样修改：

```

public class StaticTimeProxy implements TimeInterface {
    ...
    

    @Override
    public String getTime() {
        String time = mDelegate.getTime();
        Date date = new Date(Long.valueOf(time));
        Calendar calendar = Calendar.getInstance(Locale.CHINA);
        calendar.setTime(date);
        //只有等于2017年才允许获取时间
        if (calendar.get(Calendar.YEAR) == 2017) {
            return time;
        }
        //否则返回空
        return null;
    }
}
```

好处:

* 1.我只需要修改改代理类的代码

* 2.不需要出修改相对底层的`TimeImpl`类

* 3.保证了其他使用`TimeImpl`地方不会受到代理类的修改而导致错误

### 第二步 说明动态代理的好处 

假设我现在有一个这样的需求，`TimeInterface`我需要一个增加一个`getYear`直接获取年份的接口

```

public interface TimeInterface {
    String getTime();
    
    int getYear();
}
```

那么此时，所以实现了TimeInterface的实现类和代理类都要重写这个方法，如上文的`TimeImpl`和`StaticTimeProxy`

然而这在大工程里面来说，可以是几乎不能接受的，需要修改很多个类。但是如果是动态代理，动态代理本身并不需要修改，只需要修改具体的实现类即可。

为`TimeImpl`实现新的`getYear()`方法：

```

····

@Override
public int getYear() {
    Date date = new Date(System.currentTimeMillis());
    Calendar calendar = Calendar.getInstance(Locale.CHINA);
    calendar.setTime(date);
    return calendar.get(Calendar.YEAR);
}
····
```

测试代码：

```

public static void main(String args[]) {
    TimeInterface timeInterface = (TimeInterface) new ReflectTimeProxy().getPoxy();
    String time = timeInterface.getTime();
    System.out.println(time);
    System.out.println(timeInterface.getYear());
}
```

输出结果：

```

1483605552828

2017

```

## 源码
[Github - JavaProxyDemo](https://github.com/AlphaHans/JavaProxyDemo)


**-Hans**

