#   LruCache原理分析

基本用法:
```
    public class MyLruCache {
    //    private static int cacheSize = 4 * 1024 * 1024; // 4MiB
        private static int cacheSize = (int) (Runtime.getRuntime().totalMemory()/1024)/8;

        private static LruCache<String, Bitmap> bitmapCache = new LruCache<String, Bitmap>(cacheSize) {//创建LruCache实例时,提供LruCache可占用最大内存大小.
            //重写sizeOf方法.计算每个value所占内存的大小
            protected int sizeOf(String key, Bitmap value) {
                return value.getByteCount();
            }
        };

        public static void put(String url,Bitmap bitmap){
            bitmapCache.put(url,bitmap);
        }

        public static Bitmap get(String url){
            return bitmapCache.get(url);
        }

        public static void clear(){
            bitmapCache.evictAll();
        }
    }
```

1. LruCache的初始化:
```
    private final LinkedHashMap<K, V> map;
    private int maxSize;

    public LruCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        //创建一个初始大小为0.加载因子为0.75,排序模式为访问顺序的LinkedHashMap.
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
    }
```

2.  LruCache的put方法:
```
    public final V put(K key, V value) {
        //确保key,value不为null才能存
        if (key == null || value == null) {
            throw new NullPointerException("key == null || value == null");
        }

        V previous;
        //put过程使用了同步代码块.所以put过程线程安全.
        synchronized (this) {
            putCount++;
            //safeSizeOf内部调用sizeOf计算当前value所占用的内存大小.
            //size标识当前map中保存的value的总大小.
            size += safeSizeOf(key, value);
            //将对应的key和value保存进map中.并将key之前对应的value返回给previous.
            previous = map.put(key, value);
            //如果previous不为null,则size的值需要减去previous所占内存的大小.因为它已经不再map中了.
            if (previous != null) {
                size -= safeSizeOf(key, previous);
            }
        }

        if (previous != null) {
            //该方法通知条目被移除了.previous表示旧值,value表示新值.
            //false标识是由remove或者put方法被动触发.true表示.evictAll方法或者trimToSize中触发.
            entryRemoved(false, key, previous, value);
        }

        //调整size不能操作maxSize.
        trimToSize(maxSize);

        return previous;
    }
```
3.  LruCache的trimToSize方法:
```
    public void trimToSize(int maxSize) {
        while (true) {
            K key;
            V value;
            synchronized (this) {
                //如果当前条目的总内存大小小于设置的maxSize.或者map为空.则不做任何操作.结束循环.
                if (size <= maxSize || map.isEmpty()) {
                    break;
                }

                //能执行到这里,说明size已经>=maxSize了.所以需要移除条目来达到size小于maxSize的时候.

                //迭代器取下一个条目,然后从map中移除该条目.并且重新调整size的大小.
                Map.Entry<K, V> toEvict = map.entrySet().iterator().next();
                key = toEvict.getKey();
                value = toEvict.getValue();
                map.remove(key);
                size -= safeSizeOf(key, value);
                evictionCount++;
            }

            //通知条目被移除
            entryRemoved(true, key, value, null);
        }
    }
```

4.  LruCache的get方法:
```
    public final V get(K key) {
        //不允许key为null
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        V mapValue;
        synchronized (this) {
            //从map中取值,如果取到了就直接返回.
            mapValue = map.get(key);
            if (mapValue != null) {
                hitCount++;
                return mapValue;
            }
            missCount++;
        }

        //如果执行到这里,说明缓存中是没有找到目标条目.

        //1.试图根据key值创建对应的条目.默认实现返回null.可以自己覆写.
        V createdValue = create(key);

        //如果没创建成功.则返回null.
        if (createdValue == null) {
            return null;
        }


        synchronized (this) {
            createCount++;
            //2.创建成功,则将条目存入map之中.
            mapValue = map.put(key, createdValue);

            //正常情况.mapValue为null.但是可能存在并发的情况.在create的时候,别的线程已经put进去了.
            if (mapValue != null) {
                //所以这里如果mapValue不为null.则重新将mapValue存进map中.
                map.put(key, mapValue);
            } else {
                //如果mapValue为null,则可以正常执行size的重新计算.
                size += safeSizeOf(key, createdValue);
            }
        }

        if (mapValue != null) {
            //如果mapValue不为null,则通知新创建的Value被移除.
            entryRemoved(false, key, createdValue, mapValue);
            return mapValue;//返回mapValue
        } else {
            //mapValue为空,说明新存入了createValue.
            //因此,需要根据size与maxSize调整内存,移除条目.
            trimToSize(maxSize);
            return createdValue;//返回新创建的条目.createdValue.
        }
    }
```
5.  LruCache是要根据最近使用的情况缓存内容.换句话说,就是当缓存内存不足时,应该移除那些最不常用的条目.从上面的代码中.可以看出.其实对于根据使用记录对条目排序,
    完全是由LinkedHashMap支持的.所以接下来分析LinkedHashMap是如何实现根据访问记录对条目排序的.

6.  LinkedHashMap的put方法:LinkedHashMap继承自HashMap.并没有覆写该方法.所以put方法是HashMap的put方法.
```
    public V put(K key, V value) {

        //table是一个数组,HashMapEntry<K,V>[].
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
        if (key == null)
            return putForNullKey(value);
        //对key计算hash值.
        int hash = sun.misc.Hashing.singleWordWangJenkinsHash(key);
        //根据hash值以及table的长度.定位当前hash值在table中的位置.
        int i = indexFor(hash, table.length);

        //table每个节点处的条目,又是链表结构.所以循环取链表的下一条目,判断
        //条目的key值是否对应的上.
        for (HashMapEntry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;

            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                //如果hash值以及key值都能对应则将新值存赋值给节点的value字段.
                V oldValue = e.value;
                e.value = value;
                //记录访问记录,HashMap的recoredAccess是空实现.但是LinkedHashMap实现了该方法.
                e.recordAccess(this);
                //返回旧值.
                return oldValue;
            }
        }

        modCount++;
        //如果没有查找到对应的key值节点.则新添加一个节点.LinkedHashMap重写了addEntry方法.
        addEntry(hash, key, value, i);
        return null;
    }
```

7.  LinkedHashMap的addEntry方法:
```
    void addEntry(int hash, K key, V value, int bucketIndex) {

        LinkedHashMapEntry<K,V> eldest = header.after;
        if (eldest != header) {
            boolean removeEldest;
            size++;
            try {
                //在7.0上该方法返回的是false
                removeEldest = removeEldestEntry(eldest);
            } finally {
                size--;
            }
            if (removeEldest) {
                removeEntryForKey(eldest.key);
            }
        }
        //调用HashMap的addEntry方法.
        super.addEntry(hash, key, value, bucketIndex);
    }
```

8.  HashMap的addEntry方法:
```
    void addEntry(int hash, K key, V value, int bucketIndex) {
        //如果当前size达到了阈值,则需要扩大table的大小.
        if ((size >= threshold) && (null != table[bucketIndex])) {
            resize(2 * table.length);
            hash = (null != key) ? sun.misc.Hashing.singleWordWangJenkinsHash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }

        //创建节点.LinkedHashMap重写了该方法
        createEntry(hash, key, value, bucketIndex);
    }
```
9.  LinkedHashMap的createEntry方法:
```
    void createEntry(int hash, K key, V value, int bucketIndex) {
        //获取当前数组中buketIndex位置的条目.
        HashMapEntry<K,V> old = table[bucketIndex];
        //创建一个新的LinkedHashMapEntry节点.将next指向old.
        LinkedHashMapEntry<K,V> e = new LinkedHashMapEntry<>(hash, key, value, old);
        //然后将数组中的条目指向新创建的的节点
        table[bucketIndex] = e;

        //上面的算法思想就是,对于根据hash值被分配到同一个数组位置的条目.新创建的永远在链表的头部.

        //这里header始终是队尾哨兵
        e.addBefore(header);
        size++;
    }

    //LinekdHashMap被初始化时被调用.
    void init() {
        header = new LinkedHashMapEntry<>(-1, null, null, null);
        header.before = header.after = header;
    }
```

10. LinkedHashMapEntry的addBefore方法.对于新创建的节点.始终放在header哨兵节点的前一个.也就是越新的节点,越在队尾.
```
        LinkedHashMapEntry<K,V> before, after;//LinkedHashMapEntry是双向链表结构
        private void addBefore(LinkedHashMapEntry<K,V> existingEntry) {
            //把当前节点的下一个设置为existingEntry.
            after  = existingEntry;
            //把当前节点的上一个设置为existingEntry的上一个.
            before = existingEntry.before;
            //把当前节点的上一个的下一个指向当前节点.
            before.after = this;
            //把当前节点的下一个的上一个之前当前节点.
            after.before = this;

            //总述就是,将当前节点放到existingEntry节点之前.
            //第一个节点的前一个也是header哨兵节点.
        }
```

11. LinkedHashMapEntry的recordAccess方法:
```
        void recordAccess(HashMap<K,V> m) {
            LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
            //如果设置了访问顺序调整元素顺序.
            if (lm.accessOrder) {
                lm.modCount++;
                //当前节点从链表中移除.
                remove();
                //重新添加回链表中.添加到队尾.
                addBefore(lm.header);
            }
        }

        private void remove() {
            before.after = after;
            after.before = before;
        }
```

12. LinkedHashMap的LinkedHashIterator迭代器.
```
    private class EntryIterator extends LinkedHashIterator<Map.Entry<K,V>> {
        public Map.Entry<K,V> next() { return nextEntry(); }
    }

    private abstract class LinkedHashIterator<T> implements Iterator<T> {
            //可以发现nextEntry初始时指向的是第一个节点.所以说.迭代器返回节点是从最旧的节点开始返回.
            LinkedHashMapEntry<K,V> nextEntry    = header.after;
            LinkedHashMapEntry<K,V> lastReturned = null;

            int expectedModCount = modCount;

            public boolean hasNext() {
                return nextEntry != header;
            }

            public void remove() {
                if (lastReturned == null)
                    throw new IllegalStateException();
                if (modCount != expectedModCount)
                    throw new ConcurrentModificationException();

                LinkedHashMap.this.remove(lastReturned.key);
                lastReturned = null;
                expectedModCount = modCount;
            }

            Entry<K,V> nextEntry() {
                if (modCount != expectedModCount)
                    throw new ConcurrentModificationException();
                if (nextEntry == header)
                    throw new NoSuchElementException();

                LinkedHashMapEntry<K,V> e = lastReturned = nextEntry;
                //把nextEntry指向下一个节点.
                nextEntry = e.after;
                return e;
            }
        }
```

分析到这里.其实LruCache的原理基本已经搞清楚了.通过HashMap的table数组保存数据.通过LinkedHashMap的header双向链表储存访问顺序.