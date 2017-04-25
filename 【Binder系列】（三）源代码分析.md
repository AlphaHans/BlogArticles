---
title: 【Binder系列】（三）源代码分析
date: 2016-06-03 13:20
tag: Binder
category: Android进阶
---
<!-- more -->
# IBinder
****
## 1.IBinder#queryLocalInterface
看一下IBinder中queryLocalInterface的注释：
```
 /**
 * Attempt to retrieve a local implementation of an interface
 * for this Binder object.  If null is returned, you will need
 * to instantiate a proxy class to marshall calls through
 * the transact() method.
 */
public IInterface queryLocalInterface(String descriptor);
```
意思是在**本地**去**查找**实现了这个接口的Binder类。 如果是null,则需要去实例化一个代理（proxy），并通过transact方法去**引领（安排；统领 marshall）**这些**调用（calls）**

**这里的话，我们先将这个queryLocalInterface是从哪里查询的问题先放一放。**

## 2.IBinder#transact
继续看看IBinder下的transact方法
```
/**
 * Perform a generic operation with the object.
 * 
 * @param code The action to perform.  This should
 * be a number between {@link #FIRST_CALL_TRANSACTION} and
 * {@link #LAST_CALL_TRANSACTION}.
 * @param data Marshalled data to send to the target.  Must not be null.
 * If you are not sending any data, you must create an empty Parcel
 * that is given here.
 * @param reply Marshalled data to be received from the target.  May be
 * null if you are not interested in the return value.
 * @param flags Additional operation flags.  Either 0 for a normal
 * RPC, or {@link #FLAG_ONEWAY} for a one-way RPC.
 */
public boolean transact(int code, Parcel data, Parcel reply, int flags)
    throws RemoteException;
```
~~注释的意思是 通过对象来执行一个类的(generic)操作。
看注释好像不太懂，不过通过看方法好像是用来根据**code**来通过**data**传输数据，从而来调用远程方法的意思？~~
注释的意思是：执行对象的某一个方法：从**code**来判断执行对象的哪一个方法，**data**传递参数。
因为远程调用就是通过**code**来具体调用对应的方法。
AIDL帮我们生成的类中，有一个叫onTransact的方法。他们之间是不是有联系呢？不过我先看一下Binder类里面是如何实现了这个方法的。

# Binder
****
## 1.Binder#transact
```
/**
 * Default implementation rewinds the parcels and calls onTransact.  On
 * the remote side, transact calls into the binder to do the IPC.
 */
public final boolean transact(int code, Parcel data, Parcel reply,
        int flags) throws RemoteException {
    if (false) Log.v("Binder", "Transact: " + code + " to " + this);
    if (data != null) {
        data.setDataPosition(0);
    }
    boolean r = onTransact(code, data, reply, flags);
    if (reply != null) {
        reply.setDataPosition(0);
    }
    return r;
}
```
* 1.注意到transact方法为final，也就是不能被重写了
* 2.data.setDataPostion(0) 从源代码可以看到：Move the current read/write position in the parcel.也就是表明现在可以开始读取数据了。
* 3.调用了onTransact方法
* 4.reply.setDataPosition(0) 和2同理，表明可以开始写入数据了。

## 2.Binder#onTransact
```
/**
 * Default implementation is a stub that returns false.  You will want
 * to override this to do the appropriate unmarshalling of transactions.
 *
 * <p>If you want to call this, call transact().
 */
protected boolean onTransact(int code, Parcel data, Parcel reply,
        int flags) throws RemoteException {
    if (code == INTERFACE_TRANSACTION) {
        reply.writeString(getInterfaceDescriptor());
        return true;
    } else if (code == DUMP_TRANSACTION) {
        ParcelFileDescriptor fd = data.readFileDescriptor();
        String[] args = data.readStringArray();
        if (fd != null) {
            try {
                dump(fd.getFileDescriptor(), args);
            } finally {
                try {
                    fd.close();
                } catch (IOException e) {
                    // swallowed, not propagated back to the caller
                }
            }
        }
        // Write the StrictMode header.
        if (reply != null) {
            reply.writeNoException();
        } else {
            StrictMode.clearGatheredViolations();
        }
        return true;
    }
    return false;
}
```
注释：默认实现是返回一个false的存根（stub）。 你将覆盖这个方法，并做一个合适的数据编出（解组 unmarshalling）处理

感觉这个方法在aidl帮我们自动生成的类中也有。

# RemoteCaculateBinder
****
现在我们看看这里面的代码：
## 1.RemoteCaculateBinder$Stub
Stub继承了Binder类，并重写了这个方法的:
```
 public static abstract class Stub extends android.os.Binder implements
                com.example.remotebindertest.aidl.RemoteCaculateBinder {
            
            //....省略

            @Override
            public boolean onTransact(int code, android.os.Parcel data,
                    android.os.Parcel reply, int flags)
                    throws android.os.RemoteException {
                switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_add: {
                    data.enforceInterface(DESCRIPTOR);
                    int _arg0;
                    _arg0 = data.readInt();
                    int _arg1;
                    _arg1 = data.readInt();
                    this.add(_arg0, _arg1);
                    reply.writeNoException();
                    return true;
                }
                }
                return super.onTransact(code, data, reply, flags);
            }

            //....省略Proxy类

            static final int TRANSACTION_add = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        }
```
突然发现了一些不得了的东西，就是这个onTransact中有一个处理TRANSACTION_add；然后再看看里面的`_arg0 = data.readInt()`和`_arg1 = data.readInt()`这些都好像显示出，是对应了我们在.aidl文件中生命的add方法。 那到底是不是呢？实际上是的。

然后再发现了它里面的操作都是**readXxx**操作呢，再结合RemoteService中的:
```
public Binder mRemoteCaculateBinder = new RemoteCaculateBinder.Stub() {
        @Override
        public void add(int arg0, int arg1) throws RemoteException {
            System.out.println("RemoteCaculateBinder = "
                    + String.valueOf(arg0 + arg1));
        }
    };
```
那么就可以假设：**onTransact**将主进程（Local端）的数据**读取**出来，然后**传递**给子进程（Remote端）的**add**方法:
* data.enforceInterface(DESCRIPTOR);
* _arg0 = data.readInt();
*  _arg1 = data.readInt();
* this.add(_arg0, _arg1);

**的确这个假设是成立的。**

那**有读必有写**，那是在哪里写入的呢？

## 2.RemoteCaculateBinder Stub$Proxy
aidl文件中就剩下Proxy没有看到了，那么这个类可能是**写入**的地方：
```
 private static class Proxy implements
                    com.example.remotebindertest.aidl.RemoteCaculateBinder {
                private android.os.IBinder mRemote;
                Proxy(android.os.IBinder remote) {
                    mRemote = remote;
                }
                @Override
                public android.os.IBinder asBinder() {
                    return mRemote;
                }
                public java.lang.String getInterfaceDescriptor() {
                    return DESCRIPTOR;
                }
                @Override
                public void add(int arg0, int arg1)
                        throws android.os.RemoteException {
                    android.os.Parcel _data = android.os.Parcel.obtain();
                    android.os.Parcel _reply = android.os.Parcel.obtain();
                    try {
                        _data.writeInterfaceToken(DESCRIPTOR);
                        _data.writeInt(arg0);
                        _data.writeInt(arg1);
                        mRemote.transact(Stub.TRANSACTION_add, _data, _reply, 0);
                        _reply.readException();
                    } finally {
                        _reply.recycle();
                        _data.recycle();
                    }
                }
            }
```
从add方法中的：
* _data.writeInterfaceToken(DESCRIPTOR);
* _data.writeInt(arg0);
* _data.writeInt(arg1);
* mRemote.transact(Stub.TRANSACTION_add, _data, _reply, 0);
可以看出，毋庸置疑，Proxy类就是用来将数据写入的地方。

再结合RemoteActivity中：
```
mRemoteCaculateBinder = RemoteCaculateBinder.Stub.asInterface(service);
```
原来这个方法返回的实例就是Proxy代理对象，然后我们调用的add方法：
```
mAdd.setOnClickListener(new OnClickListener() {
            @Override
            public void onClick(View v) {
                try {
                    mRemoteCaculateBinder.add(
                            Integer.valueOf(mAddOne.getText().toString()),
                            Integer.valueOf(mAddTwo.getText().toString()));
                } catch (NumberFormatException e) {
                    e.printStackTrace();
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            }
        });
```
就是调用了Proxy里面这个add方法，然后将传入的两个int，通过Pracel写入。

至此整个流程好像就浮出水面了。

此外，我们还可以猜测实际上asInterface中的queryInterface返回的就是null，然后创建了Proxy对象实例。所以我猜测，这个方法就是在**主进程中查询**有没有实现了这个接口的对应的Binder对象。
**就通过下面这个例子证明**

# 去掉RemoteService的AndroidManifest的process
****
我在学习的过程中，突然想把RemoteService的process=":remote"的属性去掉，看看还能不能正常工作。
其他代码一句都不修改。包括RemoteActivity的：
```
mRemoteCaculateBinder = RemoteCaculateBinder.Stub.asInterface(service);
```
也就是说现在RemoteService实际上是一个local的service了，即和LocalService一样了。

答案如我所料：**可以正常工作**

答案就在这个方法：
```
 public static com.example.remotebindertest.aidl.RemoteCaculateBinder asInterface(android.os.IBinder obj) {
     //.....
     android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
     if (((iin != null) && (iin instanceof com.example.remotebindertest.aidl.RemoteCaculateBinder))) {
         return ((com.example.remotebindertest.aidl.RemoteCaculateBinder) iin);
     }
     return new com.example.remotebindertest.aidl.RemoteCaculateBinder.Stub.Proxy(obj);
}
```
所以我们可以知道了现在他就是：
```
android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
if (((iin != null) && (iin instanceof com.example.remotebindertest.aidl.RemoteCaculateBinder))) {
    return ((com.example.remotebindertest.aidl.RemoteCaculateBinder) iin);
 }
```
直接强转返回。
那么也就是刚好证明了：**queryLocalInterface**是在**主进程**查找**对应Binder实例**的猜测。

# 小结
****
知识点总结：
* Poxy、Stub是一对操作 Stub负责在**远端**读取Poxy在**本地**写入的数据，然后传给响应的方法。
* queryLocalInterface方法是用来查询**主进程**中查找有没有实现这个接口对应的Binder对象。
* Stub的onTransact方法就是远端用来获取数据的方法，并调用对应的方法。

流程总结：
* 1.Service中返回实现**IBinder接口**的对象
* 2.返回的实例（根据AIDL生成类的可以判断）
 * 1.Local（本地） 强制转换并直接返回
 * 2.Remote（远端） 通过Poxy代理
* 3.通讯
 * 1.Local Activity直接强制类型转换，并直接调用方法。
 * 2.Remote Activity需要调用**RemoteCaculateBinder.Stub.asInterface(service)** 通过Poxy代理

 **-Hans 2016.6.3 13:20**