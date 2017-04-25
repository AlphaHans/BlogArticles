---
title: 【RxJava】Observable基本方法
date: 2016-02-02 22:06 
---
<h1>1.前言</h1>
随着RxJava越来越火，相信在2016年必定会大方异彩。 虽然前前后后看了不少RxJava的文章，但都没有积累下来，又没有在实际项目中使用过。 

因此特意写下这篇文章记录学习过程。

<!-- more -->
<h1>2.简介RxJava</h1>
一般我们进行耗时任务，如网络、数据库查询、复杂计算等等，我们都回开启一个线程，然后通过接口回调，获取我们的结果。 但随着我们业务逻辑的越来越复杂，我们就会陷入一个回调地狱，回调里面还有回调，在日后我们维护代码来说简直是噩梦。

RxJava的出现正式为了解决这个问题而生的，它支持链式调用！

关键字：**异步**、**链式调用**、**观察者模式**

这篇文章主要来记录Observable基本用法

<h1>2.create</h1>

```
final List<String> list = Arrays.asList(new String[] {"one","two","three"});

Observable observable = Observable.create(new OnSubscribe<List<String>>() {
			
			@Override
			public void call(Subscriber<? super 
								List<String>> subscriber) {
				subscriber.onNext(list);
				subscriber.onCompleted();
			}
});

```
可以发现，我们发射的是以整个`List<String>` 我们可以发射一个一个对象吗？

当然可以：

```
		Observable observable = Observable.create(new 
			OnSubscribe<String>() {
			
			@Override
			public void call(Subscriber<? super String> 
			subscriber) {
				for (String str:list) {
					subscriber.onNext(str);
				}
				subscriber.onCompleted();
			}
		});
```
这样看起来好像还不是很优雅！有没有办法刚优雅呢？ 那我们来看看from这个方法

<h1>3.from</h1>

```
Observable.from(list).subscribe(new Observer<String>() {

			@Override
			public void onCompleted() {
				System.out.println("onCompleted");
			}

			@Override
			public void onError(Throwable arg0) {
				
			}

			@Override
			public void onNext(String string) {
				System.out.println(string);
			}
		});
```
结果是：

```
one
two
three
onCompleted

```
符合我们的预期！

<h1>4.just</h1>
如果我只想发射list中的第二和第三位可以吗？

当然可以，我们可以借助just方法：

```
		Observable.just(list.get(1), list.get(2)).subscribe(new Observer<String>() {

			@Override
			public void onCompleted() {
				System.out.println("onCompleted");
			}

			@Override
			public void onError(Throwable arg0) {
				
			}

			@Override
			public void onNext(String string) {
				System.out.println(string);
			}
		});
```
结果：

```
two
three
onCompleted

```
符合我们的预期！

**备注：just方法可以接受1-10个参数**

<h1>5.repeat</h1>
如果我们想将整个list重复发射两次或者三次呢？

```
Observable.from(list).repeat(2).subscribe(new 
		Observer<String>() {

			@Override
			public void onCompleted() {
				System.out.println("onCompleted");
			}

			@Override
			public void onError(Throwable arg0) {
				
			}

			@Override
			public void onNext(String string) {
				System.out.println(string);
			}
		});
```
结果：

```
one
two
three
one
two
three
onCompleted

```
符合我们预期

**备注：repeat可以不传参，效果是：无限循环**

<h1>6.range</h1>
从X按顺序输出Y位数字？

```
Observable.range(88,10).subscribe(new Observer<Integer>() {

			@Override
			public void onCompleted() {
				System.out.println("onCompleted");
			}

			@Override
			public void onError(Throwable arg0) {
				
			}

			@Override
			public void onNext(Integer arg0) {
				System.out.println(arg0+"");
			}
		});
``` 

```
88
89
90
91
92
93
94
95
96
97
onCompleted

```

<h1>7.interval（测试失败）</h1>
间隔时间发射：

```
Observable.interval(3, TimeUnit.SECONDS).subscribe(new Subscriber<Long>() {

		@Override
		public void onCompleted() {
			ystem.out.println("onCompleted");
		}

		@Override
		public void onError(Throwable arg0) {
			
		}

		@Override
		public void onNext(Long arg0) {
			System.out.println(arg0+"");
		}
});

```
很奇怪，这个方法并没有起作用。

<h1>8.timer（测试失败）</h1>
延迟发射：

```
Observable.timer(1, TimeUnit.SECONDS).subscribe(new Observer<Long>() {

		@Override
		public void onCompleted() {
			// TODO Auto-generated method stub
				
		}

		@Override
		public void onError(Throwable arg0) {
			// TODO Auto-generated method stub
				
		}

		@Override
		public void onNext(Long arg0) {
			System.out.println(arg0+"");
		}
});
```
这个方法和interval一样，也是不能测试。

它还有一个三个参数的方法timer(3,3,TimeUnit.SECONDS) 延迟3秒之后，每隔3秒发射一次