---
title: 【BroadcastReceiver】LocalBroadcastManager详解
date: 2017-02-25 17:04
tag: BrocastReceiver
category: Android进阶
---
## 概述

之前一直认为这个东西很神秘，当我深入查看了之后，发现是一个很简单的东西。

﻿理论上，有点类似EventBus低配版的意味。

<!-- more -->

## 成员变量

```

private static final String TAG = "LocalBroadcastManager";
private static final boolean DEBUG = false;

private final Context mAppContext;

private final HashMap<BroadcastReceiver, ArrayList<IntentFilter>> mReceivers
        = new HashMap<BroadcastReceiver, ArrayList<IntentFilter>>();
private final HashMap<String, ArrayList<ReceiverRecord>> mActions
        = new HashMap<String, ArrayList<ReceiverRecord>>();

private final ArrayList<BroadcastRecord> mPendingBroadcasts
        = new ArrayList<BroadcastRecord>();

static final int MSG_EXEC_PENDING_BROADCASTS = 1;

private final Handler mHandler;

private static final Object mLock = new Object();
private static LocalBroadcastManager mInstance;

public static LocalBroadcastManager getInstance(Context context) {
    synchronized (mLock) {
        if (mInstance == null) {
            mInstance = new LocalBroadcastManager(context.getApplicationContext());
        }
        return mInstance;
    }
}

private LocalBroadcastManager(Context context) {
    mAppContext = context;
    mHandler = new Handler(context.getMainLooper()) {

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_EXEC_PENDING_BROADCASTS:
                    executePendingBroadcasts();
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    };
}
```

* 单例模式

* 内置Handler，将广播的执行放到主线程

* 三个重要的成员变量，进行对应的记录存储

    * 第一个`mReceivers` K-V分别是BrocastReceiver和IntentFilter的规则匹配集

    * 第二个`mActions` K-V分别是广播的Action以及对应的所有ReceiverRecord组成的集合。源码可以看出蛮简单的：

```

private static class ReceiverRecord {
    final IntentFilter filter;
    final BroadcastReceiver receiver;
    boolean broadcasting;
}
```

    * 第三个`mPendingBroadcasts`的集合。执行的广播结果集。 这个是用于后面`executePendingBroadcasts`方法时候，用于发送广播的。



## registerReceiver
```
public void registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
    synchronized (mReceivers) {
        ReceiverRecord entry = new ReceiverRecord(filter, receiver);//创建一条记录
        ArrayList<IntentFilter> filters = mReceivers.get(receiver);//从我的Map集合中拿到看看这个广播是否已经注册过
        if (filters == null) {//未注册过
            filters = new ArrayList<IntentFilter>(1);//创建一个规则匹配的集合（因为过滤规则是可以很多的）
            mReceivers.put(receiver, filters);//添加到Map
        }
        filters.add(filter);//添加过滤规则
        for (int i=0; i<filter.countActions(); i++) {//遍历过滤规则中的action
            String action = filter.getAction(i);//获取广播action
            ArrayList<ReceiverRecord> entries = mActions.get(action);//从Map中获取是否已经有这个action的记录
            if (entries == null) {//没有记录
                entries = new ArrayList<ReceiverRecord>(1);//创建记录集合
                mActions.put(action, entries);//将这个action和对应的广播记录集合放入
            }
            entries.add(entry);//添加广播记录
        }
    }
}
```
以上流程
* 创建广播记录
* 将广播实体和过滤规则添加到`mReceivers`的Map集合中去
* 将广播action和广播记录添加到`mActions`的Map集合中去
* 一句话总结：将广播和匹配规则以及action做记录（方便后续分发广播）

## sendBroadcast
```
public boolean sendBroadcast(Intent intent) {
    synchronized (mReceivers) {
        final String action = intent.getAction();//获取具体action
        final String type = intent.resolveTypeIfNeeded(
                mAppContext.getContentResolver());//不明含义~
        final Uri data = intent.getData();
        final String scheme = intent.getScheme();
        final Set<String> categories = intent.getCategories();

        final boolean debug = DEBUG ||
                ((intent.getFlags() & Intent.FLAG_DEBUG_LOG_RESOLUTION) != 0);
        if (debug) Log.v(
                TAG, "Resolving type " + type + " scheme " + scheme
                + " of intent " + intent);

        ArrayList<ReceiverRecord> entries = mActions.get(intent.getAction());//遍历对应的action集合
        if (entries != null) {
            if (debug) Log.v(TAG, "Action list: " + entries);

            ArrayList<ReceiverRecord> receivers = null;
            for (int i=0; i<entries.size(); i++) {
                ReceiverRecord receiver = entries.get(i);
                if (debug) Log.v(TAG, "Matching against filter " + receiver.filter);

                if (receiver.broadcasting) {//处于广播中，那么无视该广播
                    if (debug) {
                        Log.v(TAG, "  Filter's target already added");
                    }
                    continue;
                }

                int match = receiver.filter.match(action, type, scheme, data,
                        categories, "LocalBroadcastManager");//是否匹配该广播
                if (match >= 0) {
                    if (debug) Log.v(TAG, "  Filter matched!  match=0x" +
                            Integer.toHexString(match));
                    if (receivers == null) {
                        receivers = new ArrayList<ReceiverRecord>();
                    }
                    receivers.add(receiver);//添加到发送列表中
                    receiver.broadcasting = true;//表示发送中
                } else {
                    if (debug) {//不匹配
                        String reason;
                        switch (match) {
                            case IntentFilter.NO_MATCH_ACTION: reason = "action"; break;
                            case IntentFilter.NO_MATCH_CATEGORY: reason = "category"; break;
                            case IntentFilter.NO_MATCH_DATA: reason = "data"; break;
                            case IntentFilter.NO_MATCH_TYPE: reason = "type"; break;
                            default: reason = "unknown reason"; break;
                        }
                        Log.v(TAG, "  Filter did not match: " + reason);
                    }
                }
            }

            if (receivers != null) {
                for (int i=0; i<receivers.size(); i++) {
                    receivers.get(i).broadcasting = false;//重新设置为false，这里有点奇怪哦。 刚设置为true然后为false是什么意思
                }
                mPendingBroadcasts.add(new BroadcastRecord(intent, receivers));//添加到发送列表
                if (!mHandler.hasMessages(MSG_EXEC_PENDING_BROADCASTS)) {
                    mHandler.sendEmptyMessage(MSG_EXEC_PENDING_BROADCASTS);//通过Handler执行具体逻辑
                }
                return true;
            }
        }
    }
    return false;
}
```
流程：
* 遍历已有的`mActions`集合，得到匹配action的`ArrayList<ReceiverRecord>`集合
* 遍历该集合，看看是否完美匹配规则
* 匹配则添加到发送集合
* 通过`Handler`发送

注意：
上面有两处代码很迷
```
···
//匹配成功设置为true
receiver.broadcasting = true;//表示发送中
···
//发送前设置为false
receivers.get(i).broadcasting = false;//重新设置为false，这里有点奇怪哦。 刚设置为true然后为false是什么意思
```
本来`sendBroadcast`方法就是同步的，那么为何需要这样做呢？
后面我才发现，是为了避免同一个`Receiver`多次匹配了规则，所以才这样做了一个变量标记。我个人认为`hasMatched`更加贴切...`broadcasting`容易误导

## executePendingBroadcasts

通过`Handler`发送之后，会执行`executePendingBroadcasts`

```

mHandler = new Handler(context.getMainLooper()) {

    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case MSG_EXEC_PENDING_BROADCASTS:
                executePendingBroadcasts();
                break;
            default:
                super.handleMessage(msg);
        }
    }
};

private void executePendingBroadcasts() {
    while (true) {
        BroadcastRecord[] brs = null;
        synchronized (mReceivers) {//加锁
            final int N = mPendingBroadcasts.size();
            if (N <= 0) {
                return;
            }
            brs = new BroadcastRecord[N];//创建一个数组来保存。 这里这样做的原因，是为了尽快响应下一波的广播。因为handler是可以带队列的，所以广播可以一直排好序等待处理。而mPendingBroadcasts只有一个
            mPendingBroadcasts.toArray(brs);
            mPendingBroadcasts.clear();//清除
        }
        for (int i=0; i<brs.length; i++) {
            BroadcastRecord br = brs[i];
            for (int j=0; j<br.receivers.size(); j++) {
                br.receivers.get(j).receiver.onReceive(mAppContext, br.intent);//调用生命周期
            }
        }
    }
}
```

逻辑比较简单，只有注意一下里面用了一个新的数组来保存，为的是快速响应下一次的广播。



## 小结

* 整体思路比较简单，代码不过300行左右。

* 设计思路也蛮简单的，有点EventBus低级版的意味。

* 再也不能黑广播开销大了，只能黑全局广播开销大了。（← . ←）
