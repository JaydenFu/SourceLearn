# LinkedList 实现原理

通过双向链表实现，内部维护size，first节点，last节点

1. LinkedList初始化
```
    //size表示容器类元素的个数
    transient int size = 0;

    //指向第一个节点
    transient Node<E> first;

    //指向最后一个节点
    transient Node<E> last;

    //Node表示一个节点
    private static class Node<E> {
        E item;//当前节点的元素
        Node<E> next;//指向下一个节点
        Node<E> prev;//指向上一个节点

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }

    //1. 默认初始化
    public LinkedList() {
    }

    //2. 用一个集合的数据初始化，内部调用addAll的方式完成数据的添加。
    public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }

```

2.  LinkedList.add(E e)
```
    public boolean add(E e) {
        linkLast(e);
        return true;
    }

    void linkLast(E e) {
        //1. 将当前最后一个节点赋值给l变量
        final Node<E> l = last;

        //2. 创建新的节点，并将l设置为当前新创建节点的前一个节点，下一个节点设置为null
        final Node<E> newNode = new Node<>(l, e, null);

        //3. 将当前节点赋值给last，换句话说就是讲last指向刚刚创建的节点，表明这个节点是当前最后一个节点
        last = newNode;

        if (l == null)
        //4. 如果添加新节点之前的最后一个节点为null，那说明当前容器是没有元素的，则将first也指向刚刚创建
        //的节点newNode，说明newNode是当前容器的第一个节点
            first = newNode;
        else
        //5. 如果添加新节点之前的最后一个节点不为null，则说明容器中原本是有元素的，则将之前最后一个节点的next指向
        //刚刚创建的新节点
            l.next = newNode;

        //6. size+1，表示元素个数+1
        size++;
        modCount++;
    }
```

3.  LinkedList.add(int index,E element)
```
    public void add(int index, E element) {

        //1. 检查index是否在0和size之间，否则抛出角标越界异常
        checkPositionIndex(index);

        //2. 如果index为size，则说明将新元素添加到最后，则逻辑和直接add是一样的
        if (index == size)
            linkLast(element);
        else
        //3. 如果index小于size,则说明需要将元素插入原来元素之间
            linkBefore(element, node(index));
    }

    //node方法获取index代表的节点
    Node<E> node(int index) {

        //1. 首先判断index是否小于当前元素总数的一半
        if (index < (size >> 1)) {
            //如果是，则从first节点开始next，通过循环，找到index代表的节点然后返回
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
        //如果index 大于当前元素个数的一半，则从last节点开始prev，通过循环，找到index的节点然后返回
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }

    //linkBefore方法将元素添加到succ节点前
    void linkBefore(E e, Node<E> succ) {

        //1. 首先将succ节点的前一个节点保存在pred中
        final Node<E> pred = succ.prev;

        //2. 创建新的节点，并将新节点的prev指向pred，将next指向succ
        final Node<E> newNode = new Node<>(pred, e, succ);

        //3. 将succ的prev指向新节点这样新节点就在succ节点的前面了。
        succ.prev = newNode;


        if (pred == null)
        //4. 如果pred节点为null，说明之前succ节点是容器中的第一个节点，所以这里需要将first指向当前newNode
        //节点，表示newNode是第一个节点
            first = newNode;
        else
        //5. 如果pred节点为不null，则说明succ是一个中间节点，那么需要将pred节点的next指向newNode
            pred.next = newNode;

        //6. 元素个数+1
        size++;
        modCount++;
    }
```

4.  LinkedList.addFirst(E element)
```
    public void addFirst(E e) {
        linkFirst(e);
    }

    private void linkFirst(E e) {
        //1. 将当前第一个节点赋值给f
        final Node<E> f = first;

        //2. 创建新的节点，并且新节点的prev指向null，next指向f
        final Node<E> newNode = new Node<>(null, e, f);

        //3. 将first指向newNode，表示newNode成为了第一个节点
        first = newNode;

        if (f == null)
        //4. 如果f为null，说明之前没有元素，这个时候newNode也是最后一个节点，因此last需要指向newNode
            last = newNode;
        else
        //5. 如果f不为null，说明之前是有元素的，这个时候需要将f节点的prev指向当前节点
            f.prev = newNode;
        size++;
        modCount++;
    }
```

5.  LinkedList.addLast(E e)
```
    public void addLast(E e) {
        linkLast(e);
    }

    void linkLast(E e) {
        //1. 将last节点赋值给l
        final Node<E> l = last;

        //2. 创建新的节点，并将新节点的prev指向l，next指向null
        final Node<E> newNode = new Node<>(l, e, null);

        //3. 将last指向newNode，表示当前节点是最后一个节点
        last = newNode;

        if (l == null)
        //4. 如果l为null，说明之前没有元素，那么需要将first节点指向newNode，表示newNode也是第一个节点
            first = newNode;
        else
        //5. 如果l不为null，说明之前有元素，那么需要将l的next指向newNode
            l.next = newNode;
        size++;
        modCount++;
    }
```

6.  LinkedList.set(int index,E e)
```
    public E set(int index, E element) {
        //1. 检查index是否合法，[0,size)
        checkElementIndex(index);
        //2. 通过index获取到对应的节点x
        Node<E> x = node(index);
        //3. 将节点的值赋值给oldVal
        E oldVal = x.item;
        //4. 将新元素赋值给x节点的item
        x.item = element;
        return oldVal;
    }
```

7.  LinkedList.remove(int index)
```
    public E remove(int index) {
        //1. 检查index是否合法
        checkElementIndex(index);
        //2. 通过node方法获取idex代表的节点
        //3. 然后通过unlink移除index代表的节点
        return unlink(node(index));
    }

    E unlink(Node<E> x) {
        final E element = x.item;
        //1. 获取x节点的下一个节点，赋值给next变量
        final Node<E> next = x.next;
        //2. 获取x节点的上一个节点，赋值给prev变量
        final Node<E> prev = x.prev;


        if (prev == null) {
        //3. 如果prev节点为null，则说明x节点是第一个节点，这时候需要要first指向next，表示next成为了第一个节点
            first = next;
        } else {
        //4. 如果prev不为null，则说明x节点不是第一个节点，那么需要将prev节点的next指向next节点，
        // x节点的prev节点指向null
            prev.next = next;
            x.prev = null;
        }

        if (next == null) {
        //5. 如果next为null，说明x节点是最后一个节点，那么需要将last指向prev，表示prev节点已经成为最后一个节点
            last = prev;
        } else {
        //6. 如果next不为null，说明x节点不是最后一个节点，那么需要将next的prev指向prev，同时x节点的next设置为null
            next.prev = prev;
            x.next = null;
        }

        //到这里x节点就从链表中断开出来了，然后将x节点的item设置为null，
        x.item = null;

        //元素个数-1
        size--;
        modCount++;
        return element;
    }
```

8.  LinkedList.get(int index)
```
    public E get(int index) {
        //1. 检查index合法性[0,size)
        checkElementIndex(index);
        //通过node方法获取index处的节点，返回它的item元素
        return node(index).item;
    }
```
