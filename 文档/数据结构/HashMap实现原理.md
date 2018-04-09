# HashMap 实现原理

内部维护Node<K,V>[] table，int threshold
```
   static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
        .....
   }

    static final class TreeNode<K,V> extends LinkedHashMap.LinkedHashMapEntry<K,V> {
       TreeNode<K,V> parent;  // red-black tree links
       TreeNode<K,V> left;
       TreeNode<K,V> right;
       TreeNode<K,V> prev;    // needed to unlink next upon deletion
       boolean red;

       TreeNode(int hash, K key, V val, Node<K,V> next) {
           super(hash, key, val, next);
       }
           ...
    }
```

```
总结：
HashMap插入值流程：
    1. 计算key值得hash值，根据hash值于数组容量-1的与值计算出插入的位置
    2. 如果table数组中该位置的Node节点为null，则直接创建新的节点存入table中
    3. 如果table数组中该位置的Node节点不为null，则判断该节点的key值是否和插入的key值一样，一样，则将这个
       节点的value设置为新的value
    4. 如果table数组中该位置的Node节点的key和插入的key不一样，
        * 则判断Node节点是否是TreeNode节点，如果是TreeNode节点，则遍历这个红黑树，如果找到相同key值得节点，
            则替换value，如果没找到，则新创建节点，添加新红黑树。如果Node节点
        * 不是TreeNode节点，则遍历链表，如果找到相同key值得节点，则替换value，如果没找到，则创建新的节点，
            添加到链表尾部，然后判断链表的长度是否大于7，如果大于7，则判断当前table的容量是否大于64，如果不大于，则
            执行resize扩容，否则，将这个table中该位置的链表转为红黑树
    5.  如果原来map中就存在相应的key，则直接返回原来的value。
    6.  如果原来map中不存在相应的key，也就是说向map中新添加元素成功了，则判断size个数是否大于threshold,大于就需要扩容


```


1. HashMap的初始化
```
    //1. 默认初始化
    public HashMap() {
        //将loadFactor价值因子设置为默认加载因子，0.75f
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

    //2. 设置初始容量初始化，也是采用默认加载因子
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    //3. 设置初始容量和加载因子进行初始化
    public HashMap(int initialCapacity, float loadFactor) {

        //1. 设置的初始容量不能小于0
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);

        //2. 如果设置的初始容量大于2的30次方，则将initialCapacity改为2的30次方
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;

        //3. 如果加载因子不合法，小于等于0或者isNaN，则抛出异常
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                            loadFactor);
        //4. 赋值加载因子
        this.loadFactor = loadFactor;

        //5. tableSizeFor方法会根据initialCapacitty重新计算容量，保证容量是2的幂，最大值为2的30次方
        //假设initialCapacity为10，则threshold的值为16.
        this.threshold = tableSizeFor(initialCapacity);
    }


    //4. 通过给定的map初始化新的HashMap
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
```

2. HashMap.put(K key,V val)
```
    //
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    //对key取hash
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }


    //putVal
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                       boolean evict) {

        Node<K,V>[] tab; Node<K,V> p; int n, i;

        //1. 当前table为null，或者table数组的长度为0，则需要通过resiez方法
        重新创建table数组
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;

        //2. 用hash值与数组容量-1做与运算计算出放置的位置，如果当前位置还没有节点
        //则创建新的节点，然后直接放入tab数组对应的位置中
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            //3. 如果计算出的位置处，已经有节点了
            Node<K,V> e; K k;

            //4. 如果tab中的对应位置的节点的hash和参数hash相同，且节点的key与参数key
            //相同，则说明当前节点的key值和我们新添加的key值是相同的，那么直接将p接点赋值给e节点
            //e节点用于后面赋值value
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {

                //5. 如果p节点的key值不同于传入的参数key，那么就需要遍历p节点所在的链表，
                //直到找到相同key值得节点，或者到链表的尾部还没找到，就创建新的节点，并让p.next指向
                //新创建的节点
                for (int binCount = 0; ; ++binCount) {


                    if ((e = p.next) == null) {
                        //执行到这里，说明已经遍历到尾节点了
                        p.next = newNode(hash, key, value, null);

                        //TREEIFY_THRESHOLD为8，
                        //判断链表长度大于等于7的时候，则需要扩容或者转为红黑树
                        //如果tab的length小于64，则扩容，如果tab的length大于64了，则
                        //转为2叉树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }


                    //执行到这里说明,e节点的key和参数key完全相同
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;

                    p = e;
                }
            }


            //e节点不为null，说明Map中本来就存在相同的key了，这个时候，替换掉原来的value
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;

                //afterNodAccess在HashMap中是空实现，是用于LinkedHashMap的
                afterNodeAccess(e);
                //直接返回旧的值
                return oldValue;
            }
        }

        //执行到这里，说明是新插入了值，那么size需要+1，然后判断size是否大于threshold，如果大，则需要扩容
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }


    //resize
     final Node<K,V>[] resize() {

        //1.将当前成员变量table赋值给oldTab
        Node<K,V>[] oldTab = table;

        //2. 计算oldTab的容量
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;

        //3.如果oldCap大于0，也就是说原来的容量大于0
        if (oldCap > 0) {
            //4. 则再判断原来的容量是否大于等于2的30次方，则将threshold赋值为Integer.Max_VALUE
            //然后返回原来的table
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //5. 将newCap设置为oldCap的2倍，如果newCap小于2的30次方，并且oldCap大于16.
            //则newThr设置为oldThr的2倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0)
            //6. 如果oldCap==0，并且oldThr大于0，则newCap就设置为oldThr
            //对于采用初始化容量初始化的HashMap而言，第一次resize会执行这里
            newCap = oldThr;
        else {
            //7.通常对于默认初始化的HashMap而言，第一次resize会执行这里
            //将newCap赋值为16，newThr设置为12
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }

        //8. 如果newThr为0
        if (newThr == 0) {
            //加载因子*newCap计算出ft
            float ft = (float)newCap * loadFactor;
            //如果ft的值小于2的30次方并且newCap的值也小于2的30次方，则newThr为ft，否则newThr
            //为Integer.Max_VALUE
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }

        //9. 将newThr赋值给成员变量threshold,threshold表示size达到这个数时，需要扩容
        threshold = newThr;

        //10. 创建新的Node数组，容量为newCap，并将newTab赋值给table
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;

        //11. 如果oldTab不为null，则需要将oldTab中的数据赋值到新的table中
        if (oldTab != null) {

            //循环遍历原来的数组
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;

                //取出数组中相应角标的节点，为空就不管，不为空时，由于这个节点元素是Node，单链表，所以需要
                //通过while循环遍历链表上的所有节点
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;

                    //如果当前节点的next为null，则说明当前只有一个节点，直接将当前节点存入
                    //新数组中，角标由节点的hash变量和新数组的容量计算得出
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            // 如果e.hash&oldCap为0，则说明e.hash的高位为0，
                            // 这里的高位是指的与oldCap对应的高1位，这样的话，对于扩容
                            // 后的数组而言，计算出的角标和oldCap时是一样的，所以在链表中保存这些
                            // 这些元素，然后将链表头添加到新的table数组中对应的j位置处
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                            //到这里，说明e.hash的高位为1，这里的高位和上面指的一样，这样则
                            //元素在新数组中的位置始终为当前位置+oldCap
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            //将loHead节点添加进新数组中
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            //将hiHead节点添加进新数组中
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }

        //返回新的数组
        return newTab;
    }


    //扩容或者转为红黑树
    final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        //如果table的长度小于64.则扩容
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
        //如果tab数组中的hash参数计算出的位置处的节点不为null，则转为红黑树
            TreeNode<K,V> hd = null, tl = null;
            do {
                //创建一个新的TreeNode，该TreeNode继承Node
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    //对于第一个节点，会执行这里，将第一个节点赋值给hd
                    hd = p;
                else {
                    //将后续的节点链接起来，双向链表
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);

            if ((tab[index] = hd) != null)
                //将hd节点重新添加到tab中，然后转成红黑树
                hd.treeify(tab);
        }
    }
```

2. HashMap.get(Object key)
```
    public V get(Object key) {
        Node<K,V> e;
        //返回查找到的节点的value
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;

        //用通过key计算出的hash值和数组长度-1做与计算，获取table中对应的位置的节点
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {

            //如果节点不为空，则判断当前节点是不是就是我们要查询的节点，如果是，则返回
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;

             //如果不是，则循环遍历链表或从红黑树中查询对应的key
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```

3. HashMap.remove(Object key)
```
    public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }

    final Node<K,V> removeNode(int hash, Object key, Object value,
                                   boolean matchValue, boolean movable) {

        Node<K,V>[] tab; Node<K,V> p; int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;

            //1. 通过hash值，找到在数组中的位置，然后搜索该位置节点所在的链表或树，找到key值相同的节点
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }

            //2. 如果找到了对应的节点，则删除该节点
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)
                    tab[index] = node.next;
                else
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }

```