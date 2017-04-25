---
title: 【Binder系列】（一）工程介绍
date: 2016-05-31 16:05
tag: Binder
category: Android进阶
---
<!-- more -->
> 使用Eclipse进行示范，Github地址：https://github.com/AlphaHans/Local-RemoteServiceTest

# 本地Service的通讯
****
本地Service与本地Activity的通讯
## 1.项目结构


## 2.LocalActivity代码
```
public class LocalTestActivity extends Activity {
    private EditText mAddOne, mAddTwo;
    private Button mAdd;
    private ServiceConnection mServiceConnection;
    private LocalService.LocalCaculateBinder mLocalCaculateBinder;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        initService();
        initView();
    }
    private void initView() {
        mAddOne = (EditText) findViewById(R.id.main_et_one);
        mAddTwo = (EditText) findViewById(R.id.main_et_two);
        mAdd = (Button) findViewById(R.id.main_btn_add);
        mAdd.setOnClickListener(new OnClickListener() {
            @Override
            public void onClick(View v) {
                mLocalCaculateBinder.add(
                        Integer.valueOf(mAddOne.getText().toString()),
                        Integer.valueOf(mAddTwo.getText().toString()));
            }
        });
    }
    private void initService() {
        mServiceConnection = new ServiceConnection() {
            @Override
            public void onServiceDisconnected(ComponentName name) {
            }
            @Override
            public void onServiceConnected(ComponentName name, IBinder service) {
                mLocalCaculateBinder = (LocalCaculateBinder) service;
            }
        };
        Intent i = new Intent(this, LocalService.class);
        bindService(i, mServiceConnection, BIND_AUTO_CREATE);
    }
}  
```

## 3.LocalService代码
```
public class LocalService extends Service {
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
}  
```

# 远程Service的通讯
****
远程Service与本地Activity通讯

## 1.项目结构





## 2.RemoteActivity代码
```
public class RemoteTestActivity extends Activity {
    private EditText mAddOne, mAddTwo;
    private Button mAdd;
    private ServiceConnection mServiceConnection;
    private RemoteCaculateBinder mRemoteCaculateBinder;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        initService();
        mAddOne = (EditText) findViewById(R.id.main_et_one);
        mAddTwo = (EditText) findViewById(R.id.main_et_two);
        mAdd = (Button) findViewById(R.id.main_btn_add);
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
    }
    private void initService() {
        mServiceConnection = new ServiceConnection() {
            @Override
            public void onServiceDisconnected(ComponentName name) {
            }
            @Override
            public void onServiceConnected(ComponentName name, IBinder service) {
                mRemoteCaculateBinder = RemoteCaculateBinder.Stub.asInterface(service);
            }
        };
        Intent i = new Intent(this, RemoteService.class);
        bindService(i, mServiceConnection, BIND_AUTO_CREATE);
    }
}  
```

## 3.RemoteService代码
```
public class RemoteService extends Service {
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
}  
```

## 4.AIDL代码
### 1.RemoteCaculateBinder.aidl代码
```
package com.example.remotebindertest.aidl;
interface RemoteCaculateBinder {
    void add(int arg0, int arg1);
}
```

### 2.Gen生成的AIDL代码
```
public interface RemoteCaculateBinder extends android.os.IInterface {
        /** Local-side IPC implementation stub class. */
        public static abstract class Stub extends android.os.Binder implements
                com.example.remotebindertest.aidl.RemoteCaculateBinder {
            private static final java.lang.String DESCRIPTOR = "com.example.remotebindertest.aidl.RemoteCaculateBinder";
            /** Construct the stub at attach it to the interface. */
            public Stub() {
                this.attachInterface(this, DESCRIPTOR);
            }
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
            static final int TRANSACTION_add = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        }
        public void add(int arg0, int arg1) throws android.os.RemoteException;
    }  
```

**-Hans 2016.5.31 16:06**