---
title: 【okhttp3源代码分析】（二）Request的enqueue
date: 2016-04-03 22:05
tags: [okhttp,Android]  
category: okhttp
---
<h1>前言</h1>
如果没有阅读本系列文章的第一篇，请先阅读：

[【OkHttp3源代码分析】（一）Request的execute](http://hanszone.xyz/2016/04/03/%E3%80%90OkHttp3%E6%BA%90%E4%BB%A3%E7%A0%81%E5%88%86%E6%9E%90%E3%80%91%EF%BC%88%E4%B8%80%EF%BC%89Request%E7%9A%84execute/)

因为这两者之间是有关联的！

<!-- more -->

<h1>enqueue执行流程源代码分析</h1>
先来看看源代码：
```
  @Override 
  public void enqueue(Callback responseCallback) {
    enqueue(responseCallback, false);//继续调用下面一层
  }

  void enqueue(Callback responseCallback, boolean forWebSocket) {
    synchronized (this) {//跟execute一样，防止多个子线程同时调用执行的方法。
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    client.dispatcher().enqueue(new AsyncCall(responseCallback, forWebSocket));//重点！
  }
```
看到重点的那一行：
```
client.dispatcher().enqueue(new AsyncCall(responseCallback, forWebSocket));//重点！
```
我们发现，它将我们的Callback接口封装到一个新的类AsyncCall中去。

我们来看看它的代码:
```
final class AsyncCall extends NamedRunnable {
    private final Callback responseCallback;
    private final boolean forWebSocket;

    private AsyncCall(Callback responseCallback, boolean forWebSocket) {
      super("OkHttp %s", originalRequest.url().toString());
      this.responseCallback = responseCallback;
      this.forWebSocket = forWebSocket;
    }

    String host() {
      return originalRequest.url().host();
    }

    Request request() {
      return originalRequest;
    }

    Object tag() {
      return originalRequest.tag();
    }

    void cancel() {
      RealCall.this.cancel();
    }

    RealCall get() {
      return RealCall.this;
    }

    @Override  //重点方法！
    protected void execute() {
      boolean signalledCallback = false;
      try {
        Response response = getResponseWithInterceptorChain(forWebSocket);//和execute调用的是同一个！
        if (canceled) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));//回掉接口的失败方法
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);//回掉接口的成功方法
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          logger.log(Level.INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);//移除请求
      }
    }
  }
```
要注意它是继承NamedRunnable的，这里我们从名字也可以看出，实际上它就是一个Runnable了。

简单看看NamedRunnable源代码：
```
/**
 * Runnable implementation which always sets its thread name.
 */
public abstract class NamedRunnable implements Runnable {
  protected final String name;

  public NamedRunnable(String format, Object... args) {
    this.name = String.format(format, args);
  }

  @Override public final void run() {
    String oldName = Thread.currentThread().getName();
    Thread.currentThread().setName(name);
    try {
      execute();
    } finally {
      Thread.currentThread().setName(oldName);
    }
  }

  protected abstract void execute();
}
```

了解完之后，我们就可以知道哪个是重要的方法了。 

没错就是：execute方法。（注意这个execute不是我们第一篇文章讲的那个）

然后我们发现它调用了：
```
Response response = getResponseWithInterceptorChain(forWebSocket);//和execute调用的是同一个！
```
实际上这个和Request的execute调用是一样的，只不过多了一个接口回掉的过程。这里我们就不再重复分析了。

OK，了解完AnsycCall类，我们回来来看看这句的enqueue：
```
client.dispatcher().enqueue(new AsyncCall(responseCallback, forWebSocket));//重点！
```
首先我们都知道，OkHttp3中enqueue是异步通过执行，然后通过接口回调的！ 

再加上AsyncCall本质是一个Runnable。

所以用我们程序员本能的嗅觉，其实已经能够大概擦觉到，这个client.dispatcher()通过自己封装的线程池，执行我们的AsyncCall的execute方法了！

但是要是没有察觉到也没有关系，我们看看源代码就知道是什么了！

我们来看看Dispatcher类的enqueue方法：
```
  synchronized void enqueue(AsyncCall call) {
	//如果队列任务没有爆满且小于每个Request的请求次数
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);//添加到队列，和第一篇文章加入队列的目的是一样的，也是方便用来取消任务。
      executorService().execute(call);//用线程池执行
    } else {
      //如果爆满，就放到准备队列
      readyAsyncCalls.add(call);
    }
  }
```
嘿嘿，果然和我们想的没有错，的确是使用了线程池：
```
  public synchronized ExecutorService executorService() {
    if (executorService == null) {
      //配置的线程池。
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
```
**小吐槽...这样写好像有点影响性能的样子。通过synchronized实现单例模式。 还不如使用双重判断锁的方式来实现，可以提升一定的性能，也避免synchronized开销啊！！！**

至此，我们已经完成了enqueue的分析了！ 如果看懂了上篇文章的execute，看懂这个真的只是分分钟的事情了！！

**-Hans 2016.4.3 22:05**