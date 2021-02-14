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
                //如果该链的长度超过 TREEIFY_THRESHOLD 8 转为红黑树有两个条件 长度大于64 并且链表长度超过>=8
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
    //死循环
    for (TreeNode<K,V> p = root;;) {
        int dir, ph; K pk;
        //如果为null 创建root结点
        if (p == null) {
            first = root = new TreeNode<K,V>(h, k, v, null, null);
            break;
        }
        //如果新增的结点的hash比 root结点的hash小 那么dir设置为-1
        else if ((ph = p.hash) > h)
            dir = -1;
        else if (ph < h)//比root结点的hash大那么dir设置为1 
            dir = 1;
        //key 值相等直接返回 p
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
    //如果大小为MAXIMUM_CAPACITY最大总量的一半，那么直接扩容为MAXIMUM_CAPACITY，否则计算最小幂次方
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
        tableSizeFor(size + (size >>> 1) + 1);//计算2的幂次方
    int sc;
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;
        //如果table还未进行初始化
        if (tab == null || (n = tab.length) == 0) {
            n = (sc > c) ? sc : c;
            //cas修改sizeCtl为-1，表示table正在进行初始化
            if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        //重新赋值sc
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
            }
        }
        //如果扩容大小没有达到阈值，或者超过最大容量
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        else if (tab == table) {
            //生成表的生成戳，每个n都有不同的生成戳
            int rs = resizeStamp(n);
            if (sc < 0) {
                Node<K,V>[] nt;
                /**1.第一个判断 sc右移RESIZE_STAMP_SHIFT位，也就是比较高ESIZE_STAMP_BITS位生成戳和rs是否相等
                    * 相等则代表是同一个n，是在同一容量下进行的扩容，
                    *  2.第二个和第三个判断 判断当前帮助扩容线程数是否已达到MAX_RESIZERS最大扩容线程数
                    *  3.第四个和第五个判断 为了确保transfer()方法初始化完毕
                    */
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                //将扩容线程加一
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    //开始扩容
                    transfer(tab, nt);
            } 
            /**如果没有线程在进行扩容，那么cas修改sizeCtl值，作为扩容的发起，rs左移RESIZE_STAMP_SHIFT位+2
                 * 上面说了，左移RESIZE_STAMP_SHIFT位，肯定是个负数，代表有一个线程正在进行扩容
                 * 此时sizeCtl高RESIZE_STAMP_BITS位为生成戳，低RESIZE_STAMP_SHIFT位为扩容线程数
                   添加线程数
                 */
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                //第一次扩容nexttable=null
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

```java
//开始真正的转移逻辑
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    //表长度  stride 表示每个线程可以转移的数量
    int n = tab.length, stride;
    //Runtime.getRuntime().availableProcessors(); 获取cpu的数量
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    //新的table为空 
    if (nextTab == null) {            // initiating
        try {
            //重新创建一个数组长度为原来的两倍
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            //将nt赋值给nextTab
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        //值绑定
        nextTable = nextTab;
        //转移的索引
        transferIndex = n;
    }
    //扩容后的数组长度
    int nextn = nextTab.length;
    //将扩容后的node数组用ForwardingNode标记
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        while (advance) {
            int nextIndex, nextBound;
            //如果开始边界大于结束边界 或者 已经完成 跳出循环
            if (--i >= bound || finishing)
                advance = false;
            //如果转移的索引小于等于0 结束
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            //设置初始边界 0  
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            //总扩容完成
            if (finishing) {
                //完成后将nextTable 置为空
                nextTable = null;
                //将数据 之前的next 和 全局变量绑定
                table = nextTab;
                //更新sizeCtl 扩容预期值  以16 为例 sizeCtl = 24
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            //尝试将sc-1 表示这个线程结束帮助扩容
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                //如果sc-2 和 唯一标识的前16位不相等 那么说明已经有线程帮助扩容了 直接返回
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                //完成跳出循环
                finishing = advance = true;
                //再次检查整个table
                i = n;
            }
        }
        //获取老数组下标i的位置 如果为null 那么将ForwardingNode 进行标识
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // f.hash = MOVED -1 那么说明已经处理过 ForwardingNode 在初始化的时候将hash处理为-1
        else if ((fh = f.hash) == MOVED)
            //下移
            advance = true; 
        else {
            //到这里说明这个位置有实际值了且不是占位符对这个节点上锁为什么上锁防止putVal的时候向链表插入数据
            synchronized (f) {
                //双重检查
                if (tabAt(tab, i) == f) {
                    //低位链表 高位链表
                    Node<K,V> ln, hn;
                    //如果f的hash值大于0TreeBin的hash是-2
                    if (fh >= 0) {
                        //hash值 和 长度做& 运算 为了判断 高位还是低位
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        //通过循环找到最后一个 node  并将 p赋值给 lastRun
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        //runBit = 0 那么将lastRun赋值给ln 低链表 高位链表为null
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        //再次循环将链表进行拆分 分为高低链
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        //将低位链表替换在原来i的位置
                        setTabAt(nextTab, i, ln);
                        //将高位链表替换为i+n的位置
                        setTabAt(nextTab, i + n, hn);
                        //i位置用fwd 进行替换标记
                        setTabAt(tab, i, fwd);
                        //索引下推
                        advance = true;
                    }
                    //如果位置是红黑树
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        //遍历
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            //如果&等于0 那么放在低位
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            //不是0放到高位
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        //判断链表长度是否小于等于6 变成链表 否则变成红黑树
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                         // 低位树
                        setTabAt(nextTab, i, ln);
                        // 高位树
                        setTabAt(nextTab, i + n, hn);
                        //设置为fwd标记
                        setTabAt(tab, i, fwd);
                        //往下推
                        advance = true;
                    }
                }
            }
        }
    }
}
```

```java
//添加数据 条数 采用分段计数优化
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    //counterCells 不为空 或者 BASECOUNT  cas失败的情况下
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            //ThreadLocalRandom.getProbe() & m 使用线程安全的随机 确定数字下标
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            //数值增加失败
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            //进行添加
            fullAddCount(x, uncontended);
            return;
        }
        //因为bincount数量小于1 所以不需要进行扩容 直接返回
        if (check <= 1)
            return;
        s = sumCount();
    }
    // 用于检查链表的长度 
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            if (sc < 0) {
                //标识位不相等 或者 sc == rs+1 说明已经有一个线程在扩容
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    //或者已经等于最大的扩容线程
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                //设置添加扩容线程 
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    //开始转移
                    transfer(tab, nt);
            }//大于等于0 开始扩容 
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```

```java
private final void fullAddCount(long x, boolean wasUncontended) {
    int h;
    //判断 ThreadLocalRandom 是否初始化 没有初始化为0
    if ((h = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();      // force initialization
        h = ThreadLocalRandom.getProbe();
        wasUncontended = true;
    }
    boolean collide = false;                // True if last slot nonempty
    //死循环
    for (;;) {
        CounterCell[] as; CounterCell a; int n; long v;
        //counterCells 不为空 并且长度大于0
        if ((as = counterCells) != null && (n = as.length) > 0) {
            //通过撞针 & 数字长度-1 来定位数据位置 == null
            if ((a = as[(n - 1) & h]) == null) {
                //判断counterCells数组是否在初始化过程中 0 不是初始化过程中
                if (cellsBusy == 0) {            // Try to attach new Cell
                    //创建新的CounterCell
                    CounterCell r = new CounterCell(x); // Optimistic create
                    //将CELLSBUSY 从0设置成1 
                    if (cellsBusy == 0 &&
                        U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                        boolean created = false;
                        try {               // Recheck under lock
                            CounterCell[] rs; int m, j;
                            if ((rs = counterCells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                //将新创建的 CounterCell赋值给数组
                                rs[j] = r;
                                //标记为创建完成
                                created = true;
                            }
                        } finally {
                            //cellsBusy 重置为0
                            cellsBusy = 0;
                        }
                        //停止
                        if (created)
                            break;
                        continue;           // Slot is now non-empty
                    }
                }
                collide = false;
            }
            else if (!wasUncontended)       // CAS already known to fail
                wasUncontended = true;      // Continue after rehash
            else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                break;
            //如果数组发生变化 或者 n大于 cpu的数量
            else if (counterCells != as || n >= NCPU)
                collide = false;            // At max size or stale
            else if (!collide)
                collide = true;
            //发生扩容 cas成功
            else if (cellsBusy == 0 &&
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                try {
                    if (counterCells == as) {// Expand table unless stale
                        //扩容为原来的2倍
                        CounterCell[] rs = new CounterCell[n << 1];
                        for (int i = 0; i < n; ++i)
                            //进行数组迁移
                            rs[i] = as[i];
                        //重新赋值
                        counterCells = rs;
                    }
                } finally {
                    //设置为0
                    cellsBusy = 0;
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            //发生变化重新生成撞针
            h = ThreadLocalRandom.advanceProbe(h);
        }
        //进行初始化 默认的初始化长度为2
        else if (cellsBusy == 0 && counterCells == as &&
                 U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
            boolean init = false;
            try {                           // Initialize table
                if (counterCells == as) {
                    CounterCell[] rs = new CounterCell[2];
                    rs[h & 1] = new CounterCell(x);
                    counterCells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;
            }
            if (init)
                break;
        }
        //设置成功
        else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
            break;                          // Fall back on using base
    }
}
```

```java
//进行计数 用基础basecount + counterCell.value
final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

```java
//通过key获取值  containsKey 也是调用这个方法
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    //通过散列函数处理
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        //定位到曹
        (e = tabAt(tab, (n - 1) & h)) != null) {
        //hash值相等 
        if ((eh = e.hash) == h) {
            //并且key 相等
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

```java
//如果预期的key不存在那么执行 mappingFunction 因为是方法 所以内部是线程安全的 可以处理一些特定的初始化操作
public V computeIfAbsent(K key, Function<? super K, ? extends V> mappingFunction) {
    if (key == null || mappingFunction == null)
        throw new NullPointerException();
    int h = spread(key.hashCode());
    V val = null;
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & h)) == null) {
            Node<K,V> r = new ReservationNode<K,V>();
            synchronized (r) {
                if (casTabAt(tab, i, null, r)) {
                    binCount = 1;
                    Node<K,V> node = null;
                    try {
                        if ((val = mappingFunction.apply(key)) != null)
                            node = new Node<K,V>(h, key, val, null);
                    } finally {
                        setTabAt(tab, i, node);
                    }
                }
            }
            if (binCount != 0)
                break;
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            boolean added = false;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek; V ev;
                            if (e.hash == h &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                val = e.val;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                if ((val = mappingFunction.apply(key)) != null) {
                                    added = true;
                                    pred.next = new Node<K,V>(h, key, val, null);
                                }
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        binCount = 2;
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> r, p;
                        if ((r = t.root) != null &&
                            (p = r.findTreeNode(h, key, null)) != null)
                            val = p.val;
                        else if ((val = mappingFunction.apply(key)) != null) {
                            added = true;
                            t.putTreeVal(h, key, val);
                        }
                    }
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (!added)
                    return val;
                break;
            }
        }
    }
    if (val != null)
        addCount(1L, binCount);
    return val;
}
```

```java
//如果key存在 那么会将原来的值传入remappingFunction 中执行特定的方法在处理
public V computeIfPresent(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
    if (key == null || remappingFunction == null)
        throw new NullPointerException();
    int h = spread(key.hashCode());
    V val = null;
    int delta = 0;
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & h)) == null)
            break;
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f, pred = null;; ++binCount) {
                            K ek;
                            if (e.hash == h &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                val = remappingFunction.apply(key, e.val);
                                if (val != null)
                                    e.val = val;
                                else {
                                    delta = -1;
                                    Node<K,V> en = e.next;
                                    if (pred != null)
                                        pred.next = en;
                                    else
                                        setTabAt(tab, i, en);
                                }
                                break;
                            }
                            pred = e;
                            if ((e = e.next) == null)
                                break;
                        }
                    }
                    else if (f instanceof TreeBin) {
                        binCount = 2;
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> r, p;
                        if ((r = t.root) != null &&
                            (p = r.findTreeNode(h, key, null)) != null) {
                            val = remappingFunction.apply(key, p.val);
                            if (val != null)
                                p.val = val;
                            else {
                                delta = -1;
                                if (t.removeTreeNode(p))
                                    setTabAt(tab, i, untreeify(t.first));
                            }
                        }
                    }
                }
            }
            if (binCount != 0)
                break;
        }
    }
    if (delta != 0)
        addCount((long)delta, binCount);
    return val;
}
```

```java
//remove 删除方法和 get方法差不多 先定位数据 然后将节点释放 最后将数量-1
final V replaceNode(Object key, V value, Object cv) {
    int hash = spread(key.hashCode());
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0 ||
            (f = tabAt(tab, i = (n - 1) & hash)) == null)
            break;
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            boolean validated = false;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        validated = true;
                        for (Node<K,V> e = f, pred = null;;) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                V ev = e.val;
                                if (cv == null || cv == ev ||
                                    (ev != null && cv.equals(ev))) {
                                    oldVal = ev;
                                    if (value != null)
                                        e.val = value;
                                    else if (pred != null)
                                        pred.next = e.next;
                                    else
                                        setTabAt(tab, i, e.next);
                                }
                                break;
                            }
                            pred = e;
                            if ((e = e.next) == null)
                                break;
                        }
                    }
                    else if (f instanceof TreeBin) {
                        validated = true;
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> r, p;
                        if ((r = t.root) != null &&
                            (p = r.findTreeNode(hash, key, null)) != null) {
                            V pv = p.val;
                            if (cv == null || cv == pv ||
                                (pv != null && cv.equals(pv))) {
                                oldVal = pv;
                                if (value != null)
                                    p.val = value;
                                else if (t.removeTreeNode(p))
                                    setTabAt(tab, i, untreeify(t.first));
                            }
                        }
                    }
                }
            }
            if (validated) {
                if (oldVal != null) {
                    if (value == null)
                        addCount(-1L, -1);
                    return oldVal;
                }
                break;
            }
        }
    }
    return null;
}
```