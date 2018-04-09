#   ArrayMap实现原理

ArrayMap实现了Map接口，是Android中用来代替HashMap的，相比HashMap更节约内存，
因为只有当当前容量满了，才1.5倍扩容，并且当移除元素时，还会缩小容量，但是相对来说时间效率就高了点，因为每次
插入、删除，查找都需要通过二分查找index位置，然后才能执行相应的操作。所以ArrayMap适合在数据量不是特别大的时候
代替HashMap

1. ArrayMap初始化
```

    int[] mHashes;
    Object[] mArray;
    int mSize;


    //1. 默认初始化
    public ArrayMap() {
        //默认初始化capacity为0,identityHashCode为flse
        this(0, false);
    }

    //2. 带初始容量初始化
    public ArrayMap(int capacity) {
        this(capacity, false);
    }

    //3. 带初始容量，和identityHashCode初始化
    public ArrayMap(int capacity, boolean identityHashCode) {

        //1. 将identityHashCode赋值给mIdentityHashCode
        mIdentityHashCode = identityHashCode;

        if (capacity < 0) {
            //2. 如果capacity设置为小于0，则将mHashes设置为EMPTY_IMMUTABLE_INTS
            //static final int[] EMPTY_IMMUTABLE_INTS = new int[0];
            mHashes = EMPTY_IMMUTABLE_INTS;
            //EmptyArray.OBJECT = new Object[0];
            mArray = EmptyArray.OBJECT;
        } else if (capacity == 0) {

            //3. 如果capacity设置==0，则将mHash设置为new in[0]，mArray设置为new Object[0];
            mHashes = EmptyArray.INT;
            mArray = EmptyArray.OBJECT;
        } else {

            //capacity为其他大于0的值，通过allocArrays创建对应的容量
            allocArrays(capacity);
        }

        //当前容器中元素个数mSize赋值为0
        mSize = 0;
    }
```

2.  ArrayMap.allocArrays(int capacity)
```
 private static final int BASE_SIZE = 4;

 private void allocArrays(final int size) {
        if (mHashes == EMPTY_IMMUTABLE_INTS) {
            throw new UnsupportedOperationException("ArrayMap is immutable");
        }
        //如果size个数为8
        if (size == (BASE_SIZE*2)) {
            synchronized (ArrayMap.class) {
                if (mTwiceBaseCache != null) {
                    final Object[] array = mTwiceBaseCache;
                    //采用缓存对mArray赋值
                    mArray = array;
                    mTwiceBaseCache = (Object[])array[0];
                    //采用缓存对
                    mHashes = (int[])array[1];
                    array[0] = array[1] = null;
                    mTwiceBaseCacheSize--;
                    return;
                }
            }
        } else if (size == BASE_SIZE) {
            //如果size个数为4
            synchronized (ArrayMap.class) {
                if (mBaseCache != null) {
                    final Object[] array = mBaseCache;
                    mArray = array;
                    mBaseCache = (Object[])array[0];
                    mHashes = (int[])array[1];
                    array[0] = array[1] = null;
                    mBaseCacheSize--;
                    return;
                }
            }
        }

        //创建对应size的mHashes数组，2倍size的mArray数组
        mHashes = new int[size];
        mArray = new Object[size<<1];
    }
```

3. ArrayMap.put(K key,V value)
```
    public V put(K key, V value) {
        final int osize = mSize;
        final int hash;
        int index;
        if (key == null) {
            hash = 0;
            index = indexOfNull();
        } else {
            hash = mIdentityHashCode ? System.identityHashCode(key) : key.hashCode();

            //通过key和key的hashCode查找在数组中的角标
            index = indexOf(key, hash);
        }

        //如果index>0,则说明已经存在相同key了，那么需要替换到key所对应的值
        if (index >= 0) {
            //将index乘以2，然后+1，这个位置存放的就是value，而言index乘以2这个位置存放的是index
            index = (index<<1) + 1;
            final V old = (V)mArray[index];
            mArray[index] = value;
            return old;
        }

        //执行到这里，说明index<0,
        //对index取反，此时index表示的是新元素的key的hash要插入到mHashes中的位置
        index = ~index;

        //如果osize大于等于mHashes.length,则说明mHashes数组已经满了，需要扩容
        if (osize >= mHashes.length) {
            //如果osize>=8,则扩容为当前容量+当前容量的一半，也就是扩容后的容量为之前的1.5倍
            //如果osize<8&&osize>=4,则容量扩展为8,否则容量扩展为4
            //因此对于默认初始化而言，在第一次添加元素后，容量扩充为4
            final int n = osize >= (BASE_SIZE*2) ? (osize+(osize>>1))
                    : (osize >= BASE_SIZE ? (BASE_SIZE*2) : BASE_SIZE);


            final int[] ohashes = mHashes;
            final Object[] oarray = mArray;

            //通过allocArrays方法扩充mHashes和mArray数组的大小
            allocArrays(n);

            if (CONCURRENT_MODIFICATION_EXCEPTIONS && osize != mSize) {
                throw new ConcurrentModificationException();
            }

            if (mHashes.length > 0) {
                //将扩容前的数据复制到扩容后的数组中
                System.arraycopy(ohashes, 0, mHashes, 0, ohashes.length);
                System.arraycopy(oarray, 0, mArray, 0, oarray.length);
            }

            //释放原来的数组
            freeArrays(ohashes, oarray, osize);
        }

        //如果index小于osize，表示，新元素需要插入到index位置
        if (index < osize) {
            //因此，需要先将mHashes数组中的index位置及后面位置的元素向后移动1位
            System.arraycopy(mHashes, index, mHashes, index + 1, osize - index);
            //将mArray数组中的index<<2位置及后面的元素向后移动2位
            System.arraycopy(mArray, index << 1, mArray, (index + 1) << 1, (mSize - index) << 1);
        }

        if (CONCURRENT_MODIFICATION_EXCEPTIONS) {
            if (osize != mSize || index >= mHashes.length) {
                throw new ConcurrentModificationException();
            }
        }

        //然后将hash插入到mHashes中的index位置
        mHashes[index] = hash;
        //将key插入mArray中的index<<1位置
        mArray[index<<1] = key;
        //将value插入到mArray中的index<<1+1的位置
        mArray[(index<<1)+1] = value;

        //表示元素个数的mSize+1
        mSize++;
        return null;
    }
```

4. ArrayMap.indexOf()
```
   int indexOf(Object key, int hash) {
        final int N = mSize;

        //如果当前元素个数为0，则直接返回~0，也就是0xFFFFFFFF
        if (N == 0) {
            return ~0;
        }

        //在mHashed中二分查找对应的hash值，
        int index = binarySearchHashes(mHashes, N, hash);

        //如果index小于0，说明当前mHashes没有找到hash所代表的值，直接返回index
        if (index < 0) {
            return index;
        }

        //如果index>0,并且key值和mArray[index<<1]相同，则说明找到了我们对应的key位置，返回index
        if (key.equals(mArray[index<<1])) {
            return index;
        }

        int end;
        //如果index>0,但是mArray[index<<1]的key和我们插入的key不同，则表示只是key的hash值相同了，
        //这个时候需要，从index+1开始遍历，在mHashed中查找相同hash位置，然后看对应的key是否相同，如果相同，则返回对应的位置end
        for (end = index + 1; end < N && mHashes[end] == hash; end++) {
            if (key.equals(mArray[end << 1])) return end;
        }

        //如果向后查找没找到，则向前查找，找到了就返回对应的位置i
        for (int i = index - 1; i >= 0 && mHashes[i] == hash; i--) {
            if (key.equals(mArray[i << 1])) return i;
        }

        //执行到这里说明没找到相同的key，则返回end的反码，这个时候end表示的是相同hash的在mHashes中后一个位置，
        return ~end;
    }
```

5.  ArrayMap.binarySearchHashes
```
    private static int binarySearchHashes(int[] hashes, int N, int hash) {
        try {
            return ContainerHelpers.binarySearch(hashes, N, hash);
        } catch (ArrayIndexOutOfBoundsException e) {
            if (CONCURRENT_MODIFICATION_EXCEPTIONS) {
                throw new ConcurrentModificationException();
            } else {
                throw e; // the cache is poisoned at this point, there's not much we can do
            }
        }
    }

    static int binarySearch(int[] array, int size, int value) {
        int lo = 0;
        int hi = size - 1;

        while (lo <= hi) {
            //通过二分查找，如果找到了在array中找到了value对应的值，则返回角标
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

        //如果没有找到，则返回array中大于value值得最小值所在位置的反码
        return ~lo;  // value not present
    }
```

6.  ArrayMap.freeArrays
```
 private static void freeArrays(final int[] hashes, final Object[] array, final int size) {

        //如果hashed.length为8
        if (hashes.length == (BASE_SIZE*2)) {
            synchronized (ArrayMap.class) {
                //当mTwiceBaseCacheSize小于10时，
                if (mTwiceBaseCacheSize < CACHE_SIZE) {
                    array[0] = mTwiceBaseCache;
                    array[1] = hashes;
                    for (int i=(size<<1)-1; i>=2; i--) {
                        //清空array中除了0，1位置的其他所有位置
                        array[i] = null;
                    }

                    //将array赋值给mTwiceBaseCache
                    mTwiceBaseCache = array;
                    //mTwiceBaseCacheSize+1
                    mTwiceBaseCacheSize++;
                }
            }
        } else if (hashes.length == BASE_SIZE) {
            synchronized (ArrayMap.class) {
                //和上面的逻辑差不多
                if (mBaseCacheSize < CACHE_SIZE) {
                    array[0] = mBaseCache;
                    array[1] = hashes;
                    for (int i=(size<<1)-1; i>=2; i--) {
                        array[i] = null;
                    }
                    mBaseCache = array;
                    mBaseCacheSize++;
                }
            }
        }
    }
```

7.  ArrayMap.get
```
    public V get(Object key) {
        final int index = indexOfKey(key);
        return index >= 0 ? (V)mArray[(index<<1)+1] : null;
    }

    //indexOfkey,最终调用的还是上面已经分析过的indexOf方法，
    public int indexOfKey(Object key) {
        return key == null ? indexOfNull()
                : indexOf(key, mIdentityHashCode ? System.identityHashCode(key) : key.hashCode());
    }
```

8.  ArrayMap.remove
```
    public V remove(Object key) {
        //计算出位置，然后执行removeAt方法
        final int index = indexOfKey(key);
        if (index >= 0) {
            return removeAt(index);
        }

        return null;
    }

     public V removeAt(int index) {
            final Object old = mArray[(index << 1) + 1];
            final int osize = mSize;
            final int nsize;
            if (osize <= 1) {
                freeArrays(mHashes, mArray, osize);
                mHashes = EmptyArray.INT;
                mArray = EmptyArray.OBJECT;
                nsize = 0;
            } else {
                nsize = osize - 1;

                //当mHashes.length 大于8，并且移除元素之前元素总数小于mHashes.length的3分之1时
                if (mHashes.length > (BASE_SIZE*2) && mSize < mHashes.length/3) {

                    //如果元素个数大于8,则n社会为osize的1.5倍，如果元素个数小于8，则n设置为8
                    final int n = osize > (BASE_SIZE*2) ? (osize + (osize>>1)) : (BASE_SIZE*2);

                    final int[] ohashes = mHashes;
                    final Object[] oarray = mArray;

                    //重新调整oHahes和oarray数组的大小
                    allocArrays(n);

                    if (CONCURRENT_MODIFICATION_EXCEPTIONS && osize != mSize) {
                        throw new ConcurrentModificationException();
                    }

                    //重新赋值数组
                    if (index > 0) {
                        System.arraycopy(ohashes, 0, mHashes, 0, index);
                        System.arraycopy(oarray, 0, mArray, 0, index << 1);
                    }

                    //向前移动数组中的元素完成删除操作
                    if (index < nsize) {
                        System.arraycopy(ohashes, index + 1, mHashes, index, nsize - index);
                        System.arraycopy(oarray, (index + 1) << 1, mArray, index << 1,
                                (nsize - index) << 1);
                    }
                } else {
                    if (index < nsize) {
                        System.arraycopy(mHashes, index + 1, mHashes, index, nsize - index);
                        System.arraycopy(mArray, (index + 1) << 1, mArray, index << 1,
                                (nsize - index) << 1);
                    }
                    mArray[nsize << 1] = null;
                    mArray[(nsize << 1) + 1] = null;
                }
            }
            if (CONCURRENT_MODIFICATION_EXCEPTIONS && osize != mSize) {
                throw new ConcurrentModificationException();
            }
            mSize = nsize;
            return (V)old;
        }
```
