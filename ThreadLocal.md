#ThreadLocal

ThreadLocal是一个线程内部的数据存储类，通过它可以在指定的线程中存储数据，数据存储后，只有在指定的线程中可以获取到存储的数据，对于其他线程来说则无法获取到数据。Looper、ActivityThread以及AMS中都用到了ThreadLocal。

##ThreadLocal示例

```
       private ThreadLocal<Boolean> mBooleanThreadLocal = new ThreadLocal<Boolean>();
       mBooleanThreadLocal.set(true);
        Log.d("ThreadLocal", "[Thread#main]mBooleanThreadLocal=" + mBooleanThreadLocal.get());
        new Thread("Thread#1") {
            @Override
            public void run() {
                mBooleanThreadLocal.set(false);
                Log.d("ThreadLocal", "[Thread#1]mBooleanThreadLocal=" + mBooleanThreadLocal.get());
            }
        }.start();
        new Thread("Thread#2") {
            @Override
            public void run() {
                Log.d("ThreadLocal", "[Thread#2]mBooleanThreadLocal=" + mBooleanThreadLocal.get());
            }
        }.start();
```

Log:

> [Thread#main]mBooleanThreadLocal=true
>  [Thread#1]mBooleanThreadLocal=false
>  [Thread#2]mBooleanThreadLocal=null

虽然在不同线程中访问的是同一个TheadLocal对象，但是它们通过TheadLocal获取到的值却是不一样的。不同线程访问同一个ThreadLocal的get方法，ThreadLocal内部会从各自的线程中取出一个数组，然后再从数组中根据当前ThreadLocal的索引去查找出对应的value值。

##源码解析

首先看ThreadLocal的set方法：

```

    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```

在set的过程中，首先获取当前线程的ThreadLocalMap，如果取到的map为null，则使用createMap方法初始化map，否则直接赋值。createMap的定义如下：

```
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

下面重点来看一下ThreadLocalMap的定义：

```
    static class ThreadLocalMap {
    
        static class Entry extends WeakReference<ThreadLocal> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal k, Object v) {
                super(k);
                value = v;
            }
        }

        private static final int INITIAL_CAPACITY = 16;
        private Entry[] table;
        private int size = 0;
        private int threshold; // Default to 0
        private void setThreshold(int len) {
            threshold = len * 2 / 3;
        }
        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }
        private static int prevIndex(int i, int len) {
            return ((i - 1 >= 0) ? i - 1 : len - 1);
        }
        ThreadLocalMap(ThreadLocal firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
        private ThreadLocalMap(ThreadLocalMap parentMap) {
            Entry[] parentTable = parentMap.table;
            int len = parentTable.length;
            setThreshold(len);
            table = new Entry[len];

            for (int j = 0; j < len; j++) {
                Entry e = parentTable[j];
                if (e != null) {
                    ThreadLocal key = e.get();
                    if (key != null) {
                        Object value = key.childValue(e.value);
                        Entry c = new Entry(key, value);
                        int h = key.threadLocalHashCode & (len - 1);
                        while (table[h] != null)
                            h = nextIndex(h, len);
                        table[h] = c;
                        size++;
                    }
                }
            }
        }
        private Entry getEntry(ThreadLocal key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }
        private Entry getEntryAfterMiss(ThreadLocal key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {
                ThreadLocal k = e.get();
                if (k == key)
                    return e;
                if (k == null)
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }
        private void set(ThreadLocal key, Object value) {

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }

        private void remove(ThreadLocal key) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.get() == key) {
                    e.clear();
                    expungeStaleEntry(i);
                    return;
                }
            }
        }
        ...
    }
```



### ThreadLocal变量在ThreadLocalMap的Entry数组中位置的确定？
```
        ThreadLocalMap(ThreadLocal firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }

```
核心代码是`  int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);`，`INITIAL_CAPACITY`是数组初始化长度。我们来看一下threadLocalHashCode的定义及赋值：

```
    private final int threadLocalHashCode = nextHashCode();
    private static AtomicInteger nextHashCode =
        new AtomicInteger();

    private static final int HASH_INCREMENT = 0x61c88647;

    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }


```

原来HashCode从0开始不断累加0x61c88647生成的。其目的在`HASH_INCREMENT `的注释中已经提到了：为了让HashCode能更好地分布在尺寸为2的N次方的数组里。我们来测试一下：

```
    private static AtomicInteger nextHashCode =
            new AtomicInteger();

    private static final int HASH_INCREMENT = 0x61c88647;

    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }   

```

```
    
        Log.d("Test", "Size is 16:");
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 16; i++) {
            sb.append((nextHashCode() & 15) + "  ");
        }
        Log.d("Test", sb.toString()); 

```
Log:
> D/Test: Size is 16:
> 
> D/Test: 0  7  14  5  12  3  10  1  8  15  6  13  4  11  2  9
> 

```
        Log.d("Test", "Size is 32:");
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 32; i++) {
            sb.append((nextHashCode() & 31) + "  ");
        }
        Log.d("Test", sb.toString());

```

Log:
> D/Test: Size is 32:
> 
> D/Test: 0  7  14  21  28  3  10  17  24  31  6  13  20  27  2  9  16  23  30  5  12  19  26  1  8  15  22  29  4  11  18  25

可以看到分布真的十分均匀，而且目测没有任何冲突（要验证一下），相当神奇。为什么Ox61c8847这个数字这么神奇呢？。。。。。（>_< |）


###Entry为什么要继承WeakReference<ThreadLocal>？

```
        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal k, Object v) {
                super(k);
                value = v;
            }
        }

```


    
    



