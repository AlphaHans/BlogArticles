---
title: 【RxJava Demo分析】（二）Schedulers线程调度器
date: 2016-03-05 13:36
tag: RxJava
category: RxJava
---
<h1>ConcurrencyWithSchedulersDemoFragment</h1>
用Schedulers(调度器)实现多任务(并发，Concurrency)的例子

<!-- more -->
废话不多说我们看一下有关于RxJava的代码：

```
	private Subscription _subscription;

    @OnClick(R.id.btn_start_operation) //
    public void startLongOperation() {
        _progress.setVisibility(View.VISIBLE);
        _log("Button Clicked");

        _subscription = _getObservable()//
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(_getObserver());                             // Observer
    }

    private Observable<Boolean> _getObservable() {
        return Observable.just(true).map(new Func1<Boolean, Boolean>() {
            @Override
            public Boolean call(Boolean aBoolean) {
                _log("Within Observable");
                _doSomeLongOperation_thatBlocksCurrentThread();//长时间的操作，阻塞线程
                return aBoolean;
            }
        });
    }

    private Observer<Boolean> _getObserver() {
        return new Observer<Boolean>() {
			//....一些操作
        };
    }
```
这段代码比较简单，只涉及了一个知识点Schedulers调度器
<h2>Schedulers</h2>
概括：线程调度器，用来达到切换线程的效果。

在Android开发中，由于UI线程是不能够被阻塞的，不然就会产生ANR导致程序崩溃。所以我们经常的处理是，两种方法：
<h3>1.新开线程（或者线程池中）处理阻塞的操作：</h3>
```
        new Thread(new Runnable() {
            @Override
            public void run() {
                try{
                    mEntity = getEntity();
                    mHandler.obtainMessage(1,mEntity).sendToTarget();
                }catch (Exception e){
                    mHandler.sendEmptyMessage(0);
                }
            }
        }).start();
```
大家可以看到，我们这里只是进行了一个简单的IO操作，就陷入了这么谜一般的缩进。要是设计更加复杂的呢？ 

而且我们还要用Handler处理消息，造成了编码的撕裂感！ 虽然我个人对Handler并不反感，但在编码的时候，它的存在的确让我感到有点惆怅~~
<h3>2.回调处理</h3>
```
		HttpClient.getInstance().get("http://www.qq.com",new HttpCallback(){
            public void onSuccess(String result){
                //...成功操作
            }

            public void onError(Exception e){
                //...失败操作
            }
        });
```
这里我们随便写了一个获取QQ主页的首页数据的操作。 虽然现在看起来还是蛮简单的。 只有一层回调，但是要是我们要针对QQ返回的结果，再进行一次热门词汇的搜索文章，搜索出来的文章我还要获取第五篇文章，然后我再获取第五篇文章的评论呢？

这里就会陷入Callback Hell （回调地狱）！ 维护起来真的是太恶心了！

<h3>3.小结一下Schedulers</h3>
Schedulers调度器的出现，保持了我们使用RxJava的链式调用，不会出现撕裂感，因为它可以帮我们游刃有余的进行线程切换，而不需要进行Hanlder或者回调！ 

也正是Schedulers的出现才让我下定决心学习RxJava的初衷！ 这实在的太棒了（当然还有无穷无尽的操作符~~）

<h1>结束</h1>
<h2>1.关于Schedulers的原理</h2>
因为能力还没有到这里层度，毕竟RxJava和一般的库不一样，它里面涉及了很多“高端知识”而我也不敢乱说出来误导。 所以呢~这里附上前辈的分析吧（反正我看的是有点一头雾水）：[给 Android 开发者的 RxJava 详解](http://gank.io/post/560e15be2dca930e00da1083#toc_14) 里面有一部分是涉及Schedulers分析的。

我觉得要是看不懂也没关系，随着我们使用的越来越多，没准在哪一天心血来潮想看看源码的时候突然就明白了呢~

在未来的学习，要是我突然明白了，我会尽快写一篇文章和大家讨论的~

<h2>2.附录：完整源代码</h2>
```
public class ConcurrencyWithSchedulersDemoFragment
        extends BaseFragment {

    @Bind(R.id.progress_operation_running)
    ProgressBar _progress;
    @Bind(R.id.list_threading_log)
    ListView _logsList;

    private LogAdapter _adapter;
    private List<String> _logs;
    private Subscription _subscription;

    @Override
    public void onDestroy() {
        super.onDestroy();
        RxUtils.unsubscribeIfNotNull(_subscription);
        ButterKnife.unbind(this);
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
        View layout = inflater.inflate(R.layout.fragment_concurrency_schedulers, container, false);
        ButterKnife.bind(this, layout);
        return layout;
    }

    @OnClick(R.id.btn_start_operation)
    public void startLongOperation() {
        _progress.setVisibility(View.VISIBLE);
        _log("Button Clicked");

        _subscription = _getObservable()//
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(_getObserver());                             // Observer
    }

    private Observable<Boolean> _getObservable() {

        return Observable.just(true).map(new Func1<Boolean, Boolean>() {
            @Override
            public Boolean call(Boolean aBoolean) {
                _log("Within Observable");
                _doSomeLongOperation_thatBlocksCurrentThread();
                return aBoolean;
            }
        });
    }

    /**
     * Observer that handles the result through the 3 important actions:
     * <p/>
     * 1. onCompleted
     * 2. onError
     * 3. onNext
     */
    private Observer<Boolean> _getObserver() {
        return new Observer<Boolean>() {

            @Override
            public void onCompleted() {
                _log("On complete");
                _progress.setVisibility(View.INVISIBLE);
            }

            @Override
            public void onError(Throwable e) {
                Timber.e(e, "Error in RxJava Demo concurrency");
                _log(String.format("Boo! Error %s", e.getMessage()));
                _progress.setVisibility(View.INVISIBLE);
            }

            @Override
            public void onNext(Boolean bool) {
                _log(String.format("onNext with return value \"%b\"", bool));
            }
        };
    }

    // -----------------------------------------------------------------------------------
    // Method that help wiring up the example (irrelevant to RxJava)

    private void _doSomeLongOperation_thatBlocksCurrentThread() {
        _log("performing long operation");
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            Timber.d("Operation was interrupted");
        }
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

    private void _setupLogger() {
        _logs = new ArrayList<String>();
        _adapter = new LogAdapter(getActivity(), new ArrayList<String>());
        _logsList.setAdapter(_adapter);
    }

    private boolean _isCurrentlyOnMainThread() {
        return Looper.myLooper() == Looper.getMainLooper();
    }

    private class LogAdapter
            extends ArrayAdapter<String> {

        public LogAdapter(Context context, List<String> logs) {
            super(context, R.layout.item_log, R.id.item_log, logs);
        }
    }
}
```

**-Hans 2016.3.5 13:36**