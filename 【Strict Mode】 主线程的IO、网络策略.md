---
title: 【Strict Mode】 主线程的I\O、网络策略
date: 2016-05-14 15:00
tag: Strict Mode
category: Android进阶
---

<!-- more -->
# StrictMode介绍

****

StrictMode is a developer tool which detects things you might be doing by accident and brings them to your attention so you can fix them.

StrictMode is most commonly used to catch accidental disk or network access on the application's main thread, where UI operations are received and animations take place. Keeping disk and network operations off the main thread makes for much smoother, more responsive applications. By keeping your application's main thread responsive, you also prevent ANR dialogs from being shown to users.

> Note that even though an Android device's disk is often on flash memory, many devices run a filesystem on top of that memory with very limited concurrency. It's often the case that almost all disk accesses are fast, but may in individual cases be dramatically slower when certain I/O is happening in the background from other processes. If possible, it's best to assume that such things are not fast.



上面的这一段话可以一句话概括：

**严格模式是一个帮助开发者在开发App时候的工具。 它通常用来捕捉在主线程进行本地I\O操作和网络操作。 **



这个类里面有一个很重要的方法：**setThreadPolicy** 对当前线程设置策略。



Android系统线程分为两类：1.主线程 2.子线程

对于这两种线程的策略当然是不一样的，最明显的特点就是主线程不允许进行网络和IO操作





# 测试用例

****

```

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    mEtUrl = (EditText) findViewById(R.id.main_et_url);
    mBtnSend = (Button) findViewById(R.id.main_btn_send);
    try {
        URL url = new URL("http://www.qq.com");
        HttpURLConnection conn = (HttpURLConnection) url.openConnection();
        InputStream in = conn.getInputStream();
        if (in != null) {

        }
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

在主线程进行网络请求。 



异常堆栈信息：

```

java.lang.RuntimeException: Unable to start activity ComponentInfo{xyz.hans.remoteapplicationtest/xyz.hans.remoteapplicationtest.MainActivity}: android.os.NetworkOnMainThreadException

                                                       at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2358)

                                                       at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2420)

                                                       at android.app.ActivityThread.access$900(ActivityThread.java:154)

                                                       at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1321)

                                                       at android.os.Handler.dispatchMessage(Handler.java:102)

                                                       at android.os.Looper.loop(Looper.java:135)

                                                       at android.app.ActivityThread.main(ActivityThread.java:5294)

                                                       at java.lang.reflect.Method.invoke(Native Method)

                                                       at java.lang.reflect.Method.invoke(Method.java:372)

                                                       at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:904)

                                                       at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:699)

                                                    Caused by: android.os.NetworkOnMainThreadException

                                                       at android.os.StrictMode$AndroidBlockGuardPolicy.onNetwork(StrictMode.java:1147)

                                                       at java.net.InetAddress.lookupHostByName(InetAddress.java:418)

                                                       at java.net.InetAddress.getAllByNameImpl(InetAddress.java:252)

                                                       at java.net.InetAddress.getAllByName(InetAddress.java:215)

                                                       at com.android.okhttp.HostResolver$1.getAllByName(HostResolver.java:29)

                                                       at com.android.okhttp.internal.http.RouteSelector.resetNextInetSocketAddress(RouteSelector.java:232)

                                                       at com.android.okhttp.internal.http.RouteSelector.next(RouteSelector.java:124)

                                                       at com.android.okhttp.internal.http.HttpEngine.connect(HttpEngine.java:272)

                                                       at com.android.okhttp.internal.http.HttpEngine.sendRequest(HttpEngine.java:211)

                                                       at com.android.okhttp.internal.http.HttpURLConnectionImpl.execute(HttpURLConnectionImpl.java:382)

                                                       at com.android.okhttp.internal.http.HttpURLConnectionImpl.getResponse(HttpURLConnectionImpl.java:332)

                                                       at com.android.okhttp.internal.http.HttpURLConnectionImpl.getInputStream(HttpURLConnectionImpl.java:199)

                                                       at xyz.hans.remoteapplicationtest.MainActivity.onCreate(MainActivity.java:30)

                                                       at android.app.Activity.performCreate(Activity.java:5990)

                                                       at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1106)

                                                       at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2311)

                                                       at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2420) 

                                                       at android.app.ActivityThread.access$900(ActivityThread.java:154) 

                                                       at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1321) 

                                                       at android.os.Handler.dispatchMessage(Handler.java:102) 

                                                       at android.os.Looper.loop(Looper.java:135) 

                                                       at android.app.ActivityThread.main(ActivityThread.java:5294) 

                                                       at java.lang.reflect.Method.invoke(Native Method) 

                                                       at java.lang.reflect.Method.invoke(Method.java:372) 

                                                       at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:904) 

                                                       at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:699)

```

重点信息

**Caused by: android.os.NetworkOnMainThreadException at android.os.StrictMode$AndroidBlockGuardPolicy.onNetwork(StrictMode.java:1147)**

就是在这个StrictMode类中的内部类AndroidBlockGuardPolicy的onNetwork方法抛出的。

看看这个方法：
```
// Part of BlockGuard.Policy interface:
public void onNetwork() {
    if ((mPolicyMask & DETECT_NETWORK) == 0) {
        return;
    }
    if ((mPolicyMask & PENALTY_DEATH_ON_NETWORK) != 0) {
        throw new NetworkOnMainThreadException();
    }
    if (tooManyViolationsThisLoop()) {
        return;
    }
    BlockGuard.BlockGuardPolicyException e = new StrictModeNetworkViolation(mPolicyMask);
    e.fillInStackTrace();
    startHandlingViolationException(e);
}
```
第二个判断，就可以看到在主线程进行网络请求抛出的异常了。

# ThreadProlicy
****
如何制定一个线程的策略？

官方文档给出了一个例子：
```
 StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder()
                 .detectDiskReads() //探测硬盘读
                 .detectDiskWrites() //探测硬盘写
                 .detectNetwork()// 探测网络请求 or .detectAll() for all detectable problems（或者直接调用detectAlll探测所有可探测的问题）
                 .penaltyLog()
                 .build());
```

具体有什么策略呢？

附录（ThreadPolicy）：
```
/**
 * {@link StrictMode} policy applied to a certain thread.
 *
 * <p>The policy is enabled by {@link #setThreadPolicy}.  The current policy
 * can be retrieved with {@link #getThreadPolicy}.
 *
 * <p>Note that multiple penalties may be provided and they're run
 * in order from least to most severe (logging before process
 * death, for example).  There's currently no mechanism to choose
 * different penalties for different detected actions.
 */
public static final class ThreadPolicy {
    /**
     * The default, lax policy which doesn't catch anything.
     */
    public static final ThreadPolicy LAX = new ThreadPolicy(0);

    final int mask;

    private ThreadPolicy(int mask) {
        this.mask = mask;
    }

    @Override
    public String toString() {
        return "[StrictMode.ThreadPolicy; mask=" + mask + "]";
    }

    /**
     * Creates {@link ThreadPolicy} instances.  Methods whose names start
     * with {@code detect} specify what problems we should look
     * for.  Methods whose names start with {@code penalty} specify what
     * we should do when we detect a problem.
     *
     * <p>You can call as many {@code detect} and {@code penalty}
     * methods as you like. Currently order is insignificant: all
     * penalties apply to all detected problems.
     *
     * <p>For example, detect everything and log anything that's found:
     * <pre>
     * StrictMode.ThreadPolicy policy = new StrictMode.ThreadPolicy.Builder()
     *     .detectAll()
     *     .penaltyLog()
     *     .build();
     * StrictMode.setThreadPolicy(policy);
     * </pre>
     */
    public static final class Builder {
        private int mMask = 0;

        /**
         * Create a Builder that detects nothing and has no
         * violations.  (but note that {@link #build} will default
         * to enabling {@link #penaltyLog} if no other penalties
         * are specified)
         */
        public Builder() {
            mMask = 0;
        }

        /**
         * Initialize a Builder from an existing ThreadPolicy.
         */
        public Builder(ThreadPolicy policy) {
            mMask = policy.mask;
        }

        /**
         * Detect everything that's potentially suspect.
         *
         * <p>As of the Gingerbread release this includes network and
         * disk operations but will likely expand in future releases.
         */
        public Builder detectAll() {
            return enable(ALL_THREAD_DETECT_BITS);
        }

        /**
         * Disable the detection of everything.
         */
        public Builder permitAll() {
            return disable(ALL_THREAD_DETECT_BITS);
        }

        /**
         * Enable detection of network operations.
         */
        public Builder detectNetwork() {
            return enable(DETECT_NETWORK);
        }

        /**
         * Disable detection of network operations.
         */
        public Builder permitNetwork() {
            return disable(DETECT_NETWORK);
        }

        /**
         * Enable detection of disk reads.
         */
        public Builder detectDiskReads() {
            return enable(DETECT_DISK_READ);
        }

        /**
         * Disable detection of disk reads.
         */
        public Builder permitDiskReads() {
            return disable(DETECT_DISK_READ);
        }

        /**
         * Enable detection of slow calls.
         */
        public Builder detectCustomSlowCalls() {
            return enable(DETECT_CUSTOM);
        }

        /**
         * Disable detection of slow calls.
         */
        public Builder permitCustomSlowCalls() {
            return disable(DETECT_CUSTOM);
        }

        /**
         * Disable detection of mismatches between defined resource types
         * and getter calls.
         */
        public Builder permitResourceMismatches() {
            return disable(DETECT_RESOURCE_MISMATCH);
        }

        /**
         * Enables detection of mismatches between defined resource types
         * and getter calls.
         * <p>
         * This helps detect accidental type mismatches and potentially
         * expensive type conversions when obtaining typed resources.
         * <p>
         * For example, a strict mode violation would be thrown when
         * calling {@link android.content.res.TypedArray#getInt(int, int)}
         * on an index that contains a String-type resource. If the string
         * value can be parsed as an integer, this method call will return
         * a value without crashing; however, the developer should format
         * the resource as an integer to avoid unnecessary type conversion.
         */
        public Builder detectResourceMismatches() {
            return enable(DETECT_RESOURCE_MISMATCH);
        }

        /**
         * Enable detection of disk writes.
         */
        public Builder detectDiskWrites() {
            return enable(DETECT_DISK_WRITE);
        }

        /**
         * Disable detection of disk writes.
         */
        public Builder permitDiskWrites() {
            return disable(DETECT_DISK_WRITE);
        }

        /**
         * Show an annoying dialog to the developer on detected
         * violations, rate-limited to be only a little annoying.
         */
        public Builder penaltyDialog() {
            return enable(PENALTY_DIALOG);
        }

        /**
         * Crash the whole process on violation.  This penalty runs at
         * the end of all enabled penalties so you'll still get
         * see logging or other violations before the process dies.
         *
         * <p>Unlike {@link #penaltyDeathOnNetwork}, this applies
         * to disk reads, disk writes, and network usage if their
         * corresponding detect flags are set.
         */
        public Builder penaltyDeath() {
            return enable(PENALTY_DEATH);
        }

        /**
         * Crash the whole process on any network usage.  Unlike
         * {@link #penaltyDeath}, this penalty runs
         * <em>before</em> anything else.  You must still have
         * called {@link #detectNetwork} to enable this.
         *
         * <p>In the Honeycomb or later SDKs, this is on by default.
         */
        public Builder penaltyDeathOnNetwork() {
            return enable(PENALTY_DEATH_ON_NETWORK);
        }

        /**
         * Flash the screen during a violation.
         */
        public Builder penaltyFlashScreen() {
            return enable(PENALTY_FLASH);
        }

        /**
         * Log detected violations to the system log.
         */
        public Builder penaltyLog() {
            return enable(PENALTY_LOG);
        }

        /**
         * Enable detected violations log a stacktrace and timing data
         * to the {@link android.os.DropBoxManager DropBox} on policy
         * violation.  Intended mostly for platform integrators doing
         * beta user field data collection.
         */
        public Builder penaltyDropBox() {
            return enable(PENALTY_DROPBOX);
        }

        private Builder enable(int bit) {
            mMask |= bit;
            return this;
        }

        private Builder disable(int bit) {

            mMask &= ~bit;
            return this;
        }

        /**
         * Construct the ThreadPolicy instance.
         *
         * <p>Note: if no penalties are enabled before calling
         * <code>build</code>, {@link #penaltyLog} is implicitly
         * set.
         */
        public ThreadPolicy build() {
            // If there are detection bits set but no violation bits
            // set, enable simple logging.
            if (mMask != 0 &&
                (mMask & (PENALTY_DEATH | PENALTY_LOG |
                          PENALTY_DROPBOX | PENALTY_DIALOG)) == 0) {
                penaltyLog();
            }
            return new ThreadPolicy(mMask);
        }
    }
}
```

**-Hans 2016.05.14 15:00**


