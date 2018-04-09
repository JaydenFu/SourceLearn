#   LinkedHashMap实现原理

```
LinkedHashMap继承了HashMap，其内部维护了2个LinkedHashMapEntry节点：表示头节点，尾节点

transient LinkedHashMapEntry<K,V> head;
transient LinkedHashMapEntry<K,V> tail;

static class LinkedHashMapEntry<K,V> extends HashMap.Node<K,V> {
    //before，after节点表示迭代顺序上的前一个节点和后一个节点
    LinkedHashMapEntry<K,V> before, after;
    LinkedHashMapEntry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}

如果设置了accessOrder为true，则迭代顺序是从最老访问的节点到最近访问的节点
如果accessOrder为false，则迭代顺序是从最先添加的节点到最后添加的节点
```

1. LinkedHashMap初始化： LinkedHashMap初始化和HashMap表现是一致的，只是多了一个accessOrder用于控制迭代顺序
```
    //1. 默认初始化
    public LinkedHashMap() {
        super();
        //accessOrder 表示迭代器迭代的时候按访问顺序输出还是按插入顺序输出
        accessOrder = false;
    }

    //2. 带初始化容量初始化
     public LinkedHashMap(int initialCapacity) {
            super(initialCapacity);
            accessOrder = false;
     }

    //3. 带初始化容量和加载因子初始化
     public LinkedHashMap(int initialCapacity, float loadFactor) {
         super(initialCapacity, loadFactor);
         accessOrder = false;
     }
     ...
```

2. LinkedHashMap.put(K key,V val)
```
LinkedHashMap.put方法并没有重写HashMap的put方法，下面是HashMap中的putVal方法

   final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            //LinkedHashMap重写了newNode方法，返回LinkedHashMapEntry实例
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {

                        //LinkedHashMap重写了newNode方法，返回LinkedHashMapEntry实例
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;

                //1. LinekdHashMap通过该方法调整顺序
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();

        //2. LinkedHashMap通过该方法调整顺序
        afterNodeInsertion(evict);
        return null;
    }

    //LinkedHashMap的newNode方法：
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {

        //创建节点
        LinkedHashMapEntry<K,V> p =
            new LinkedHashMapEntry<K,V>(hash, key, value, e);
        //将节点链接到LinkedHashMapd的tail节点
        linkNodeLast(p);
        return p;
    }

    //LinkedHashMap的linkNodeLast方法
    private void linkNodeLast(LinkedHashMapEntry<K,V> p) {
        LinkedHashMapEntry<K,V> last = tail;
        //将tail节点指向p，表示p为当前的为节点
        tail = p;
        //如果之前tail为null，则说明之前还没有节点，需要将head节点指向p，表示p为头节点
        if (last == null)
            head = p;
        else {
            //如果之前tail不null，则将当前节点的before指向last，并且将last的after指向p
            p.before = last;
            last.after = p;
        }
    }
```

3.  LinkedHashMap.afterNodeAccess(Node e)
```
void afterNodeAccess(Node<K,V> e) {
        LinkedHashMapEntry<K,V> last;

        //如果accessOrder为true,并且tail节点不等于e，那么tail需要调整为指向e节点
        if (accessOrder && (last = tail) != e) {

            //将p节点指向e节点，b节点指向e的before节点，a节点指向e节点的after节点
            LinkedHashMapEntry<K,V> p =
                (LinkedHashMapEntry<K,V>)e, b = p.before, a = p.after;

            //因为p节点需要变成最后一个节点，因此首先将after设置为null
            p.after = null;

            if (b == null)
                //如果b为null,则说明p节点是头节点，因此这个时候，需要将head节点指向a节点
                head = a;
            else
                //如果b不为null，则将b的after指向a节点
                b.after = a;


            if (a != null)
                //如果a不为null，则需要将a的before指向b
                a.before = b;
            else
                //如果a为nul，则说明p节点是当前最后一个节点，因此将last指向b节点
                last = b;

            //指向到这里，节点p就从原来的链表中断开了
            //后面再讲节点p添加到尾节点上

            if (last == null)
                //如果last节点为null，则说明p节点是头节点
                head = p;
            else {
                //将p的before指向last，将last的after向p，这个时候，p就被添加到链表的尾部了
                p.before = last;
                last.after = p;
            }

            //再讲tail指向p
            tail = p;
            ++modCount;
        }
    }
```

4.  LinkedHashMap.afterNodeInsertion
```
    void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMapEntry<K,V> first;

        //removeEldestEntry默认是false
        //如果removeEldestEntry为true，并且evict为true，则需要移除最老的节点
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
    }
```

5. LinkedHashMap.afterNodeRemoval
```
    //将e节点从head，tail所在的链表中断开
    void afterNodeRemoval(Node<K,V> e) { // unlink
        LinkedHashMapEntry<K,V> p =
            (LinkedHashMapEntry<K,V>)e, b = p.before, a = p.after;
        p.before = p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;

        if (a == null)
            tail = b;
        else
            a.before = b;
    }
```

6.  LinkedHashMap.entrySet
```
    public Set<Map.Entry<K,V>> entrySet() {

        //创建LinkedEntrySet,赋值给entrySet
        Set<Map.Entry<K,V>> es;
        return (es = entrySet) == null ? (entrySet = new LinkedEntrySet()) : es;
    }
```

7.  LinkedEntrySet.iterator
```
    public final Iterator<Map.Entry<K,V>> iterator() {
        //返回的是LinkedEntryIterator
        return new LinkedEntryIterator();
    }
```

8.  LinkedEntryIterator
```
    final class LinkedEntryIterator extends LinkedHashIterator
        implements Iterator<Map.Entry<K,V>> {
        public final Map.Entry<K,V> next() { return nextNode(); }
    }

     abstract class LinkedHashIterator {
            LinkedHashMapEntry<K,V> next;//指向下一个节点
            LinkedHashMapEntry<K,V> current;//指向当前节点
            int expectedModCount;

            LinkedHashIterator() {
                //初始化时next指向head节点，因此该迭代器，是从头节点开始往尾节点遍历
                next = head;
                expectedModCount = modCount;
                current = null;
            }

            public final boolean hasNext() {
                return next != null;
            }

            final LinkedHashMapEntry<K,V> nextNode() {
                LinkedHashMapEntry<K,V> e = next;
                if (modCount != expectedModCount)
                    throw new ConcurrentModificationException();
                if (e == null)
                    throw new NoSuchElementException();
                current = e;
                next = e.after;
                return e;
            }

            public final void remove() {
                Node<K,V> p = current;
                if (p == null)
                    throw new IllegalStateException();
                if (modCount != expectedModCount)
                    throw new ConcurrentModificationException();
                current = null;
                K key = p.key;
                removeNode(hash(key), key, null, false, false);
                expectedModCount = modCount;
            }
        }
```