#   SparseArray实现原理

1. SparseArray初始化
```
    private int[] mKeys;
    private Object[] mValues;
    private int mSize;

    //1. 默认初始化
    public SparseArray() {
        //给定初始化容量10
        this(10);
    }

    //2. 给定初始容量初始化，但实际创建的容量不一定就是给定的容量，一般会比给定的容量大
    public SparseArray(int initialCapacity) {
        if (initialCapacity == 0) {
            mKeys = EmptyArray.INT;
            mValues = EmptyArray.OBJECT;
        } else {

            //创建mValues，mKeys数组
            mValues = ArrayUtils.newUnpaddedObjectArray(initialCapacity);
            mKeys = new int[mValues.length];
        }
        mSize = 0;
    }
```

2. SparseArray.put(int key,E value)
```
 public void put(int key, E value) {
        //在mKeys中进行二分查找
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        //如果i>0,说明找到了对应的key，那么将mValues中i位置的值替换成value，则put过程完成
        if (i >= 0) {
            mValues[i] = value;
        } else {

            //如果i<0,则说明key不在，这个时候取反i，该位置为新值需要插入的位置
            i = ~i;

            //如果i小于mSize，并且mValues[i] == DELETED,则说明当前i位置的值被删除了，也就是说可以将值
            //直接存在i位置，然后返回
            if (i < mSize && mValues[i] == DELETED) {
                mKeys[i] = key;
                mValues[i] = value;
                return;
            }

            //执行到这里，说明i等于mSize或则mValues[i]!=DELETED.

            //mGarbage默认为false，当删除元素后mGarbase会被赋值为true，在gc方法中被赋值为false
            if (mGarbage && mSize >= mKeys.length) {
                gc();

                // 重新执行二分查找，找出需要插入的位置
                i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
            }

            //增大数组，并将值插入数组
            mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
            mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);

            //表示元素个数的mSize+1
            mSize++;
        }
    }

    //gc方法，释放已删除元素
    private void gc() {

        int n = mSize;
        int o = 0;
        int[] keys = mKeys;
        Object[] values = mValues;

        //通过for循环遍历values列表，如果其中有val值为DELETED,则需要移除该位置的key，以及value
        for (int i = 0; i < n; i++) {
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
    }

```

3.  SparseArray.get(int key)
```
    public E get(int key) {
        return get(key, null);
    }

    public E get(int key, E valueIfKeyNotFound) {
        //在mKeys中二分查找key，如果返回值i大于0，则说明找到对应的key了，否则没找到
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        if (i < 0 || mValues[i] == DELETED) {
            return valueIfKeyNotFound;
        } else {
            //找到对应的key了，直接返回mValues数组中i位置的值
            return (E) mValues[i];
        }
    }
```

4. SparseArray.remove(int key)
```
    //在remove过程中并没有去改变mSize大小，说明这个时候其实并没有真正的从数组中移除，只是将其标记为删除
    //只有在gc方法中，才会对标记删除的元素执行真正的移除，并改变mSize
    public void remove(int key) {
        delete(key);
    }

    public void delete(int key) {
        // 二分查找位置
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);
        //如果i大于0，说明找打了对应的key值，这个时候，直接将mValues数组中位置的元素设置成DELETED,然后将
        //mGarbage设置为true
        if (i >= 0) {
            if (mValues[i] != DELETED) {
                mValues[i] = DELETED;
                mGarbage = true;
            }
        }
    }
```