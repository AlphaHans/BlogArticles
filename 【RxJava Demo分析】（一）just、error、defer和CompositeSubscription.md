---
title: 【RxJava Demo分析】（一）just、error、defer和CompositeSubscription
date: 2016-03-05 1:32
tag: RxJava
category: RxJava
---
<h1>前言</h1>
RxJava可能是目前最难以掌握的库之一，面对大量的书籍和理论讲解，我选择结合项目来进行理解和记录。

<!-- more -->
本系列教程是基于Github开源项目[RxJava-Android-Samples](https://github.com/kaushikgopal/RxJava-Android-Samples) 进行分析。  里面有RxJava在Android开发中较为日常的使用操作，所以具有非常好的学习意义啊~ 

但可能因为是英文，可能大家多少有排斥心理，所以我就写下这一系列教程，希望能帮到有意愿学习的童鞋~ 

**注意：因为本人对RxJava也不是非常了解的状态，所以可能会有错误。 希望大家都多批评指正！**

<h1>教程大纲</h1>>
本教程主要是截取该开源项目中每一个例子中使用到RxJava的场景，然后进行分析！

分析的时候，会隐藏无关于RxJava的一些代码，但在每篇最后都会贴出完整的源代码

OK 那我们废话不多说啦，直接进入今天的主题：通过分析VolleyDemoFragment学习just、error、defer、CompositeSubscription。

<h1>VolleyDemoFragment</h1>
```
    //返回一个JSONObject对象
    private JSONObject getRouteData() throws ExecutionException, InterruptedException {
        ...
        return future.get();
    }
    
    //已经创建好的
    public Observable<JSONObject> newGetRouteData() {
        return Observable.defer(new Func0<Observable<JSONObject>>() {
            @Override
            public Observable<JSONObject> call() {
                try {
                    //直接将JSONObject对象转为Observable
                    return Observable.just(getRouteData());
                } catch (InterruptedException | ExecutionException e) {
                    Log.e("routes", e.getMessage());
                    //异常处理
                    return Observable.error(e);
                }
            }
        });
    }
    
     //用了一个奇怪的类~
    private CompositeSubscription _compositeSubscription = new CompositeSubscription();
    
    //点击按钮之后开始
     private void startVolleyRequest() {
        //用了一个奇怪的类~
        _compositeSubscription.add(newGetRouteData().subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Observer<JSONObject>() {
                    ...
                }));
    }
    
```
知识点有:
<h2>1.just操作符</h2>
概括：直接将对象转化为Observable对象

一共有7个重载方法，可以接受1-7个参数：
![这里写图片描述](http://img.blog.csdn.net/20160305000917146)

调用顺序为： n*onNext+onCompleted（0<n<8）

<h2>2.error操作符</h2>
概括：处理异常对象，当调用该方法的时候直接调用onError而不调用onCompleted

<h2>3.defer操作符</h2>
概括：懒加载 保证数据是最新鲜的。
这个操作符有点难以理解

所以我们这边举一个例子：
```
		App app = new App("微信");
		Observable<App> obervable = Observable.just(app);
		app = new App("QQ");
		obervable.subscribe(new Observer<App>() {

			@Override
			public void onCompleted() {
				
			}

			@Override
			public void onError(Throwable arg0) {
				
			}

			@Override
			public void onNext(App a) {
				System.out.println(a.getName());
			}
		});
```
你觉得在onNext输出的信息会是什么？ 是微信还是QQ？

大家可能会觉得，应该是QQ吧？ 这样想就错了！

实际上使用just创建一个Observable，已经将数据保存好了！

这并不是我们希望看到的结果。

这里有两种解决方式：
<h3>1.使用create</h3>
这个操作符可以说是最简单的操作符啦~ 在里面创建这个App对象，再封装成Observable。
```
Observable.create(new OnSubscribe<App>() {
			@Override
			public void call(Subscriber<? super App> s) {
				App a = new App("QQ");
				s.onNext(a);
				s.onCompleted();
			}
		});
```
这段代码在有订阅的时候才会执行 所以里面的信息可以保证是最新的！

但这个方法好像还是不太完美~ 因为i可能我们希望随时更改这个对象，而不是推迟到创建的时候。

<h3>2.使用defer</h3>
终于来到重点啦~ 使用这个defer操作符 封装原来的just代码，就可以保证数据是最新的！

```
private App mApp = new App("微信");

Observable<App> deferObservable = Observable.defer(new Func0<Observable<App>>() {
			@Override
			public Observable<App> call() {
				return Observable.just(mApp);
			}
		});
		mApp = new App("QQ");
		deferObservable.subscribe(new Observer<App>() {

			@Override
			public void onCompleted() {
				
			}

			@Override
			public void onError(Throwable arg0) {
				
			}

			@Override
			public void onNext(App a) {
				System.out.println(a.getName());
			}
		});
```
被defer封装的操作会推迟到 当有订阅的时候才会执行。所以这就是它的奥秘啦~

这里附上相关分析：[RxJava的懒加载，慎重使用自定义操作符，优先考虑内置操作符](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0821/3337.html)

<h2>3.CompositeSubscription</h2>
这个我们先看到 源码注释：
```
/**
 * Subscription that represents a group of Subscriptions that are unsubscribed together.
 * <p>
 * All methods of this class are thread-safe.
 */
```
概括：线程安全、由所有订阅者组的组

看看我们上面的实例代码：
```
_compositeSubscription.add(...);
```
他将所有的订阅者都添加进去，然后再Activity onPause或者onDestroy时候统一取消订阅，避免造成内存泄漏：
```
_compositeSubscription.unsubscribe();
```
<h1>结束</h1>
<h2>1.小结</h2>
好的今天的教程就到这里了，我觉得应该每一个都讲的很详细了吧。 希望大家都能理解到啦~~

<h2>2.附录：完整源代码</h2>
```
public class VolleyDemoFragment extends BaseFragment {

    public static final String TAG = "VolleyDemoFragment";

    @Bind(R.id.list_threading_log)
    ListView _logsList;

    private List<String> _logs;
    private LogAdapter _adapter;

    private CompositeSubscription _compositeSubscription = new CompositeSubscription();

    @Override
    public View onCreateView(LayoutInflater inflater,
                             @Nullable ViewGroup container,
                             @Nullable Bundle savedInstanceState) {
        View layout = inflater.inflate(R.layout.fragment_volley, container, false);
        ButterKnife.bind(this, layout);
        return layout;
    }

    @Override
    public void onActivityCreated(@Nullable Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        _setupLogger();
    }

    @Override
    public void onPause() {
        super.onPause();
        _compositeSubscription.unsubscribe();
    }


    @Override
    public void onDestroy() {
        super.onDestroy();
    }

    @Override
    public void onDestroyView() {
        super.onDestroyView();
        ButterKnife.unbind(this);
    }

    public Observable<JSONObject> newGetRouteData() {
        return Observable.defer(new Func0<Observable<JSONObject>>() {
            @Override
            public Observable<JSONObject> call() {
                try {
                    return Observable.just(getRouteData());
                } catch (InterruptedException | ExecutionException e) {
                    Log.e("routes", e.getMessage());
                    return Observable.error(e);
                }
            }
        });
    }

    @OnClick(R.id.btn_start_operation)
    void startRequest() {
        startVolleyRequest();
    }

    private void startVolleyRequest() {
        _compositeSubscription.add(newGetRouteData().subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Observer<JSONObject>() {
                    @Override
                    public void onCompleted() {
                        Log.e(TAG, "onCompleted");
                        Timber.d("----- onCompleted");
                        _log("onCompleted ");
                    }

                    @Override
                    public void onError(Throwable e) {
                        VolleyError cause = (VolleyError) e.getCause();
                        String s = new String(cause.networkResponse.data, Charset.forName("UTF-8"));
                        Log.e(TAG, s);
                        Log.e(TAG, cause.toString());
                        _log("onError " + s);

                    }

                    @Override
                    public void onNext(JSONObject jsonObject) {
                        Log.e(TAG, "onNext " + jsonObject.toString());
                        _log("onNext " + jsonObject.toString());

                    }
                }));
    }

    private JSONObject getRouteData() throws ExecutionException, InterruptedException {
        RequestFuture<JSONObject> future = RequestFuture.newFuture();
        String url = "http://www.weather.com.cn/adat/sk/101010100.html";
        final Request.Priority priority = Request.Priority.IMMEDIATE;
        JsonObjectRequest req = new JsonObjectRequest(Request.Method.GET, url, future, future);
        MyVolley.getRequestQueue().add(req);
        return future.get();
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
小吐槽：我对于该作者的代码风格表示有点...  

**-Hans 2016.3.5 1:32**