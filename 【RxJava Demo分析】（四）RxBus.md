---
title: 【RxJava Demo分析】（四）RxBus
date: 2016-03-13 20:52
tag: RxJava
category: RxJava
---
<h1>RxBus</h1>
也许大家都听过或者项目中都是使用Otto或者EventBus这两个事件总线库，这两者都是基于订阅者实现的。大家也知道RxJava也是基于订阅者实现的。 

<!-- more -->
那么是否RxJava也可以实现事件总线呢？ 

是的！ RxJava的出现，给我们带了了一个无比轻量的事件总线“库”，而且实现的方式非常简单，完全能够满足我们的需求！

需要注意，RxBus不是一个库，而是依托RxJava给我们带来的一种全新思想！

正是这个特性！ 让我学习RxJava热情又加深了~~ 

照旧，我们来看看RxBus这个类的代码！

```
public class RxBus {
    //private final PublishSubject<Object> _bus = PublishSubject.create();

    // If multiple threads are going to emit events to this
    // then it must be made thread-safe like this instead
    private final Subject<Object, Object> _bus = new SerializedSubject<>(PublishSubject.create());

    public void send(Object o) {
        _bus.onNext(o);
    }

    public Observable<Object> toObserverable() {
        return _bus;
    }

    public boolean hasObservers() {
        return _bus.hasObservers();
    }
    
}
```
我第一次理解这里的代码的时候，我简直不敢相信！ 这么简单的几行代码就可以搞定了！

我们来看看这里面的PublishSubject、SerializedSubject和Subject

<h2>1.PublishSubject</h2>
概括：只会把订阅之后的数据发射给观察者

可以看这张图：
![这里写图片描述](http://img.blog.csdn.net/20160313203316365)

这图表明的意思就是，订阅者订阅之后，只会收到订阅之后的消息~

所以这个跟我们事件总线的要求很符合~。

<h2>2.SerializedSubject</h2>
概括：使Subject线程同步,返回一个线程安全的对象

<h2>3.Subject</h2>
概括：兼有Observer和Observable功能。因为是Observer，所以可以订阅数据；又因为是Observable也可以转发数据。

OK！ 我们了解完这三个知识点之后对RxBus也应该有一个初步的了解了吧？

实际上他就是通过一个线程安全的Subject（可以理解为Observe） 然后不断进行onNext进行分发。 他的每个订阅者都会收到消息，但是你可以根据事件对象来判断是否是自己所需要的对象。

比如：
```
o.subscribe(new Action1<Object>() {
	@Override
	public void call(Object event) {
		if (event instanceof RxBusDemoFragment.TapEvent) {
			_showTapText();
		}
	}
});
```
我们可以通过 instanceof 来判断是否是我们的目标事件~

<h1>修改</h1>
实际上这个RxBus还不是特别地符合我们的需求~

实际上呢，我们可以修改一下
```
 public Observable<Object> toObserverable() {
        return _bus;
 }
```
方法 改成：
```
public <T> Observable<T> toObserverable(final Class<T> eventType) {
	return _bus.filter(new Func1<Object, Boolean>() {
		@Override
		public Boolean call(Object o) {
			return eventType.isInstance(o);
		}}).cast(eventType);
}
```
直接返回我们目标的Observable对象~ 这样我们就不需要进行判断了，因为filter操作符帮我们过滤了。 然后通过cast转换成我们需要的Observable。

思路来自：[用RxJava实现事件总线(Event Bus)](http://www.jianshu.com/p/ca090f6e2fe2)

**-Hans 2016.3.13 20:52**