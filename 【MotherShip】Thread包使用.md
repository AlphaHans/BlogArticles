---
title: 【MotherShip】Thread包使用
date: 2016.4.11 19:03
tag: MotherShip
---
<!-- more -->
# 简单使用
```
Tasker<LoginEntity> tasker = new Tasker<LoginEntity>(ThreadPoolConst.THREAD_TYPE_SIMPLE_HTTP) {
    @Override
    protected LoginEntity doInBackground() {
        Response response = getResult(map);//进行了网络请求
        LoginEntity loginEntity = null;
        try {
            String result = response.body().string();//获取网络请求结果
            Gson gson = new Gson();//解析
            loginEntity = gson.fromJson(result, LoginEntity.class);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return loginEntity;
    }
    @Override
    protected void onPostExecute(LoginEntity data) {
        super.onPostExecute(data);
        System.out.println(data.toString());//主线程回调
    }
};

tasker.start();
```
这是一个简单的使用例子。

# 源代码解析
这个是MotherShip里面关于Thread有关的代码。  类不是很多，我们可以来看看。

## 1.ThreadPoolConst
首先看一下线程池常量的代码。
```
public class ThreadPoolConst {
   /**
    * 普通工作线程池
    */
   public static final int THREAD_TYPE_WORK = 0;
   
   /**
    * 接口请求线程池
    */
   public static final int THREAD_TYPE_SIMPLE_HTTP = 1;
   
   /**
    * 文件传输线程池
    */
   public static final int THREAD_TYPE_FILE_HTTP = 2;
   
   /**
    * 其他线程池
    */
   public static final int THREAD_TYPE_OTHERS = 3;
   
   /**
    * 空闲线程存活时间,5秒
    */
   public static final long KEEP_ALIVE_TIME = 5000;

   /**
    * 有界队列长度
    */
   public final static int DEFAULT_WORK_QUEUE_SIZE = 10;
}
```
我们看到这里面代码很简单：
1.声明了线程池类型，有普通工作线程池、请求接口线程池、文件传输线程池、其他线程池
2.声明了线程池中线程的配置，空余线程存活时间、默认线程数量。（需要注意，线程池有core和额外线程之分）代码中DEFAULT_WORK_QUEUE_SIZE（暂时没用到，不过可能愿意是用来定义线程池Core线程数量）、而额外开的线程我们都会生命一个存活时间KEEP_ALIVE_TIME，来规定线程不使用的时候自动回收。

## 2.ThreadPoolParams
刚面说得是声明了线程池的常量配置，那么这个有什么不同呢？

上面我们说了，里面声明了四种线程池，那这个类就是用来帮我们定义这四种线程池，不同的配置的。

```
public enum ThreadPoolParams {
   /**
    * json网络请求线程池
    */
   jsonHttpThreadPool(ThreadPoolConst.THREAD_TYPE_SIMPLE_HTTP,10,40,ThreadPoolConst.KEEP_ALIVE_TIME, 10000, false),
   
   /**
    * 文件传输线程池
    */
   fileHttpThreadPool(ThreadPoolConst.THREAD_TYPE_FILE_HTTP,15,40,ThreadPoolConst.KEEP_ALIVE_TIME, 10000, false),
   
   /**
    * 工作线程池
    */
   workThreadPool(ThreadPoolConst.THREAD_TYPE_WORK,10,40,ThreadPoolConst.KEEP_ALIVE_TIME, 10000, false),
   
   /**
    * 其他线程
    */
   othersThreadPool(ThreadPoolConst.THREAD_TYPE_OTHERS,10,10,ThreadPoolConst.KEEP_ALIVE_TIME, 10000, false);
   
   /**
    * 核心线程大小:线程池中存在的线程数，包括空闲线程(就是还在存活时间内，没有干活，等着任务的线程)
    */
   private int corePoolSize = 0;
   
   /**
    * 线程池维护线程的最大数量,当基础线程池以及队列都满了的情况继续创建新线程
    */
   private int maximumPoolSize = 0;
   
   /**
    * 线程池维护线程所允许的空闲时间,空闲时间超出该值则移除
    */
   private long keepAliveTime = 0;
   
   /**
    * 线程池类型
    */
   private int type = 0;
   
   /**
    * 线程池所使用的缓冲队列
    */
   private int poolQueueSize = 0;
   
   /**
    * 是否可以超时
    */
   private boolean allowCoreThreadTimeOut = true;
   
   private ThreadPoolParams(int type, int corePoolSize,int maximumPoolSize,long keepAliveTime, int poolQueueSize, boolean allowCoreThreadTimeOut) {
      this.type = type;
      this.corePoolSize = corePoolSize;
      this.maximumPoolSize = maximumPoolSize;
      this.keepAliveTime = keepAliveTime;
      this.poolQueueSize = poolQueueSize;
      this.allowCoreThreadTimeOut = allowCoreThreadTimeOut;
   }
   
   public static ThreadPoolParams getInstance(int type) {
      for(ThreadPoolParams threadPoolParams : ThreadPoolParams.values())
      {
         if(type == threadPoolParams.getType())
         {
            return threadPoolParams;
         }
      }
      return ThreadPoolParams.othersThreadPool;
   }
   //... 其他一些get/set方法
  }
```

**注意！！！** 这个是一个enum 枚举类，它枚举了这些线程池的类型：JsonHttpThreadPool（Json网络请求线程池）、FileHttpThreadPool（文件传输线程池）、WorkThreadPool（工作线程池）、OthersThreadPool（其他线程池） 

那有什么用呢？ 很明显我们看到上面类截图有一个ThreadPoolManager，这个ThreadPoolParams就是用来辅助Manager来对不同的Tasker分派到不同的ThreadPool中处理的类。

我们可以看到
```
public static ThreadPoolParams getInstance(int type) {....}
```
就是根据刚刚ThreadPoolConst 里面的几种type类型，获取相对应的线程池配置（ThreadPoolConst.THREAD_TYPE_SIMPLE_HTTP,10,40,ThreadPoolConst.KEEP_ALIVE_TIME, 10000, false）。这里面的这些东西都是配置，它为不同的类型的线程池，配置了不同的线程数量，线程池最大数量，线程池队列大小，是否允许线程池Core线程超时等。

## 3.BaseThreadPool
有了线程池的配置，那么是不是应该就可以看看如何创建一个线程池了。

```
public class BaseThreadPool extends ThreadPoolExecutor {
   public BaseThreadPool(ThreadPoolParams threadPoolParamter){
      super(threadPoolParamter.getCorePoolSize(), 
           threadPoolParamter.getMaximumPoolSize(), 
           threadPoolParamter.getKeepAliveTime(), 
           TimeUnit.MILLISECONDS, //线程池维护线程所允许的空闲时间的单位
           new LinkedBlockingDeque<Runnable>(threadPoolParamter.getPoolQueueSize()), //线程池所使用的缓冲队列
           new CallerRunsPolicy());//线程池对拒绝任务的处理策略,重试添加当前的任务，他会自动重复调用execute()方法
      if (Build.VERSION.SDK_INT > 10) {
         this.allowCoreThreadTimeOut(threadPoolParamter.isAllowCoreThreadTimeOut());
      }

   }
}
```
我们可以看到，这个类很简单，就是继承了ThreadPoolExecutor 线程池类，然后传入我们的ThreadPoolParams线程池配置参数，创建对应的线程池。

这个类也是放在ThreadPoolManager里面。如果进行任务分派的时，如果没有对应Type的线程池，我们传入线程池配置参数，然后创建对应的线程池~！

## 4.IThreadPoolManager和ThreadPoolManager
这个就是我们的核心！ 线程池管理者，这里面对不同的Tasker进行分派到不同的线程池中进行任务的执行。

一般这样的核心，为了保持扩展性，我们都可以抽象出一个接口，然后再实现，提供给外部进行定制。

看看IThreadPoolManager的代码
```
public interface IThreadPoolManager {
   /**
    * 往线程池中增加一个线程任务
    * @param tasker 线程任务
    */
   public <T> void addTask(Tasker<T> tasker);

   /**
    * 
    * @description:获取指定类型的线程池，如果不存在则创建
    * @param @param ThreadPoolType
    * @return BaseThreadPool
    * @throws
    */
   public BaseThreadPool getThreadPool(int threadPoolType);
   
   /**
    * 从线程队列中移除一个线程任务
    * @param tasker 线程任务
    * @return 是否移除成功
    */
   public <T> boolean removeTask(Tasker<T> tasker);
   
   /**
    * 停止所有任务
    */
   public void stopAllTask();
}
```
很简单，就四个方法，添加任务、移除任务、停止所有任务、以及获取指定Type类型的线程池。

如果理解了以上的类，那么ThreadPoolManager其实看一下代码，一下就可以理解了，直接贴源代码：
```
public class ThreadPoolManager implements IThreadPoolManager {
   private static final String TAG = "MotherShip.ThreadPoolManager";
   /**
    * 不同类型的线程池，可以同时管理多个线程池
    */
   @SuppressLint("UseSparseArrays")
   private final Map<Integer, BaseThreadPool> threadPoolMap = new HashMap<Integer, BaseThreadPool>();
   public static Handler handler = new Handler(Looper.getMainLooper());

   @Override
   public <T> void addTask(final Tasker<T> tasker) {
      if(tasker != null)
      {
         BaseThreadPool threadPool = null;
         synchronized(threadPoolMap)
         {
            threadPool = threadPoolMap.get(tasker.getThreadPoolType());
            //指定类型的线程池不存在则创建一个新的
            if (threadPool == null)
            {
               threadPool = new BaseThreadPool(ThreadPoolParams.getInstance(tasker.getThreadPoolType()));
               threadPoolMap.put(tasker.getThreadPoolType(), threadPool);
            }
         }

         Callable<T> call = new Callable<T>() {
            @Override
            public T call() throws Exception {
               return postResult(tasker, tasker.doInBackground());
            }
         };

         FutureTask<T> task = new FutureTask<T>(call) {
            @Override
            protected void done() {
               try {
                  get();
               } catch (InterruptedException e) {
                  Log.e(TAG, e);

                  tasker.abort();
                  postCancel(tasker);
                  e.printStackTrace();
               } catch (ExecutionException e) {
                  Log.e(TAG, e.getMessage());
                  e.printStackTrace();
                  throw new RuntimeException("An error occured while executing doInBackground()", e.getCause());
               } catch (CancellationException e) {
                  tasker.abort();
                  postCancel(tasker);
                  Log.e(TAG, e);
                  e.printStackTrace();
               }
            }
         };
         tasker.setTask(task);
         threadPool.execute(task);
      }
   }

   /**
    * 将子线程结果传递到UI线程
    *
    * @param tasker
    * @param result
    * @return
    */
   private <T> T postResult(final Tasker<T> tasker, final T result) {
      handler.post(new Runnable() {
         @Override
         public void run() {
            tasker.onPostExecute(result);
         }
      });
      return result;
   }

   /**
    * 将子线程结果传递到UI线程
    * @mainThread
    * @param tasker
    * @return
    */
   private void postCancel(final Tasker tasker) {
      handler.post(new Runnable() {
         @Override
         public void run() {
            tasker.onCanceled();
         }
      });
   }

   @Override
   public BaseThreadPool getThreadPool(int threadPoolType) {
      BaseThreadPool threadPool = null;
      synchronized(threadPoolMap)
      {
         threadPool = threadPoolMap.get(threadPoolType);
         //指定类型的线程池不存在则创建一个新的
         if (threadPool == null)
         {
            threadPool = new BaseThreadPool(ThreadPoolParams.getInstance(threadPoolType));
         }
      }
      
      return threadPool;
   }

   @Override
   public boolean removeTask(Tasker tasker) {
      BaseThreadPool threadPool = threadPoolMap.get(tasker.getThreadPoolType());

      if (threadPool != null)
      {
         return threadPool.remove(tasker.getTask());
      }
      
      return false;
   }
   
   @Override
   public void stopAllTask() {
      if (threadPoolMap != null)
      {
         for (Integer key : threadPoolMap.keySet())
         {
            BaseThreadPool threadPool = threadPoolMap.get(key);
            
            if (threadPool != null)
            {
               threadPool.shutdownNow();//试图停止所有正在执行的线程，不再处理还在池队列中等待的任务
            }
         }
         
         threadPoolMap.clear();
      }
   }
}
```
结合里面的注释，看懂也是分分钟的了~

## 5.Tasker
前面那些理解了，那这个也是分分钟的了。
```
public abstract class Tasker<T> {

    protected abstract T doInBackground();

    protected void onPostExecute(T data) {
    }

    protected void onCanceled() {
    }

    protected void abort() {
    }

    /**
     * 线程池类型
     */
    protected int threadPoolType;

    protected String taskName = null;

    public Tasker(int threadPoolType, String threadTaskName) {
        initThreadTaskObject(threadPoolType, threadTaskName);
    }

    public Tasker(int threadPoolType) {
        initThreadTaskObject(threadPoolType, this.toString());
    }

    /**
     * 在默认线程池中执行
     */
    public Tasker() {
        initThreadTaskObject(ThreadPoolConst.THREAD_TYPE_WORK, this.toString());
    }

    /**
     * 初始化线程任务
     *
     * @param threadPoolType 线程池类型
     * @param threadTaskName 线程任务名称
     */
    private void initThreadTaskObject(int threadPoolType, String threadTaskName) {
        this.threadPoolType = threadPoolType;
        String name = ThreadPoolParams.getInstance(threadPoolType).name();
        if (threadTaskName != null) {
            name = name + "_" + threadTaskName;
        }

        setTaskName(name);
    }

    /**
     * 取得线程池类型
     *
     * @return
     */
    public int getThreadPoolType() {
        return threadPoolType;
    }

    /**
     * 开始任务
     */
    public void start() {
        ThreadPoolFactory.getThreadPoolManager().addTask(this);
    }


    /**
     * 取消任务
     */
    public void cancel() {
        ThreadPoolFactory.getThreadPoolManager().removeTask(this);
    }

    public String getTaskName() {
        return taskName;
    }

    public void setTaskName(String taskName) {
        this.taskName = taskName;
    }

    private FutureTask<T> task;

    /**
     * 设置当前任务
     * @param task
     */
    public void setTask(FutureTask<T> task) {
        this.task = task;
    }

    /**
     * 获取当前任务
     * @return
     */
    public FutureTask<T> getTask() {
        return task;
    }
}
```
需要注意的是，ThreadPoolManager的获取方式，是通过ThreadPoolFactory来获取的。 
```
public class ThreadPoolFactory {
   public static IThreadPoolManager getThreadPoolManager()
   {
      return SingletonFactory.getInstance(ThreadPoolManager.class);
   }
}
```
这个是MotherShip里面通过单例工厂进行直接创建的，这样在开发中，我们可以写少很多关于获取单例模式的代码。



