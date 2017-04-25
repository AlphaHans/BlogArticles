---
title: 【数据结构】震惊！SparseArray高效的原因居然是这样！
date: 2017-03-22 22:09
tag: SparseArray
category: 数据结构
---
## 概述

* SparseArray（稀疏数组） 以int为键，Object为值进行一一映射

* 对比HashMap，在内存方面SparseArray存在更大的优势

 * 避免了key的自动装箱

 * 其数据结构不依赖于额外的Entry对象来存储


<!-- more -->


### 内部实现

* 该容器使用数组作为数据结构来保持一一映射

* 使用**二分查找**对键进行定位搜索（搜索时间为O(logn)，HashMap是O(1)）

* 这样会带来缺点：

 * 当容器内在大量元素的时候，使用二分查找会带来很差的性能

 * 在存在大量元素以及涉及大量增删的时候，由于会引起数组的频繁变化；所以SparseArray的性能不一定是最佳的



### 优化

* 为了避免上述的缺点，提高性能；在删除的时候进行了优化：

 * 删除时，不直接压缩数组；而是对改为进行标记为DELETED

 * 该节点可以被后续相同的key使用，或者在后续回收中，统一被回收

 * 统一回收将在以下情况发生：

  * 数组扩容

  * 大小或值被恢复时



## 成员变量与构造方法

```

public class SparseArray<E> implements Cloneable {
    private static final Object DELETED = new Object();
    private boolean mGarbage = false;

    private int[] mKeys;
    private Object[] mValues;
    private int mSize;


    public SparseArray() {
        this(10);
    }

    public SparseArray(int initialCapacity) {
        if (initialCapacity == 0) {
            mKeys = EmptyArray.INT;
            mValues = EmptyArray.OBJECT;
        } else {
            mValues = ArrayUtils.newUnpaddedObjectArray(initialCapacity);
            mKeys = new int[mValues.length];
        }
        mSize = 0;
    }
```

* 创建了两个数组

 * 一个用于存储键、一个用于存储值

* DELETED的标记对象Object

* mGarbage的boolean判断（暂不知道干嘛，后续分析）



## get

```

public E get(int key) {
    return get(key, null);
}

@SuppressWarnings("unchecked")
public E get(int key, E valueIfKeyNotFound) {
    //二分查找
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i < 0 || mValues[i] == DELETED) {
        return valueIfKeyNotFound;
    } else {
        return (E) mValues[i];
    }
}
```

* 在`mKeys`对`key`进行二分查找，找到键的位置

* 如果pos小于零或者value已经从`mValue`数组删除，则返回自己指定的`valueIfKeyNotFound`

* 找到位置返回值



这里需要特别注意pos小于零的情况，是什么意思？下面先分析一下`ContainerHelpers.binarySearch`的二分查找法

```

binarySearch二分查找法



// This is Arrays.binarySearch(), but doesn't do any argument validation.
static int binarySearch(int[] array, int size, int value) {
    int lo = 0;
    int hi = size - 1;

    while (lo <= hi) {
        final int mid = (lo + hi) >>> 1;//>>>表示无符号位的右移，>>表示右移
        final int midVal = array[mid];

        if (midVal < value) {
            lo = mid + 1;
        } else if (midVal > value) {
            hi = mid - 1;
        } else {
            return mid;  // value found
        }
    }
    return ~lo;  // value not present //取反
}
```

**该二分查找算法是SparseArray算法核心之一**

注意一下`lo`的有两个意义：

* 当查找的key存在时候，返回的是key在数组位置的index

* 当查找的key不存在的时候，返回的index是该key应该存在的位置

举个例子：

```

public class BinarySearchTest {

    public static void main(String args[]) {
        int[] array = {2, 5, 8, 0, 0};
        System.out.println("返回值 = " + binarySearch(array, 3, 1) + " 重新取反值:" + ~binarySearch(array, 3, 1));
        System.out.println("返回值 = " + binarySearch(array, 3, 4) + " 重新取反值:" + ~binarySearch(array, 3, 4));
        System.out.println("返回值 = " + binarySearch(array, 3, 9) + " 重新取反值:" + ~binarySearch(array, 3, 9));
    }

    // This is Arrays.binarySearch(), but doesn't do any argument validation.
    static int binarySearch(int[] array, int size, int value) {
        int lo = 0;
        int hi = size - 1;

        while (lo <= hi) {
            final int mid = (lo + hi) >>> 1;
            final int midVal = array[mid];

            if (midVal < value) {
                lo = mid + 1;
            } else if (midVal > value) {
                hi = mid - 1;
            } else {
                return mid;  // value found
            }
        }
        return ~lo;  // value not present
    }
}
```

上述代码查找了三个key的位置，然后重新`~`后输出：

```

返回值 = -1 重新取反值:0

返回值 = -2 重新取反值:1

返回值 = -4 重新取反值:3

```

可以总结出规律：

* 当key不存在与array时候，返回的一定是负数

* 返回之后再次取反的值，就是该key应该处于的array的位置(比如说我们查找` key = 1`的位置，此时数组是`{2，5，8}`，而应该处于2之前，所以`key = 1`的`index应该是0`)



所以至此就可以回答上面留下的疑问，为何要判断`pos<0`的情况：

** `pos < 0 ` 表明当前`array`中不存在这个key **



## delete与remove

```

public void delete(int key) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i >= 0) {
        if (mValues[i] != DELETED) {
            mValues[i] = DELETED;
            mGarbage = true;
        }
    }
}


public E removeReturnOld(int key) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i >= 0) {
        if (mValues[i] != DELETED) {
            final E old = (E) mValues[i];
            mValues[i] = DELETED;
            mGarbage = true;
            return old;
        }
    }
    return null;
}


public void remove(int key) {
    delete(key);
}

public void removeAt(int index) {
    if (mValues[index] != DELETED) {
        mValues[index] = DELETED;
        mGarbage = true;
    }
}


public void removeAtRange(int index, int size) {
    final int end = Math.min(mSize, index + size);
    for (int i = index; i < end; i++) {
        removeAt(i);
    }
}
```



* delete和remove实际上是调用相同的方法

* 删除过程是使用二分查找法，先找到key的位置，然后将对应的values中的值标记为`DELETE`

* delete不涉及删除键，只涉及讲values的值置空

* 特别注意这个变量被赋值了`mGarbage = true`（目前暂不明用途，继续往后分析）





## put

```
public void put(int key, E value) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);
    if (i >= 0) {//已经存在该键-值，直接替换值
        mValues[i] = value;
    } else {//查找失败
        i = ~i;//再次取反，找到该键应该归属的位置
        if (i < mSize && mValues[i] == DELETED) {//如果值标记是被删除的话 则直接将键换掉传入的key，值为传入的值（这个下面会解释）
            mKeys[i] = key;
            mValues[i] = value;
            return;
        }
        if (mGarbage && mSize >= mKeys.length) {//如果数组被回收过 则需要调用gc，为下面的数组扩容做准备
            gc();
            // Search again because indices may have changed.
            i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);//gc执行完之后，重新进行二分查找正确的位置
        }
        mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
        mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
        mSize++;
    }
}
```



* 搞清楚`mSize`的变化，`remove`和`delete`的操作不会对`mSize`进行影响，只是进行`mValues`数组对应`index`置为`DELTED`

* `if(i < mSize && mValues[i] == DELETED)`在判断什么？

 * 还记得我们上面二分查找举得例子么，数组是`{2,5,8}`

 * 现在执行`sp.remove(2)`然后再进行`sp.put(1,value)`，就会进入到该判断了

 * 因为`remove`方法只影响`mValues`，当它为`DELETED`时候，说明对应的`key`也无意义了

 * 因此该if内执行的是直接对key-value进行替换

* `gc()`可以看出是数组回收的过程，这里`mGarbage`就用上了。 （该方法后续分析）

* `GrowingArrayUtils.inser()`从类名可以看出，当空间不够的时候，数组会自增后插入（因为AS没有该源码，就不分析了）



## gc

```

private void gc() {
    // Log.e("SparseArray", "gc start with " + mSize);
    int n = mSize;
    int o = 0;
    int[] keys = mKeys;
    Object[] values = mValues;
    for (int i = 0; i < n; i++) {//数组往前压缩，将原本的DELETED对应的key-value都删掉
        Object val = values[i];
        if (val != DELETED) {
            if (i != o) {
                keys[o] = keys[i];
                values[o] = val;
                values[i] = null;
            }
            o++;
        }
    }
    mGarbage = false;
    mSize = o;
    // Log.e("SparseArray", "gc end with " + mSize);
}
```



 * `gc()`很简单，就是讲数组往前部压缩

 * 举个例子：

 * `mKeys = {2,4,5,6} mValues = {Object1,DELETED,Object2,DELETED}`

 * 调用`gc()`之后：`mKeys = {2,5} mValues = {Object1,Object2}`

 * 起到紧凑数组的作用



## keyAt与indexOfkey



```

public int keyAt(int index) {
    if (mGarbage) {
        gc();
    }

    return mKeys[index];
}


public int indexOfKey(int key) {
    if (mGarbage) {
        gc();
    }

    return ContainerHelpers.binarySearch(mKeys, mSize, key);
}
```



* 这两个方法容易混杂，所以通过源码来讲比较简单

* `keyAt`直接通过`index`指明获取`mKeys`数组中第几位的`key`（用于数组遍历）

 * 该方法可能存在不可靠性.因为有可能获取到的的`key`是默认数组默认初始化的`0`。

 * 为了避免上面的情况，需要结合`size()`方法来遍历

* `indexOfKey`通过`key`来查明该`key`在`mKeys`所在的`index`



## valueAt与indexOfValue



```


@SuppressWarnings("unchecked")
public E valueAt(int index) {
    if (mGarbage) {
        gc();
    }

    return (E) mValues[index];
}

public int indexOfValue(E value) {
    if (mGarbage) {
        gc();
    }

    for (int i = 0; i < mSize; i++)
        if (mValues[i] == value)
            return i;

    return -1;
}
```

* 大致和上面的`keyAt与indexOfKey`相同

* 使用`keyAt`和`valueAt`可以实现遍历



## setValueAt

```

public void setValueAt(int index, E value) {
    if (mGarbage) {
        gc();
    }

    mValues[index] = value;
}
```

* 该方法使用需要调用者确定当前数据内容

* 举个例子：

 * 已有`mKeys = {2,5,8} mValues = {o1,o2,o3}`此时如果调用`setValueAt(0,o4)`

 * 那么就会变成`mKeys = {2,5,8} mValues = {o4,o2,o3}`

 * 如果调用`setValueAt(3,o4)`那么内存情况是`mKeys = {2,5,8,0} mValues = {o1,o2,o3,o4}`而可获取的情况是`mKeys = {2,5,8} mValues = {o1,o2,o3}`

 * 所以该方法需要调用者熟悉当前数据内容



## append

```

public void append(int key, E value) {
    //这里是一个代码的容错机制，避免开发者以为自己的现在key最大
    if (mSize != 0 && key <= mKeys[mSize - 1]) {//key不是大于数组所有的key的时候
        put(key, value);//重新执行put
        return;
    }

    if (mGarbage && mSize >= mKeys.length) {//扩容准备
        gc();
    }

    mKeys = GrowingArrayUtils.append(mKeys, mSize, key);//直接插入（理论上是直接插入到末尾）
    mValues = GrowingArrayUtils.append(mValues, mSize, value);
    mSize++;
}
```

* 该方法，针对于插入的`key`大于所有已存在的`key`的优化插入

* 第一个`if`判断，该`key`是否真的为最大。若不是，重新执行`put`方法

* 如果该`key`的确最大，那么直接`GrowingArrayUtils.append()`。理论上这个是直接拼接到数组有效位末尾，因为该`key`最大没必要二分查找了

* **一个不是很成熟的结论：理论上使用`append`可能比`put`会有更高的性能，因为在`key`最大情况下可以免去二分查找的开销**



## 与HashMap的对比

* 查找速度

 * `HashMap`基于`hash`查找，时间复杂度是O(1)

 * `SparseArray`基于二分查找，时间复杂度是O(logn)

 * 数据量大的时候，`SparseArray`性能会急剧恶化；数据量小，两者应该区别不大（- - 不是很严谨，但是从时间复杂度来说是体现很明显的）

* 插入速度

 * 对于大量数据来说，由于`SparseArray`二分查找会带来较大的性能开销，而且可能由于数组长度限制，会导致`gc()`的压缩调用，以及导致大数组的复制

 * 插入速度还是`HashMap`略胜一筹

* 删除速度

 * 还是由于`SparseArray`二分查找（数据量较大情况），`HashMap`略胜一筹

* 内存开销

 * `HashMap`由于自动装箱以及`HashEntry`额外占用内存，所以`HashMap`内存占用较大

 * `SparseArray`基于数组，不需要自动装箱，内存占用较小



一句话概括：**SparseArray并不是完全优于HashMap，要根据实际情况具体分析**



## 感悟

该数据结构实际上很早就用过，一直没有来探索其中的奥秘。 在前几天今日头条面试官问我细节的时候，才发现自己才疏学浅。特此进行一波总结。

实际上数据结构真的是一门博大精深的学科...这些编写代码的人总能想到一些高潮的方式来优化性能；这些思想是很值得我们学习的！

从中吸取思想，运用到我们自己的实战经验中去。



个人认为已经将`SparseArray`完全解剖完毕，若有错漏请批评指正。