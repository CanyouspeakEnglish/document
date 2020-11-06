#### 										ConcurrentHashMap

##### 结构图

![image-20201104151733115](C:\Users\lzx\AppData\Roaming\Typora\typora-user-images\image-20201104151733115.png)

![image-20201104154325106](C:\Users\lzx\Desktop\document\juc\image-20201104154325106.png)

##### put(key,value)

```java
public V put(K key, V value) {
    //调用putVal
    return putVal(key, value, false);
}
```

```java
//放值 key value 存在是否覆盖
final V putVal(K key, V value, boolean onlyIfAbsent) {
    //key 为空 或者 value 为空 抛出空指针
    if (key == null || value == null) throw new NullPointerException();
    //散列函数
    int hash = spread(key.hashCode());
    int binCount = 0;
    //自旋 
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        //如果是空 或者 长度为0 
        if (tab == null || (n = tab.length) == 0)
            //进行初始化
            tab = initTable();
        //通过长度&上散列过得hash值 判断是否为空 为空说明不存在hash碰撞
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            //通过cas 进行赋值 不需要加锁
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                 
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            //存在竞争 需要加锁
            V oldVal = null;
            //f 为当前位置上的 node 将当前位置
            synchronized (f) {
                //双锁检查 判断是否
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        //判断是否需要转红黑树的长度
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            //hash 相同 key 相同 允许覆盖 false
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                //跳出
                                break;
                            }
                            Node<K,V> pred = e;
                            //为空 将链表构建
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    //如果为红黑树
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        //向放入红黑树
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            //判断是否需要转换
            if (binCount != 0) {
                //如果该链的长度超过 TREEIFY_THRESHOLD 8
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

```java
//初始化 数组
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    //自循环
    while ((tab = table) == null || tab.length == 0) {
        //如果 sizeCtl < 0 说明并发初始化 
        if ((sc = sizeCtl) < 0)
            //让出cpu
            Thread.yield(); // lost initialization race; just spin
        //首先cas 将sizeCtl 变成-1
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                //再次判断
                if ((tab = table) == null || tab.length == 0) {
                    //第一次 不大于零 取默认值 16
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    // new 一个 node集合 
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    //赋值给全局变量table
                    table = tab = nt;
                    //sc = 12 
                    sc = n - (n >>> 2);
                }
            } finally {
                // 将sc 赋值给sizeCtl
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

```java
//赋值红黑树
final TreeNode<K,V> putTreeVal(int h, K k, V v) {
    Class<?> kc = null;
    boolean searched = false;
    for (TreeNode<K,V> p = root;;) {
        int dir, ph; K pk;
        if (p == null) {
            first = root = new TreeNode<K,V>(h, k, v, null, null);
            break;
        }
        else if ((ph = p.hash) > h)
            dir = -1;
        else if (ph < h)
            dir = 1;
        else if ((pk = p.key) == k || (pk != null && k.equals(pk)))
            return p;
        else if ((kc == null &&
                  (kc = comparableClassFor(k)) == null) ||
                 (dir = compareComparables(kc, k, pk)) == 0) {
            if (!searched) {
                TreeNode<K,V> q, ch;
                searched = true;
                if (((ch = p.left) != null &&
                     (q = ch.findTreeNode(h, k, kc)) != null) ||
                    ((ch = p.right) != null &&
                     (q = ch.findTreeNode(h, k, kc)) != null))
                    return q;
            }
            dir = tieBreakOrder(k, pk);
        }

        TreeNode<K,V> xp = p;
        if ((p = (dir <= 0) ? p.left : p.right) == null) {
            TreeNode<K,V> x, f = first;
            first = x = new TreeNode<K,V>(h, k, v, f, xp);
            if (f != null)
                f.prev = x;
            if (dir <= 0)
                xp.left = x;
            else
                xp.right = x;
            if (!xp.red)
                x.red = true;
            else {
                lockRoot();
                try {
                    root = balanceInsertion(root, x);
                } finally {
                    unlockRoot();
                }
            }
            break;
        }
    }
    assert checkInvariants(root);
    return null;
}
```

```java
//链表转换成红黑树
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    //数组不为空
    if (tab != null) {
        //首先判断长度是否小于64 优先扩容
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            tryPresize(n << 1);
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```

```java
//扩容
private final void tryPresize(int size) {
    //判断边缘值
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
    	//
        tableSizeFor(size + (size >>> 1) + 1);
    int sc;
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;
        if (tab == null || (n = tab.length) == 0) {
            n = (sc > c) ? sc : c;
            if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
            }
        }
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        else if (tab == table) {
            int rs = resizeStamp(n);
            if (sc < 0) {
                Node<K,V>[] nt;
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
        }
    }
}
```

```java
//帮助转移 在检测f.hash = move 的时候
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    //tab 不为空 并且 f结点已经被包装为ForwardingNode 并且下一个node集合不为空
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        //唯一标识 前16位标识状态 后16位标识具体正在扩容的线程数量
        int rs = resizeStamp(tab.length);
        // 没有发生并发修改 sizeCtl < 0说明还在扩容
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            //如果sizeCtl 右移16位 和 唯一标识的前16位不相等 则标识位发生了变化
            //如果sizeCtl = rs + 1 说明线程已经扩容完成 因为在发生扩容的时候 是sizeCtl + 2 
            //如果sizeCtl = rs + 65535 说明达到了 最大的帮助线程也需要退出
            //如果转移下标发生了变化 说明扩容正在结束 需要退出
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            //上面的跳出条件全部都不满足那么开始用cas 帮助扩容 首先线程数加1 
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                //开始扩容
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

```java
//高位为1 一定为负
static final int resizeStamp(int n) {
     	// 27 = 0000 0000 0001 1011
     	// 0000 0000 0001 1011 | 1000 0000 0000 0000 = 1000 0000 0001 1011
     	// 以16位例  0000 0000 0000 0000 0000 0000 0001 0000
     	//Integer.numberOfLeadingZeros(n) = 27 表示第一个不为0前面0的个数 
     	//RESIZE_STAMP_BITS = 16  1左移15位
     	//1000 0000 0001 1011
        return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
    }
```