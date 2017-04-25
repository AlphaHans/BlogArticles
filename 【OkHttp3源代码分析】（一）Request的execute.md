---
title: 【okhttp3源代码分析】（一）Request的execute
date: 2016-04-03 21:30
tags: [okhttp,Android]  
category: okhttp
---
# 简单使用OkHttp3
**阅读本文需要对OkHttp3的使用有一定了解。**

首先我们先看看如何简单进行一个get请求的Request。

<!-- more -->

```
Request qqRequest = new Request.Builder()
                        .url("http://www.qq.com")
                        .build();
Call call = mOkHttp.newCall(qqRequest);
call.execute();//特别注意 这里要在子线程执行
```
或者可以使用：
```
Request qqRequest = new Request.Builder()
                        .url("http://www.qq.com")
                        .build();
Call call = mOkHttp.newCall(qqRequest);
//使用enquue
call.enqueue(new Callback() {
	@Override
	public void onFailure(Call call, IOException e) {
	    e.printStackTrace();
    }

	@Override
	public void onResponse(Call call, Response response) throws IOException {
		Log.i(TAG, response.body().string());
	}
});
```

# execute执行流程源代码分析
要看execute就要先看
```
Call call = mOkHttp.newCall(qqRequest);
```
点进去源代码我们发现 返回了一个RealCall对象，因为Call是一个接口，而RealCall才是真正的实现类
```
  /**
   * Prepares the {@code request} to be executed at some point in the future.
   */
  @Override 
  public Call newCall(Request request) {
    return new RealCall(this, request);
  }
```
现在我们找到execute方法的源代码：
```
  @Override 
  public Response execute() throws IOException {
    synchronized (this) {//同步锁，为了防止子线程同时调用而导致出现多次网络请求
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;//这里表明，对应的Call只能执行一次
    }
    try {
      client.dispatcher().executed(this);//加入到请求队列
      Response result = getResponseWithInterceptorChain(false);//核心代码！
      if (result == null) throw new IOException("Canceled");
      return result;
    } finally {
      client.dispatcher().finished(this);//从请求队列中移除
    }
  }
```
首先 我们看下一不太重点的两行代码：
```
···
client.dispatcher().executed(this);//加入到请求队列
···
client.dispatcher().finished(this);//从请求队列中移除
```
这里是用来干嘛的？ 

## 1.Dispatcher辅助管理请求
我把相关代码贴出来：
```
  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();

  /** Used by {@code Call#execute} to signal it is in-flight. */
  synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }

  /** Used by {@code Call#execute} to signal completion. */
  synchronized void finished(Call call) {
    if (!runningSyncCalls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
  }

  /**
   * Cancel all calls currently enqueued or executing. Includes calls executed both {@linkplain
   * Call#execute() synchronously} and {@linkplain Call#enqueue asynchronously}.
   */
  public synchronized void cancelAll() {
    for (AsyncCall call : readyAsyncCalls) {
      call.cancel();
    }

    for (AsyncCall call : runningAsyncCalls) {
      call.cancel();
    }

    for (RealCall call : runningSyncCalls) {
      call.cancel();
    }
  }
```
从上面这段代码可以看出，它是用来将这个请求加入队列中的，方便我们取消任务的。

## 2.ApplicationInterceptorChain与proceed()
搞懂了之后，我们来看看execute执行的那一行核心代码：
```
//核心代码！
Response result = getResponseWithInterceptorChain(false);
```
这里调用了getResponseWithInterceptorChain方法（废话，我够知道）- -#。 我们来看看相关代码：
```
private Response getResponseWithInterceptorChain(boolean forWebSocket) throws IOException {
    Interceptor.Chain chain = new ApplicationInterceptorChain(0, originalRequest, forWebSocket);
    return chain.proceed(originalRequest);
  }

  class ApplicationInterceptorChain implements Interceptor.Chain {
    private final int index;
    private final Request request;
    private final boolean forWebSocket;

    ApplicationInterceptorChain(int index, Request request, boolean forWebSocket) {
      this.index = index;
      this.request = request;
      this.forWebSocket = forWebSocket;
    }

    @Override public Connection connection() {
      return null;
    }

    @Override public Request request() {
      return request;
    }

    @Override public Response proceed(Request request) throws IOException {
      // If there's another interceptor in the chain, call that.
      if (index < client.interceptors().size()) {
        Interceptor.Chain chain = new ApplicationInterceptorChain(index + 1, request, forWebSocket);
        Interceptor interceptor = client.interceptors().get(index);
        Response interceptedResponse = interceptor.intercept(chain);

        if (interceptedResponse == null) {
          throw new NullPointerException("application interceptor " + interceptor
              + " returned null");
        }

        return interceptedResponse;
      }

      // No more interceptors. Do HTTP.
      return getResponse(request, forWebSocket);
    }
  }
```
getResponseWithInterceptorChain里面创建了一个ApplicationInterceptorChain对象，并且调用了它的proceed方法。

我们来看一下传入的这个参数：0。 实际上这个0是有特殊含义的。

这里面涉及了OkHttp3的新特性：拦截器Interceptor。 它可以添加多个拦截器，而这里的0的意思，就是从第1个拦截器开始。

简单了解完之后，我们看看关键的proceed代码
```
	@Override 
	public Response proceed(Request request) throws IOException {
      // If there's another interceptor in the chain, call that.
      // 判断还有没有拦截器？ 有的话先走拦截器
      if (index < client.interceptors().size()) {
        Interceptor.Chain chain = new ApplicationInterceptorChain(index + 1, request, forWebSocket);//创建下一个拦截器的chain
        Interceptor interceptor = client.interceptors().get(index);//获取拦截器
        Response interceptedResponse = interceptor.intercept(chain);//传递过去
		//这里要注意，拦截器返回的Response不能为空
        if (interceptedResponse == null) {
          throw new NullPointerException("application interceptor " + interceptor
              + " returned null");
        }
		//返回结果
        return interceptedResponse;
      }

      // No more interceptors. Do HTTP.
      //当执行完拦截器之后，执行真正的Http请求。
      return getResponse(request, forWebSocket);
    }
```
相信看完上面的注释，我相信大家也有一定的了解了。

有一个地方需要大家注意的，在官方文档中关于Application Interceptors有一句这样的话：
>Permitted to short-circuit and not call Chain.proceed().

意思是，我们可以在拦截器中，不调用proceed()。 但是proceed正是返回我们Response的方法啊！

我不太明白这个设计理念！

就此问题，我专门去OkHttp的Issues提问，有幸获得了JakeWharton大神的回答（没错，那位JakeWharton）
附上链接：
[What's mean "Permitted to short-circuit and not call Chain.proceed()."?](https://github.com/square/okhttp/issues/2460)

请原谅我的渣英语。(逃

JakeWharton大神是这样说的：
![](http://img.blog.csdn.net/20160403211445190)

意思是说：这样的设计是用在Http缓存里面的。执行请求之前，我们可以先从本地读取，看看有没有这个请求的缓存。如果没有，我们就调用proceed的方法！

但是需要再次注意，就算我们不调用这个proceed方法，也一定要返回一个Response对象！ 这样才可以避免抛出空指针异常。:
```
        if (interceptedResponse == null) {
          throw new NullPointerException("application interceptor " + interceptor
              + " returned null");
        }
```

分析完上面之后，我们发现，它调用了：
```
	  // No more interceptors. Do HTTP.
      //当执行完拦截器之后，执行真正的Http请求。
      return getResponse(request, forWebSocket);
```
这里才是真正执行Http请求的地方。

关于getResponse具体内容，我就不再继续分析了....因为需要的篇幅比较大，我会在接下来的文章进行分析！ 先大概了解是用**HttpEngine**这个类来进行请求的！



# 结束
OK！关于execute方法，就分析地差不多了。应该也讲地比较清楚了，请务必搞懂上面的内容，因为enqueue的方法是调用execute的方法的！ 

所以要是看不懂execute...那enqueue也看不懂了！

下篇文章将带来enqueue的源代码流程分析！（虽然你看懂了execute之后，也可以自己尝试看看enqueue的源代码了！）

# 备注
**关于拦截器，这不属于本文范围，将在后续的文章中进行讲解。**
这里也附上一些学习的链接：
[官方Wiki Interceptor](https://github.com/square/okhttp/wiki/Interceptors)

[官方Wiki Interceptor中文翻译](http://www.tuicool.com/articles/Uf6bAnz)

或者可以看我另外一篇文章（这里是项目中的实际应用，品完官方文档，食用更佳）：
[【项目重构】使用OkHttp解决Session过期导致用户掉线的问题](http://hanszone.xyz/2016/03/30/%E3%80%90%E9%A1%B9%E7%9B%AE%E9%87%8D%E6%9E%84%E3%80%91%E4%BD%BF%E7%94%A8OkHttp%E8%A7%A3%E5%86%B3Session%E8%BF%87%E6%9C%9F%E5%AF%BC%E8%87%B4%E7%94%A8%E6%88%B7%E6%8E%89%E7%BA%BF%E7%9A%84%E9%97%AE%E9%A2%98/)

**-Hans 2016.4.3 21:30**