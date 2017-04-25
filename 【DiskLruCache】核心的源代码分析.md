---
title: 【DiskLruCache】核心的源代码分析
tag: LruCache 
category: Android进阶
date: 2016-05-28 16:12
---
> 本文只包含DiskLruCache日志系统的创建以及缓存文件删除的源码分析。

# 数据结构LinkedHashMap
****
LinkedHashMap是DiskLruCache中**最重要**的一个环节之一。
<!-- more -->
为什么呢？

**LRU（Least Recently Used）**算法是最少使用算法。这个算法也就是说需要按照**使用的频率顺序**要进行**排序**的。

那么LinkedHashMap正好可以**满足这个条件**
```
private final LinkedHashMap<String, Entry> lruEntries =
    new LinkedHashMap<String, Entry>(0, 0.75f, true);
```
第三个参数指定了一个排序模式（mode）。true为**访问顺序排序**（access-order）false为**插入顺序排序**（insertion-order）


> 构造函数中，源代码对于该参数的注释：  @param accessOrder
 {@code true} if the ordering should be done based on the last access (from least-recently accessed to most-recently accessed), and {@code false} if the ordering should be the order in which the entries were inserted.

举一个例子：
```
private static Map<String, String> mLinkedHashMap = new LinkedHashMap<>
        (0, 0.7f, true);

public static void main(String[] args) {
    mLinkedHashMap.put("one", "one");
    mLinkedHashMap.put("two", "two");
    mLinkedHashMap.put("three", "three");
    System.out.println(mLinkedHashMap);
    String value = mLinkedHashMap.get("one");//访问one
    System.out.println(mLinkedHashMap);
}
```
输出结果：
```
{one=one, two=two, three=three}
{two=two, three=three, one=one}
```
可以发现LinkedHashMap根据**访问频率**进行了**重排序**。

但如果我们将true改为false，顺序就不会变了。

所以，当DiskLruCache的缓存大小**大于**我们**指定的大小**时，就可以根据LinkedHashMap**从头**开始**删除缓存文件**，直到大小**小于**我们指定的缓存大小
# DiskLruCache创建
****
```
private DiskLruCache(File directory, int appVersion, int valueCount, long maxSize) {
  this.directory = directory;
  this.appVersion = appVersion;
  this.journalFile = new File(directory, JOURNAL_FILE);
  this.journalFileTmp = new File(directory, JOURNAL_FILE_TEMP);
  this.journalFileBackup = new File(directory, JOURNAL_FILE_BACKUP);
  this.valueCount = valueCount;
  this.maxSize = maxSize;
}

/**
 * Opens the cache in {@code directory}, creating a cache if none exists
 * there.
 *
 * @param directory a writable directory
 * @param valueCount the number of values per cache entry. Must be positive.
 * @param maxSize the maximum number of bytes this cache should use to store
 * @throws IOException if reading or writing the cache directory fails
 */
public static DiskLruCache open(File directory, int appVersion, int valueCount, long maxSize)
    throws IOException {
  if (maxSize <= 0) {
    throw new IllegalArgumentException("maxSize <= 0");
  }
  if (valueCount <= 0) {
    throw new IllegalArgumentException("valueCount <= 0");
  }
 
  // If a bkp file exists, use it instead.
  File backupFile = new File(directory, JOURNAL_FILE_BACKUP);
  if (backupFile.exists()) {
    File journalFile = new File(directory, JOURNAL_FILE);
    // If journal file also exists just delete backup file.
    if (journalFile.exists()) {
      backupFile.delete();
    } else {
      renameTo(backupFile, journalFile, false);
    }
  }

  // Prefer to pick up where we left off.
  DiskLruCache cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
  if (cache.journalFile.exists()) {
    try {
      cache.readJournal();
      cache.processJournal();
      cache.journalWriter = new BufferedWriter(
          new OutputStreamWriter(new FileOutputStream(cache.journalFile, true), Util.US_ASCII));
      return cache;
    } catch (IOException journalIsCorrupt) {
      System.out
          .println("DiskLruCache "
              + directory
              + " is corrupt: "
              + journalIsCorrupt.getMessage()
              + ", removing");
      cache.delete();
    }
  }

  // Create a new empty cache.
  directory.mkdirs();
  cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
  cache.rebuildJournal();
  return cache;
}
```
调用open方法：
* 1.创建日志文件
 * 1.若存在日志备份文件bkp,且日志文件不存在，则将备份文件文件改名为日志文件名字
 * 2.若存在日志备份文件bkp,且日志文件存在，则直接删除备份文件
* 2 初始化日志内容
 * 1.若日志文件存在，调用readJournal 读取日志文件中的内容 并放入LinkedHashMap中；processJournal 根据LinekedHashMap中的内容：统计大小、删除无用的文件等等操作
 * 2.若日志文件不存在，调用rebuildJournal创建日志文件

## 1. rebuildJournal第一次创建日志文件
```
/**
 * Creates a new journal that omits redundant information. This replaces the
 * current journal if it exists.
 */
private synchronized void rebuildJournal() throws IOException {
  if (journalWriter != null) {
    journalWriter.close();
  }

  Writer writer = new BufferedWriter(
      new OutputStreamWriter(new FileOutputStream(journalFileTmp), Util.US_ASCII));
  try {
    writer.write(MAGIC);
    writer.write("\n");
    writer.write(VERSION_1);
    writer.write("\n");
    writer.write(Integer.toString(appVersion));
    writer.write("\n");
    writer.write(Integer.toString(valueCount));
    writer.write("\n");
    writer.write("\n");

    for (Entry entry : lruEntries.values()) {
      if (entry.currentEditor != null) {
        writer.write(DIRTY + ' ' + entry.key + '\n');
      } else {
        writer.write(CLEAN + ' ' + entry.key + entry.getLengths() + '\n');
      }
    }
  } finally {
    writer.close();
  }

  if (journalFile.exists()) {
    renameTo(journalFile, journalFileBackup, true);
  }
  renameTo(journalFileTmp, journalFile, false);
  journalFileBackup.delete();

  journalWriter = new BufferedWriter(
      new OutputStreamWriter(new FileOutputStream(journalFile, true), Util.US_ASCII));
}
```
流程：
* 1.关闭**journal**文件的输出流（**若开启了**）
* 2.开启**journal.tmp**文件的输出流
* 3.写入头部信息 libcore.io.DiskLruCache、DiskLruCache版本号、App版本号、**valueCount（一个key可以对应多少个值）**
* 4.写入已经LinkedHashMap中的数据：包括DIRRY（**未commit**）文件、CLEAN（**已commit**）文件的列表
* 5.关闭**journal.tmp**文件的输出流
* 6.若**journal.bkp**文件存在，将其删除。并将**journal文件改名为journal.bkp**
* 7.将**journal.tmp改名为journal**
* 8.将**journal.bkp文件删除。**
* 9.**重新开启**journal文件的输出流

**678这三个过程看起来是重复操作了**，但是这对于IO文件操作却很有必要。**防止数据量巨大的时候（DiskLruCache限制条目数是2000），导致数据拷贝过程中遇到问题，而导致文件损坏，从而对App造成影响。**

## 2.readJournal读取日志内容
readJournal读取日志内容并放入LinkedHashMap容器中
```
private void readJournal() throws IOException {
  StrictLineReader reader = new StrictLineReader(new FileInputStream(journalFile), Util.US_ASCII);
  try {
    String magic = reader.readLine();
    String version = reader.readLine();
    String appVersionString = reader.readLine();
    String valueCountString = reader.readLine();
    String blank = reader.readLine();
    if (!MAGIC.equals(magic)
        || !VERSION_1.equals(version)
        || !Integer.toString(appVersion).equals(appVersionString)
        || !Integer.toString(valueCount).equals(valueCountString)
        || !"".equals(blank)) {
      throw new IOException("unexpected journal header: [" + magic + ", " + version + ", "
          + valueCountString + ", " + blank + "]");
    }

    int lineCount = 0;
    while (true) {
      try {
        readJournalLine(reader.readLine());
        lineCount++;
      } catch (EOFException endOfJournal) {
        break;
      }
    }
    redundantOpCount = lineCount - lruEntries.size();
  } finally {
    Util.closeQuietly(reader);
  }
}

private void readJournalLine(String line) throws IOException {
  int firstSpace = line.indexOf(' ');
  if (firstSpace == -1) {
    throw new IOException("unexpected journal line: " + line);
  }

  int keyBegin = firstSpace + 1;
  int secondSpace = line.indexOf(' ', keyBegin);
  final String key;
  if (secondSpace == -1) {
    key = line.substring(keyBegin);
    if (firstSpace == REMOVE.length() && line.startsWith(REMOVE)) {
      lruEntries.remove(key);
      return;
    }
  } else {
    key = line.substring(keyBegin, secondSpace);
  }

  Entry entry = lruEntries.get(key);
  if (entry == null) {
    entry = new Entry(key);
    lruEntries.put(key, entry);
  }

  if (secondSpace != -1 && firstSpace == CLEAN.length() && line.startsWith(CLEAN)) {
    String[] parts = line.substring(secondSpace + 1).split(" ");
    entry.readable = true;
    entry.currentEditor = null;
    entry.setLengths(parts);
  } else if (secondSpace == -1 && firstSpace == DIRTY.length() && line.startsWith(DIRTY)) {
    entry.currentEditor = new Editor(entry);
  } else if (secondSpace == -1 && firstSpace == READ.length() && line.startsWith(READ)) {
    // This work was already done by calling lruEntries.get().
  } else {
    throw new IOException("unexpected journal line: " + line);
  }
}
```

流程
* 1.校验头部
* 2.创建一个输入流StrictLineReader（DiskLruCache内部封装的，主要是对读取一行进行严格控制以及对文件结束的判断有所不同）
* 3.获取Key
* 4.根据Key获取对应的Entry（这一步**很重要**，它说明哪个文件被操作地最频繁，这样只就是排到LinkedHashMap的末尾，当缓存饱满时候，这个文件就不会率先被移除）
* 5.若是CLEAN，读取其文件大小，并且写入Entry中
* 6.若是DRITY，直接创建新的Entry

**特别注意判断条件：secondSpace**，意思是第二个空格的意思。只有READ的字段有第二个空格，且接着的是文件大小。
DiskLruCache Journal文件内容格式，如下：
```
libcore.io.DiskLruCache
1
1
1

DIRTY c3bac86f2e7a291a1a200b853835b664
CLEAN c3bac86f2e7a291a1a200b853835b664 32751
READ c3bac86f2e7a291a1a200b853835b664
DIRTY c59f9eec4b616dc6682c7fa8bd1e061f
CLEAN c59f9eec4b616dc6682c7fa8bd1e061f 8981
READ c59f9eec4b616dc6682c7fa8bd1e061f
DIRTY be8bdac81c12a08e15988555d85dfd2b
CLEAN be8bdac81c12a08e15988555d85dfd2b 222
READ be8bdac81c12a08e15988555d85dfd2b
DIRTY 536788f4dbdffeecfbb8f350a941eea3
REMOVE 536788f4dbdffeecfbb8f350a941eea3 
```

**至此，通过这一步。LinkedHashMap已经按照文件的访问频率进行了“排序”，为删除文件的时候打下了基础。**

# 整理日志与删除缓存
****
若我们频繁进行缓存**存储和读取**，很快日志文件就会变得很大。

为了避免日志文件的无限制增长，以及影响文件的读取以及存储，DiskLruCache中限定了日志为2000。 超越了2000行，那么就会启动日志整理了。

这个过程也包含了，**不常使用或没有使用的缓存文件的删除**。

在缓存文件保存、获取、删除的操作方法的末尾都会调用（**稍有不同**）：
```
if (size > maxSize || journalRebuildRequired()) {
  executorService.submit(cleanupCallable);
}
```
那么我们现在来看看这里面的方法：
```
/**
 * We only rebuild the journal when it will halve the size of the journal
 * and eliminate at least 2000 ops.
 */
private boolean journalRebuildRequired() {
  final int redundantOpCompactThreshold = 2000;
  return redundantOpCount >= redundantOpCompactThreshold //
      && redundantOpCount >= lruEntries.size();
}

final ThreadPoolExecutor executorService =
    new ThreadPoolExecutor(0, 1, 60L, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>());

private final Callable<Void> cleanupCallable = new Callable<Void>() {
  public Void call() throws Exception {
    synchronized (DiskLruCache.this) {
      if (journalWriter == null) {
        return null; // Closed.
      }
      trimToSize();
      if (journalRebuildRequired()) {
        rebuildJournal();
        redundantOpCount = 0;
      }
    }
    return null;
  }
};

private void trimToSize() throws IOException {
  while (size > maxSize) {
    Map.Entry<String, Entry> toEvict = lruEntries.entrySet().iterator().next();
    remove(toEvict.getKey());
  }
}
```
流程：
* 1.journalRebuildRequired判断，日志文件是否超过2000行
* 2.超过2000，让线程池执行一个Callback任务（和Runnable差不多，但Callback可以返回值）
* 3.调用trimToSize删除文件。LinkedHashMap从头开始遍历，并删除文件，直到小于指定大小。
* 4.重建日志rebuildJournal。

# 小结
****
上面已经将DiskLruCache最核心的部分分析完了，主要是理清DiskLruCache整个大的流程：DiskLruCache日志系统的创建以及缓存文件删除的部分。

相信其他的代码也没有太大的问题。若有问题可以参考鸿洋前辈的一篇文章：[Android DiskLruCache 源码解析 硬盘缓存的绝佳方案](http://blog.csdn.net/lmj623565791/article/details/47251585)

# 后记
****
文件包含：
journal、journal.tmp、journal.bkp：第一个是真正的日志系统文件；后面两个都是为了保证IO数据拷贝安全创建的临时文件

c3bac86f2e7a291a1a200b853835b664.0、c3bac86f2e7a291a1a200b853835b664.1：是**缓存文件**。  .0和.1的意思是这个key对应的两个value文件。但在正常使用过程中，我们一般都是**指定只对应1个**。所以一般只会看到**.0文件**

c3bac86f2e7a291a1a200b853835b664.0.tmp:DIRTY的文件。DIRTY实际上即使还**未commit**的缓存文件。这个文件就是c3bac86f2e7a291a1a200b853835b664.0的**前身**。当我们**调用commit之后**，后面的.tmp就会被去掉了（重命名）。

可以从源码的两个方法中看到：
```
public File getCleanFile(int i) {
  return new File(directory, key + "." + i);
}

public File getDirtyFile(int i) {
  return new File(directory, key + "." + i + ".tmp");
}
```


**-Hans 2016.05.28 16:12**