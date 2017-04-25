---
title: 【RxJava Demo分析】（三）buffer操作符、RxBinding库
date: 2016-03-05 17:15
tag: RxJava
category: RxJava
---
<h1>BufferDemoFragment</h1>
我们看一下这个原作者的注释：

<!-- more -->
```
This is a demonstration of the `buffer` Observable.
 
The buffer observable allows taps to be collected only within a time span. So taps outside the
2s limit imposed by buffer will get accumulated in the next log statement.
```

简而言之，这个buffer Observable可以让你在一个时间段内，计算你点击的次数（或者其他）。

照旧我们看一下所用到的RxJava代码：
```
    private Subscription _subscription;

	@Override
    public void onStart() {
        super.onStart();
        _subscription = _getBufferedSubscription();
    }

    @Override
    public void onPause() {
        super.onPause();
        _subscription.unsubscribe();
    }
	
	// Main Rx entities
    private Subscription _getBufferedSubscription() {
        return RxView.clickEvents(_tapBtn) //RxView
              .map(new Func1<ViewClickEvent, Integer>() {
                  @Override
                  public Integer call(ViewClickEvent onClickEvent) {
                      Timber.d("--------- GOT A TAP");
                      _log("GOT A TAP");
                      return 1;
                  }
              })
              .buffer(2, TimeUnit.SECONDS)
              .observeOn(AndroidSchedulers.mainThread())
              .subscribe(new Observer<List<Integer>>() {
				//一些处理
              });
    }
```
从这个例子中我们可以看到两个知识点：buffer操作符、RxBinding库

<h2>1.buffer</h2>
概括：定期收集Observable的数据放进一个数据包裹，然后发射这些数据包裹，而不是一次发射一个值。 

我们看看buffer有多少个重载方法：
![这里写图片描述](http://img.blog.csdn.net/20160305142504124)
可见这是一个强大的操作符。

我们举几个典型的来看。
<h3>1.带有timespan参数的方法</h3>
我们例子中就是使用了带有timespan的方法。
```
buffer(long timespan,TimeUnit unit)
```
在2s内收集点击的次数。

他还有一个方法：
```
buffer(long timespan,TimeUnit unit,int count)
```
第三个参数是限定你在这段时间内，最大是多少。 如果放在本例中的话，就是规定点击次数上限。


<h3>2.buffer(int count)、buffer(int count，int skip)</h3>
```
		List<String> list = new ArrayList<String>();
		list.add("1");
		list.add("2");
		list.add("3");
		list.add("4");
		list.add("5");
		list.add("6");
		
		Observable.from(list).buffer(2).subscribe(new Action1<List<String>>() {	
			@Override
			public void call(List<String> arg0) {
				System.out.println(arg0);
			}
		});
```
我们可以看到输出为：

> [1, 2]
[3, 4]
[5, 6]

两两一组~

那么我们就可以猜测一下buffer(2，2)会是什么？
```
Observable.from(list).buffer(2,2).subscribe(new Action1<List<String>>() {	
			@Override
			public void call(List<String> arg0) {
				System.out.println(arg0);
			}
		});
```
输出还是一样：

> [1, 2]
[3, 4]
[5, 6]

我们看看两个方法的源代码：
```
	public final Observable<List<T>> buffer(int count) {
        return buffer(count, count);
    }
    
    public final Observable<List<T>> buffer(int count, int skip) {
        return lift(new OperatorBufferWithSize<T>(count, skip));
    }
```
发现 buffer(2)就是buffer(2,2)~

再举个例子：buffer(2，3)
输出：
> [1, 2]
[4, 5]

所以能**总结出：第二个参数就是用来指定 每次分完组之后 从哪一位开始算起~**

如果还不明白你看看这个例子，比如list中有1-8的数字，那么buffer(2,4)答案是什么呢？
> [1, 2]
[5, 6]

如果答对了，说明你就真的懂了

<h3>3.其他的方法</h3>
其他的方法呢，我也不太了解，也没有用过。 看文档也不是特别懂，这里就不展开讲了~

有兴趣的同学可以看看这里：[ReactiveX文档中文翻译-Buffer](https://mcxiaoke.gitbooks.io/rxdocs/content/operators/Buffer.html)

<h2>2.RxBinding</h2>
RxBinding库是大名鼎鼎[JakeWharton](https://github.com/JakeWharton)大神的又一力作。

如果没有听过真的是要好好去学习膜拜一下，目前他流行库有：ButterKnife、NineOldAndroids等，而且他还是Square的工程师，Square也有大量的开源库，如OkHttp等。

他可以说是Android开发界最出名的大神了。

RxBinding主要是将View和RxJava绑定起来，使得View可以用RxJava的API来处理点击事件等等操作。


像我们在为控件实现点击事件的时候，我们都是findViewById然后setOnclickListener再处理时间，而经过RxBinding的处理，我们只要这样：
```
RxView.clickEvents(btn).subscribe(new Observer<ViewClickEvent>() {
            @Override
            public void onCompleted() {
                //不会回调这里
            }

            @Override
            public void onError(Throwable e) {

            }

            @Override
            public void onNext(ViewClickEvent viewClickEvent) {
                //进行事件处理
            }
        });
```
是不是感觉碉堡了！

然而这只是RxBinding的冰山一角。 比如我们有时候希望用户点击了这个按钮之后。（网络请求） ，在一段时间内再次点击我们就可以写：
```
RxView.clickEvents(button)
    .throttleFirst(500, TimeUnit.MILLISECONDS)//500ms内不得点击
    .subscribe(...);
```
是不是又碉堡了！

如此优雅简洁的代码，简直是开发神器。

<h1>结束</h1>
<h2>1.小结</h2>
实际上我觉得buffer是一个很难使用的操作符，感觉除了点击Button来计算用户点击次数之外，我确实没有想到有什么场景能够使用！  所以说RxJava的痛苦在于例子实在很难想出来啊！ 希望有想到应用场景的童鞋能在楼下留言~

RxBinding是一个极其强大的View相关的库，如果我们学习了RxJava并打算用在实际开发中，那么学习RxBinding更是如虎添翼。 但我也处于RxJava的学习阶段，所以上面就没有展开RxBinding。 希望学习完RxJava之后，有机会给大家带来RxBinding的讲解

<h2>2.附录：完整源代码</h2>
```
public class BufferDemoFragment
        extends BaseFragment {

    @Bind(R.id.list_threading_log)
    ListView _logsList;
    @Bind(R.id.btn_start_operation)
    Button _tapBtn;

    private LogAdapter _adapter;
    private List<String> _logs;

    private Subscription _subscription;

    @Override
    public void onStart() {
        super.onStart();
        _subscription = _getBufferedSubscription();
    }

    @Override
    public void onPause() {
        super.onPause();
        _subscription.unsubscribe();
    }

    @Override
    public void onActivityCreated(@Nullable Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        _setupLogger();
    }

    @Override
    public View onCreateView(LayoutInflater inflater,
                             @Nullable ViewGroup container,
                             @Nullable Bundle savedInstanceState) {
        View layout = inflater.inflate(R.layout.fragment_buffer, container, false);
        ButterKnife.bind(this, layout);
        return layout;
    }

    @Override
    public void onDestroyView() {
        super.onDestroyView();
        ButterKnife.unbind(this);
    }

    // -----------------------------------------------------------------------------------
    // Main Rx entities
    private Subscription _getBufferedSubscription() {
        return RxView.clickEvents(_tapBtn)
                .map(new Func1<ViewClickEvent, Integer>() {
                    @Override
                    public Integer call(ViewClickEvent onClickEvent) {
                        Timber.d("--------- GOT A TAP");
                        _log("GOT A TAP");
                        return 1;
                    }
                })
                .buffer(2, TimeUnit.SECONDS)
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Observer<List<Integer>>() {
                    @Override
                    public void onCompleted() {
                        // fyi: you'll never reach here
                        Timber.d("----- onCompleted");
                    }

                    @Override
                    public void onError(Throwable e) {
                        Timber.e(e, "--------- Woops on error!");
                        _log("Dang error! check your logs");
                    }

                    @Override
                    public void onNext(List<Integer> integers) {
                        Timber.d("--------- onNext");
                        if (integers.size() > 0) {
                            _log(String.format("%d taps", integers.size()));
                        } else {
                            Timber.d("--------- No taps received ");
                        }
                    }
                });
    }

    // -----------------------------------------------------------------------------------
    // Methods that help wiring up the example (irrelevant to RxJava)

    private void _setupLogger() {
        _logs = new ArrayList<>();
        _adapter = new LogAdapter(getActivity(), new ArrayList<String>());
        _logsList.setAdapter(_adapter);
    }

    private void _log(String logMsg) {

        if (_isCurrentlyOnMainThread()) {
            _logs.add(0, logMsg + " (main thread) ");
            _adapter.clear();
            _adapter.addAll(_logs);
        } else {
            _logs.add(0, logMsg + " (NOT main thread) ");

            // You can only do below stuff on main thread.
            new Handler(Looper.getMainLooper()).post(new Runnable() {

                @Override
                public void run() {
                    _adapter.clear();
                    _adapter.addAll(_logs);
                }
            });
        }
    }

    private boolean _isCurrentlyOnMainThread() {
        return Looper.myLooper() == Looper.getMainLooper();
    }
}
```

**-Hans 2016.3.5 17:10**