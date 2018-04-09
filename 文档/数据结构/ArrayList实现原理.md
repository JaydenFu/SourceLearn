#   ArrayList 实现原理

通过数组实现，内部维护Object[] elementData， size

1.  ArrayList初始化
```
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    transient Object[] elementData;

    //1. 默认初始化
    //用空数组对存放元素的elementData赋值
    public ArrayList() {
        //DEFAULTCAPACITY_EMPTY_ELEMENTDATA是个空数组
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    //2. 给定初始化容量初始化
    //创建给定容量大小的数据，赋值给elementData
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

    //3. 给定初始化Collection数据进行初始化
    //将Collection的值转成数组，然后通过Arrays.copyOf方式，复制数据，其内部也是通过System.arraycopy完成
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```

2.  ArrayList.add(E element)
```
    public boolean add(E e) {
        //1. 确保容器容量足够,这里size表示的当前容器内部元素的个数
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        //2. 将新元素存放进数组，然后size的值+1
        elementData[size++] = e;
        return true;
    }

    private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            //对于采用默认初始化方式而言，当第一次添加元素的时候，会执行这里
            //DEFAULT_CAPACITY系统给定的值是10.所以对于默认初始化而言，在第一次添加
            //元素的时候，会将容器改为容量为10的容器
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // 当希望的容量大于当前容量时，就增长容量
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

    private void grow(int minCapacity) {
        // 1. oldCapacity 表示的当前的容量，对于默认初始化而言，第一执行这里时，当前容量为0.
        int oldCapacity = elementData.length;

        // 2. newCapacity 表示扩大后的容量，扩大的方式是，每次增长当前容量的一半
        int newCapacity = oldCapacity + (oldCapacity >> 1);

        // 3. 如果扩容后的容量小于希望的容量，则将新容量大小设定为希望的大小，否则就采用扩容的大小
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;

        // 4.MAX_ARRAY_SIZE为Intger.Max - 8
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);

        // 5. 内部创建新的容量为newCapactity的数组，然后通过System.arraycory方式，复制数据
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

    private static int hugeCapacity(int minCapacity) {
        //1. 因为minCapacity始终的当前容量+1,所以当minCapacity为负数时，说明当前容量已经是Integer.Max了
        //不能再扩容了，因此抛出OOM
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();

        //2. 当希望的容量大于Integer.Max -8 是，则容量就设为Integer.Max，否则，就设为Integer.Max - 8
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```

3. ArrayList.add(int index,E element)
```
    public void add(int index, E element) {
        //1. 确认index没有超出上界和下界
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        //2. 确保容量足够，前面已经分析过原理
        ensureCapacityInternal(size + 1);  // Increments modCount!!

        //3. 通过System.arraycopy实现index以及之后的元素的位置移动
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);

        //4. 将新元素添加到index索引处
        elementData[index] = element;

        //5. 表示当前元素个数的size++
        size++;
    }
```

4.  ArrayList.addAll(Collection<? extends E> c)
```
    public boolean addAll(Collection<? extends E> c) {
        //1. 首先将需要添加的collection集合转成数组，内部基本也是通过创建新的数组，然后
        //通过System.arraycopy方法将数据复制到新数组，然后返回新数组
        Object[] a = c.toArray();
        int numNew = a.length;

        //2. 确保容量大小足够
        ensureCapacityInternal(size + numNew);  // Increments modCount

        //3. 再次通过System.arraycopy方法，将元素复制到elementData数组中
        System.arraycopy(a, 0, elementData, size, numNew);
        //表示当前元素个数的size加上集合c的元素个数，重新赋值给size
        size += numNew;

        //4. 如果添加的集合c元素个数不为0，则返回true说明添加成功
        return numNew != 0;
    }
```

5.  ArrayList.addAll(int index,Collection<？extends E> c)
```
    public boolean addAll(int index, Collection<? extends E> c) {
        //1. 确认index是否越界
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        //2. 将想要添加的集合c转成数组
        Object[] a = c.toArray();
        int numNew = a.length;

        //3. 确保容器容量足够
        ensureCapacityInternal(size + numNew);  // Increments modCount

        //4. 计算需要移动的元素个数，当需要移动的元素个数大于0时，通过System.arraycopy
        //复制元素实现元素位置的移动
        int numMoved = size - index;
        if (numMoved > 0)
            System.arraycopy(elementData, index, elementData, index + numNew,
                             numMoved);

        //5. 再次通过System.arraycopy将a中的元素复制到elementData中
        System.arraycopy(a, 0, elementData, index, numNew);
        size += numNew;
        return numNew != 0;
    }
```

6.  ArrayList.get(int index)
```
    public E get(int index) {
        //1. 确认是否越界
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        //2. 直接返回数组中index位置的元素
        return (E) elementData[index];
    }

```

7.  ArrayList.contains(Object o)
```
    public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }

    public int indexOf(Object o) {
        //1. 如果需要查找为值的Object为null，则采用==判断
        if (o == null) {

            //从0开始，循环遍历elementData数组，每个元素与目标元素对比
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
        //2. 否则，采用equals判断
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
```

8.  ArrayList.remove(int index)
```
    public E remove(int index) {
        //1. 首先确认index是否越界
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        modCount++;

        //2. 取出当前index上的元素
        E oldValue = (E) elementData[index];

        //3. 计算出要移动的元素个数
        int numMoved = size - index - 1;

        //4. 如果要移动的元素个数大于0，则通过System.arraycopy实现数组内元素的移动
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);

        //5. 因为元素已经移动过了，所以将最后一个元素的位置设置为null
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
```

9.  ArrayList.remove(Object o)
```
    public boolean remove(Object o) {
        //通过循环遍历元素，找到元素对应的index，然后通过index删除元素
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }

    //fastRemove和普通的remove对比，没有index越界检查，也没有取出index位置的元素，并返回
    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }
```

10. ArrayList.set(int index, E element)
```
    public E set(int index, E element) {
        //1. 确认index是否越界
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        //2. 取出当前index位置的元素
        E oldValue = (E) elementData[index];

        //3. 将新元素放进elememntData数组中的index位置
        elementData[index] = element;

        //4. 返回原来的元素
        return oldValue;
    }
```