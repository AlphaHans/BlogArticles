---
title: 【Binder系列】（二）探究Remote和Local的异同
date: 2016-05-31 17:20
tag: Binder
category: Android进阶
---
<!-- more -->
# Local与Remote的比较
****
远程的Service和本地的Service实现代码有所不同。
* LocalService
```
    @Override
    public IBinder onBind(Intent intent) {
        return new LocalCaculateBinder();
    }
    public static class LocalCaculateBinder extends Binder {
        public void add(int arg0, int arg1) {
            System.out.println("CaculateBinder = "
                    + String.valueOf(arg0 + arg1));
        }
    }  
```
直接创建一个类继承自Binder的类即可
* LocalActivity
```
mLocalCaculateBinder = (LocalCaculateBinder) service;  
```
直接进行强制转换即可
* RemoteService
```
    @Override
    public IBinder onBind(Intent intent) {
        return mRemoteCaculateBinder;
    }
    public Binder mRemoteCaculateBinder = new RemoteCaculateBinder.Stub() {
        @Override
        public void add(int arg0, int arg1) throws RemoteException {
            System.out.println("RemoteCaculateBinder = "
                    + String.valueOf(arg0 + arg1));
        }
    };
```
Binder对象为：创建AIDL生成类中的RemoteCaculate的静态内部类Stub
*RemoteActivity
```
mRemoteCaculateBinder = RemoteCaculateBinder.Stub.asInterface(service);
```
拿到远程端的通讯Binder也需要用到**创建AIDL生成类中的RemoteCaculate的静态内部类Stub**

首先让我感到困惑的有：
* 如果把LocalService的process属性设为:remote会有什么问题
* LocalActivity直接强转就行，而RemoteActivity却也要调用Stub对象的asInterface方法？ 

# LocalService的process属性设为:remote？
****
在不修改任何代码的情况下，或出现如下错误：
```
05-31 16:20:19.884: E/AndroidRuntime(6682): java.lang.ClassCastException: android.os.BinderProxy cannot be cast to com.example.remotebindertest.local.service.LocalService$LocalCaculateBinder
05-31 16:20:19.884: E/AndroidRuntime(6682):     at com.example.remotebindertest.local.activity.LocalTestActivity$2.onServiceConnected(LocalTestActivity.java:59)
```
根据失败的堆栈消息，不难看出失败的原因是：**强制类型转换失败的代码**

从这里的强制类型转换失败我们可以知道了，原来LocalCaculateBinder是由`BinderProxy`向上转型而来

# RemoteActivity要调用Stub对象的asInterface？
****
要弄明白这个问题，我们就需要看一下源代码
```
public interface RemoteCaculateBinder extends android.os.IInterface {
        /** Local-side IPC implementation stub class. */
        public static abstract class Stub extends android.os.Binder implements
                com.example.remotebindertest.aidl.RemoteCaculateBinder {
            private static final java.lang.String DESCRIPTOR = "com.example.remotebindertest.aidl.RemoteCaculateBinder";
           
            /**
             * Cast an IBinder object into an
             * com.example.remotebindertest.aidl.RemoteCaculateBinder interface,
             * generating a proxy if needed.
             */
            public static com.example.remotebindertest.aidl.RemoteCaculateBinder asInterface(
                    android.os.IBinder obj) {
                if ((obj == null)) {
                    return null;
                }
                android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
                if (((iin != null) && (iin instanceof com.example.remotebindertest.aidl.RemoteCaculateBinder))) {
                    return ((com.example.remotebindertest.aidl.RemoteCaculateBinder) iin);
                }
                return new com.example.remotebindertest.aidl.RemoteCaculateBinder.Stub.Proxy(
                        obj);
            }
            @Override
            public android.os.IBinder asBinder() {
                return this;
            }
            //....省略一些代码

            private static class Proxy implements
                    com.example.remotebindertest.aidl.RemoteCaculateBinder {
                private android.os.IBinder mRemote;
                Proxy(android.os.IBinder remote) {
                    mRemote = remote;
                }    
                 //....省略一些代码
        }
    }  
```
过程
* 1.调用Stub的asInterface方法（看一下这个方法的注释：把一个IBinder对象转换为你aidl声明的接口；在必要时**使用代理(Proxy)**）
* 2.调用传入的obj（IBinder对象）的queryLocalInterface
* 3.若不为null，则返回这个IBinder对象
* 4.若为null，则使用Proxy代理

实际上通过queryLocalInterface返回的是null，所以最终的还是要通过来使用Proxy代理

这里我又产生了疑问：
* 1.queryLocalInterface是什么？ 从方法名字来看是查询**本地**接口。那这个本地是指**同一个进程**内的意思吗？
* 2.Proxy有什么作用？

**-Hans 2016.5.31 17:20**