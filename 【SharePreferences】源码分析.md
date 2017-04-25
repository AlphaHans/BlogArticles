---
title: 【SharePreferences】源码分析
date: 2016-10-27 17:00
category: Android进阶
---

<!-- more -->
## getString

从getString开始跟踪一下，看看如何获取一个字符串

```

@Nullable
public String getString(String key, @Nullable String defValue) {
    synchronized (this) {
        awaitLoadedLocked();
        String v = (String)mMap.get(key);
        return v != null ? v : defValue;
    }
}
```

可以发现是从一个`Map`集合中获取的。



那么这个Map，是在什么时候载入数据的呢。

```

SharedPreferencesImpl(File file, int mode) {
    mFile = file;
    mBackupFile = makeBackupFile(file);
    mMode = mode;
    mLoaded = false;
    mMap = null;
    startLoadFromDisk();
}

private void startLoadFromDisk() {
    synchronized (this) {
        mLoaded = false;
    }
    new Thread("SharedPreferencesImpl-load") {
        public void run() {
            loadFromDisk();
        }
    }.start();
}

private void loadFromDisk() {
    //...省略一些校验

    Map map = null;
    StructStat stat = null;
    try {
        stat = Os.stat(mFile.getPath());
        if (mFile.canRead()) {
            BufferedInputStream str = null;
            try {
                str = new BufferedInputStream(
                        new FileInputStream(mFile), 16*1024);
                map = XmlUtils.readMapXml(str);
            } catch (XmlPullParserException | IOException e) {
                Log.w(TAG, "getSharedPreferences", e);
            } finally {
                IoUtils.closeQuietly(str);
            }
        }
    } catch (ErrnoException e) {
        /* ignore */
    }

    synchronized (SharedPreferencesImpl.this) {
        mLoaded = true;
        if (map != null) {
            mMap = map;
            mStatTimestamp = stat.st_mtime;
            mStatSize = stat.st_size;
        } else {
            mMap = new HashMap<>();
        }
        notifyAll();
    }
}
```

可以发现当`SharePreferencesImpl`创建的事后会调用`startLoadFromDisk`方法，然后开启了一个子线程调用`loadFromDisk`方法。 

经过一些校验之后看到通过`BufferedInputStream`获取文件内容，然后通过了`XmlUtils.readMapXml`方法来将`map`解析出来。最终将局部变量`map`赋值给`mMap`。



## putString

`putXxx`都是通过`editor`对象进行操作，最终调用`commit`或`apply`

```



public final class EditorImpl implements Editor {
    private final Map<String, Object> mModified = Maps.newHashMap();
    private boolean mClear = false;

    public Editor putString(String key, @Nullable String value) {
        synchronized (this) {
            mModified.put(key, value);
            return this;
        }
    }


public void apply() {
    final MemoryCommitResult mcr = commitToMemory();
    final Runnable awaitCommit = new Runnable() {
            public void run() {
                try {
                    mcr.writtenToDiskLatch.await();
                } catch (InterruptedException ignored) {
                }
            }
        };

    QueuedWork.add(awaitCommit);

    Runnable postWriteRunnable = new Runnable() {
            public void run() {
                awaitCommit.run();
                QueuedWork.remove(awaitCommit);
            }
        };

    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

    // Okay to notify the listeners before it's hit disk
    // because the listeners should always get the same
    // SharedPreferences instance back, which has the
    // changes reflected in memory.
    notifyListeners(mcr);
}

// Returns true if any changes were made
private MemoryCommitResult commitToMemory() {
    MemoryCommitResult mcr = new MemoryCommitResult();
    synchronized (SharedPreferencesImpl.this) {
        // We optimistically don't make a deep copy until
        // a memory commit comes in when we're already
        // writing to disk.
        if (mDiskWritesInFlight > 0) {
            // We can't modify our mMap as a currently
            // in-flight write owns it.  Clone it before
            // modifying it.
            // noinspection unchecked
            mMap = new HashMap<String, Object>(mMap);
        }
        mcr.mapToWriteToDisk = mMap;
        mDiskWritesInFlight++;

        boolean hasListeners = mListeners.size() > 0;
        if (hasListeners) {
            mcr.keysModified = new ArrayList<String>();
            mcr.listeners =
                    new HashSet<OnSharedPreferenceChangeListener>(mListeners.keySet());
        }

        synchronized (this) {
            if (mClear) {
                if (!mMap.isEmpty()) {
                    mcr.changesMade = true;
                    mMap.clear();
                }
                mClear = false;
            }

            for (Map.Entry<String, Object> e : mModified.entrySet()) {
                String k = e.getKey();
                Object v = e.getValue();
                // "this" is the magic value for a removal mutation. In addition,
                // setting a value to "null" for a given key is specified to be
                // equivalent to calling remove on that key.
                if (v == this || v == null) {
                    if (!mMap.containsKey(k)) {
                        continue;
                    }
                    mMap.remove(k);
                } else {
                    if (mMap.containsKey(k)) {
                        Object existingValue = mMap.get(k);
                        if (existingValue != null && existingValue.equals(v)) {
                            continue;
                        }
                    }
                    mMap.put(k, v);
                }

                mcr.changesMade = true;
                if (hasListeners) {
                    mcr.keysModified.add(k);
                }
            }

            mModified.clear();
        }
    }
    return mcr;
}
public boolean commit() {
    MemoryCommitResult mcr = commitToMemory();
    SharedPreferencesImpl.this.enqueueDiskWrite(
        mcr, null /* sync write on this thread okay */);
    try {
        mcr.writtenToDiskLatch.await();
    } catch (InterruptedException e) {
        return false;
    }
    notifyListeners(mcr);
    return mcr.writeToDiskResult;
}

private void notifyListeners(final MemoryCommitResult mcr) {
    if (mcr.listeners == null || mcr.keysModified == null ||
        mcr.keysModified.size() == 0) {
        return;
    }
    if (Looper.myLooper() == Looper.getMainLooper()) {
        for (int i = mcr.keysModified.size() - 1; i >= 0; i--) {
            final String key = mcr.keysModified.get(i);
            for (OnSharedPreferenceChangeListener listener : mcr.listeners) {
                if (listener != null) {
                    listener.onSharedPreferenceChanged(SharedPreferencesImpl.this, key);
                }
            }
        }
    } else {
        // Run this function on the main thread.
        ActivityThread.sMainThreadHandler.post(new Runnable() {
                public void run() {
                    notifyListeners(mcr);
                }
            });
    }
}
```

* 成员变量中有一个名字为`mModified`的`Map`集合，可以表明它是可以修改的。

* 这个`mModified`主要是保存我们当前的操作，然后在commit的时候再一次性写入到磁盘中

* 从commit来分析，分为两大步骤（apply实际就是异步而已，没啥不同）

    * `commitToMemory` 表明提交到内存进去

    * `enqueueDiskWirte` 表明进去写硬盘的"队列"中去

* 注意两个小细节 

    * 一个是`enqueueDiskWrite`第二个参数传的是`null`，并且注释为`sync write on this thread okay` 后面会看到，这个方法是通过第二个参数来区别`commit`和`apply`的    

    * 在`notifyListener`之前，使用了`CountDownLatch`（数量为1），来进行同步操作。 这个`Listener`是用来监听`SharePreferencs`数据改变的时候会回调的。   



### commitToMemory

```

/ Returns true if any changes were made
private MemoryCommitResult commitToMemory() {
    MemoryCommitResult mcr = new MemoryCommitResult();
    synchronized (SharedPreferencesImpl.this) {
        // We optimistically don't make a deep copy until
        // a memory commit comes in when we're already
        // writing to disk.
        
        // 当mDiskWritesInFilght大于0的时候，说明当前正在有写入操作
           。因此不采用深拷贝
        if (mDiskWritesInFlight > 0) {
            // We can't modify our mMap as a currently
            // in-flight write owns it.  Clone it before
            // modifying it.
            // noinspection unchecked
            //当有在写入的操作的事后，我们先会克隆一份，然后才会进行操作
            mMap = new HashMap<String, Object>(mMap);
        }
        mcr.mapToWriteToDisk = mMap;
        mDiskWritesInFlight++;

        boolean hasListeners = mListeners.size() > 0;
        if (hasListeners) {
            mcr.keysModified = new ArrayList<String>();
            mcr.listeners =
                    new HashSet<OnSharedPreferenceChangeListener>(mListeners.keySet());
        }

        synchronized (this) {
            if (mClear) {
                if (!mMap.isEmpty()) {
                    mcr.changesMade = true;
                    mMap.clear();
                }
                mClear = false;
            }

            for (Map.Entry<String, Object> e : mModified.entrySet()) {
                String k = e.getKey();
                Object v = e.getValue();
                // "this" is the magic value for a removal mutation. In addition,
                // setting a value to "null" for a given key is specified to be
                // equivalent to calling remove on that key.
                if (v == this || v == null) {
                    if (!mMap.containsKey(k)) {
                        continue;
                    }
                    mMap.remove(k);
                } else {
                    if (mMap.containsKey(k)) {
                        Object existingValue = mMap.get(k);
                        if (existingValue != null && existingValue.equals(v)) {
                            continue;
                        }
                    }
                    mMap.put(k, v);
                }

                mcr.changesMade = true;
                if (hasListeners) {
                    mcr.keysModified.add(k);
                }
            }

            mModified.clear();
        }
    }
    return mcr;
}
```

主要注意一下几点：

* `mDiskWritesInFlight`是用于判断当前有多少个需要写入硬盘的“任务数”。

* 如果是有在写入，那么则对原本的数据`Map`进行拷贝一份，再进行操作。

* 后面的代码都是进行`Map`的重复键移除或者插入等等。

* 注意此时数据已经成功插入到内存中的`Map`去了，并且保存到了`MemoryCommitResult`的`mapToWirtesInDisk`变量中去，等待写入到硬盘中。



### enqueueDiskWrite

```

private void enqueueDiskWrite(final MemoryCommitResult mcr,
                              final Runnable postWriteRunnable) {
    final Runnable writeToDiskRunnable = new Runnable() {
            public void run() {
                synchronized (mWritingToDiskLock) {
                    writeToFile(mcr);
                }
                synchronized (SharedPreferencesImpl.this) {
                    mDiskWritesInFlight--;
                }
                if (postWriteRunnable != null) {
                    postWriteRunnable.run();
                }
            }
        };

    final boolean isFromSyncCommit = (postWriteRunnable == null);

    // Typical #commit() path with fewer allocations, doing a write on
    // the current thread.
    if (isFromSyncCommit) {
        boolean wasEmpty = false;
        synchronized (SharedPreferencesImpl.this) {
            wasEmpty = mDiskWritesInFlight == 1;
        }
        if (wasEmpty) {
            writeToDiskRunnable.run();
            return;
        }
    }

    QueuedWork.singleThreadExecutor().execute(writeToDiskRunnable);
}
```

* `isFromSyncCommit`用于判定是否直接在当前线程执行，否则就是用`QueueWork.singleThreadExecutor`的线程池异步执行了。

* `writeFile`方法将数据写入



### writeFile

```

// Note: must hold mWritingToDiskLock
private void writeToFile(MemoryCommitResult mcr) {
    // Rename the current file so it may be used as a backup during the next read
    if (mFile.exists()) {
        if (!mcr.changesMade) {
            // If the file already exists, but no changes were
            // made to the underlying map, it's wasteful to
            // re-write the file.  Return as if we wrote it
            // out.
            mcr.setDiskWriteResult(true);
            return;
        }
        if (!mBackupFile.exists()) {
            if (!mFile.renameTo(mBackupFile)) {
                Log.e(TAG, "Couldn't rename file " + mFile
                      + " to backup file " + mBackupFile);
                mcr.setDiskWriteResult(false);
                return;
            }
        } else {
            mFile.delete();
        }
    }

    // Attempt to write the file, delete the backup and return true as atomically as
    // possible.  If any exception occurs, delete the new file; next time we will restore
    // from the backup.
    try {
        FileOutputStream str = createFileOutputStream(mFile);
        if (str == null) {
            mcr.setDiskWriteResult(false);
            return;
        }
        XmlUtils.writeMapXml(mcr.mapToWriteToDisk, str);
        FileUtils.sync(str);
        str.close();
        ContextImpl.setFilePermissionsFromMode(mFile.getPath(), mMode, 0);
        try {
            final StructStat stat = Os.stat(mFile.getPath());
            synchronized (this) {
                mStatTimestamp = stat.st_mtime;
                mStatSize = stat.st_size;
            }
        } catch (ErrnoException e) {
            // Do nothing
        }
        // Writing was successful, delete the backup file if there is one.
        mBackupFile.delete();
        mcr.setDiskWriteResult(true);
        return;
    } catch (XmlPullParserException e) {
        Log.w(TAG, "writeToFile: Got exception:", e);
    } catch (IOException e) {
        Log.w(TAG, "writeToFile: Got exception:", e);
    }
    // Clean up an unsuccessfully written file
    if (mFile.exists()) {
        if (!mFile.delete()) {
            Log.e(TAG, "Couldn't clean up partially-written file " + mFile);
        }
    }
    mcr.setDiskWriteResult(false);
}
```

* 这里也不是特别难，也就是一些`IO`的读写方法，理解起来没有什么难度

* 不过需要注意的是`mcr.setDiskWriteResult`这个方法：

```

public void setDiskWriteResult(boolean result) {
    writeToDiskResult = result;
    writtenToDiskLatch.countDown();
}
```

执行了`writtenToDiskLatch.countDown`，这样一来在`commit`方法中的同步锁释放了，可以进行`notifyListener`通知数据改变了


**-Hans 2016.10.27**