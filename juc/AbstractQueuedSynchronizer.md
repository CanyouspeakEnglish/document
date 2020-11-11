#### AbstractQueuedSynchronizer

![image-20201109174333690](C:\Users\lzx\Desktop\document\juc\image-20201109174333690.png)

#### 自定义锁

```java
@Override
protected boolean tryAcquire(int arg) {
    int state = getState();
     Thread currentThread = Thread.currentThread();
    if (state !=0){
        Thread exclusiveOwnerThread = getExclusiveOwnerThread();
        if (exclusiveOwnerThread == currentThread){
            setState(state + arg);
            return true;
        }
        return false;
    }
    if (compareAndSetState(0,arg)){
        setExclusiveOwnerThread(currentThread);
        return true;
    }
    return false;
}

@Override
protected boolean tryRelease(int arg) {
    int c = getState() - arg;
    Thread exclusiveOwnerThread = getExclusiveOwnerThread();
    Thread thread = Thread.currentThread();
    if(exclusiveOwnerThread != thread){
        throw new IllegalMonitorStateException("加锁线程不和释放锁的线程不是一个线程请检查");
    }
    boolean free = false;
    if(c == 0){
        setExclusiveOwnerThread(null);
        free = true;
    }
    setState(c);
    return free;
}
```

```java
//加锁
public final void acquire(int arg) {
    //模板方法自己实现在返回为false的时候 会往队列里放node
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        //响应中断
        selfInterrupt();
}
```

```java
//加入同步队列
private Node addWaiter(Node mode) {
    //构造结点 参数1 当前线程 2 结点类型 （共享，排他）
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    //tail 不为null 通过cas 进行构建链
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```

```java
//通过自旋创建链表
private Node enq(final Node node) {
    for (;;) {
        //为空 初始化  初始化的时候 最少自旋两次
        Node t = tail;
        if (t == null) { 
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            //结点绑定 返回结点
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

```java
//再次尝试获取锁 获取不到 进行挂起
final boolean acquireQueued(final Node node, int arg) {
    //标记是否失败
    boolean failed = true;
    try {
        //响应中断状态
        boolean interrupted = false;
        for (;;) {
            //获取当前结点的前置结点
            final Node p = node.predecessor();
            //再次尝试获取锁 在竞争不激烈的情况下有大概率获取锁
            if (p == head && tryAcquire(arg)) {
                //设置头结点为当前结点
                setHead(node);
                //帮助gc置null
                p.next = null; 
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

```java
//在挂起之前将前置结点的状态变更 如果状态是CANCELLED 状态的将从链表中移除
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    //前置结点为 SIGNAL 直接返回
    if (ws == Node.SIGNAL)
        return true;
    //取消结点
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        //将前置结点状态变更为SIGNAL
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

```java
private final boolean parkAndCheckInterrupt() {
    //挂起线程
    LockSupport.park(this);
    //返回中断状态 （当释放时从这返回）
    return Thread.interrupted();
}
```

```java
//释放锁
public final boolean release(int arg) {
    //自定义释放方法 true 释放成功
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

```java
//释放 从head开始
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        //从尾开找 应为从尾部找不出出现断链的情况 因为链的构建是分段的在极端情况下会出现 从尾部就能解决这种
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        //取消挂起
        LockSupport.unpark(s.thread);
}
```