## ReentrantLock



[TOC]

![](C:\Users\lzx\AppData\Roaming\Typora\typora-user-images\image-20200430115106897.png)

##### 1.   构造函数

```java
//没有参数的构造方法 默认创建非公平
public ReentrantLock() {
    sync = new NonfairSync();
}
```

```java
//参数为boolean 为true为公平锁 false为非公平
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

##### 2. 公平锁 FairSync

![image-20200507131112258](C:\Users\lzx\AppData\Roaming\Typora\typora-user-images\image-20200507131112258.png)

###### 继承Sync   Sync实现了AbstractQueuedSynchronizer 同步器

##### 2.1 方法 lock() 流程

```java
//锁方法
public void lock() {
    //此处的sync 如果为公平锁为 FairSync 如果为非公平锁 NonFairSync
    sync.lock();
}
//此处以FairSync为例
final void lock() {
     acquire(1);
}
//调用AbstractQueuedSynchronizer类中方法 参数为状态值
public final void acquire(int arg) {
    //如果tryAcquire为false首次尝试加锁失败
    //acquireQueued
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            //返回中断状态 为true 进行中断
            selfInterrupt();
    }
//1 首先调用tryAcquire(arg) 尝试加锁 需要在继承中实现 在FairSync中
protected final boolean tryAcquire(int acquires) {
    		//获取当前线程
            final Thread current = Thread.currentThread();
    		//获取状态值
            int c = getState();
    		//为零 说明无锁
            if (c == 0) {
                //hasQueuedPredecessors 判断链表的头和尾不相同 并且 头结点的下一个结点等于null或者 				下一个结点的线程不等于当前线程
                if (!hasQueuedPredecessors() &&
                    //通过cas设置状态值为 1 如果成功
                    compareAndSetState(0, acquires)) {
                    //将当前锁的线程设置为当前线程
                    setExclusiveOwnerThread(current);
                    //返回true
                    return true;
                }
            }
    		//重入逻辑 判断当前的线程为当前锁的占用线程
            else if (current == getExclusiveOwnerThread()) {
                //状态+acquires
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                //更新状态
                setState(nextc);
                return true;
            }
    		//加锁失败
            return false;
        }
    //acquireQueued(addWaiter(Node.EXCLUSIVE), arg) 中先调用addWaiter方法
	// 其中的Mode 为了区分独占锁还是共享锁
	// static final Node SHARED = new Node();
	// static final Node EXCLUSIVE = null;
	private Node addWaiter(Node mode) {
        //构建一个Node 参数为当前线程 独占共享标识
        Node node = new Node(Thread.currentThread(), mode);
        //首先将尾结点赋值给当前局部变量
        Node pred = tail;
        //当前变量不为空 说明tail 已经初始化过
        if (pred != null) {
            //将新创建的Node结点的上一个结点
            node.prev = pred;
            //并尝试将当前创建的结点赋值给tail
            if (compareAndSetTail(pred, node)) {
                //设置成功后将链表连接
                //将之前tail的next设置为当前创建的node
                pred.next = node;
                return node;
            }
        }
        //走到这个方法说明 锁竞争激烈 或者 还没有初始化tail
        enq(node);
        return node;
    }

//enq方法 第一次加锁一定会走两次循环
private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            //为空
            if (t == null) { 
                //进行cas初始化将head 结点设置成当前新创建的node
                //1 
                if (compareAndSetHead(new Node()))
                    //成功后将head 赋值给tail
                    tail = head;
            } else {
                //2
                // 将t 赋值给当前创建的node的上一个结点
                node.prev = t;
                //并且尝试cas 将当前node赋值给tail
                if (compareAndSetTail(t, node)) {
                    //成功后将创建的node赋值给t的下一个结点 进行链表连接
                    t.next = node;
                    return t;
                }
            }
            //没有成功再次进行自旋
        }
    }
//acquireQueued
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                //拿取当前结点的上一个node
                final Node p = node.predecessor();
                //如果为头结点 尝试一次获取 （这为优化假如该线程走到这的时候上一个线程释放了锁就不用进行自					旋了）
                if (p == head && tryAcquire(arg)) {
                    //将当前结点赋值给头结点 并且将当前线程 和上一个结点赋值为空
                    setHead(node);
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
//shouldParkAfterFailedAcquire
//pred 为node的上一个结点 node为当前结点
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    	//获取上一个结点的状态
        int ws = pred.waitStatus;
    	//① CANCELLED，值为1，由于同步队列中等待线程超时或者被中断，需要从同步队列中取消等待，节点进入该          状态将不会变化。② SIGNAL值为-1，后继节点的线程处于等待状态，而当前线程如果释放了同步状态或者取            消，将会通知后继节点，使得后继节点得以运行。③ CONDITION值为-2，节点在等待队列中，节点等待在              Condtion上，当其他线程对Condtion调用了signal方法后，该节点将会从等待队列中转移到同步队列中，加         入到对同步状态的获取中。④ PROPAGATE值为-3，表示下一次共享式同步状态将会无条件地被传播下去。⑤      		 INITAL，值为0，初始状态
        if (ws == Node.SIGNAL)
            //返回true当前线程将挂起
            return true;
    	//说明已经取消了
        if (ws > 0) {
            //这段代码是将取消的结点去除掉
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            //将上一个节点标记为SIGNAL
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
    	//返回false
        return false;
    }
//parkAndCheckInterrupt()
//将当前线程挂起 
private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
    //取消挂起后检测在挂起时间内是否存在中断
        return Thread.interrupted();
    }

//如果失败 cancelAcquire(node);
 private void cancelAcquire(Node node) {
        if (node == null)
            return;
     	//将要取消的结点线程赋值为null
        node.thread = null;
     	//获取上一个结点
        Node pred = node.prev;
     	//如果状态为大于0就继续往前找
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;
     	//node的不是取消状态的前置结点
        Node predNext = pred.next;
     	//将node的等待状态赋值为取消
        node.waitStatus = Node.CANCELLED;
     	//如果取消的结点为尾结点 并且 取消结点的上一个结点cas成功
        if (node == tail && compareAndSetTail(node, pred)) {
            //将上一个结点的下一个结点cas赋值为空
            compareAndSetNext(pred, predNext, null);
        } else {
            
            int ws;
            //找到当前要取消的结点 并且不是头结点 状态为SIGNAL 或者 设置SIGNAL成功 并且前置结点的线程不				为空
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) {
                //获取要取消结点的下一个结点
                Node next = node.next;
                //如果不为空 并且状态不是取消
                if (next != null && next.waitStatus <= 0)
                    //cas将链表连接
                    compareAndSetNext(pred, predNext, next);
            } else {
                //取消挂起
                unparkSuccessor(node);
            }
            node.next = node; 
        }
    }
//unparkSuccessor
 private void unparkSuccessor(Node node) {
        //获取要取消结点的等待状态
        int ws = node.waitStatus;
     	//如果小于0
        if (ws < 0)
            //通过cas将状态赋为0
            compareAndSetWaitStatus(node, ws, 0);
        Node s = node.next;
     	//如果 下一个结点为空 或者 状态为取消
        if (s == null || s.waitStatus > 0) {
            //为了判断
            s = null;
            //从tail开始查找往前第一个状态小于零的node
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            //取消挂起状态
            LockSupport.unpark(s.thread);
    }
```

##### 2.2 方法 unLock() 流程

```java
//sync 以FairSync为例
public void unlock() {
    sync.release(1);
}
```

```java
//AbstractQueuedSynchronizer 中的方法
public final boolean release(int arg) {
    //tryRelease 在ReentrantLock中的抽象静态类Sync
    //如果为true
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            //当h的下一个结点不为null 或者 状态为取消
            //通过自旋来查找需要释放的node
            //否则直接下一个进行释放
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

```java
//尝试减
protected final boolean tryRelease(int releases) {
    //获取当前同步器的状态 减少releases
    int c = getState() - releases;
    //如果当前线程不是当前同步器的占用线程
    if (Thread.currentThread() != getExclusiveOwnerThread())
        //抛出监控器状态异常
        throw new IllegalMonitorStateException();
    boolean free = false;
    //如果当前状态为0 说明已经没有线程占用锁
    if (c == 0) {
        free = true;
        //将当前同步器的占用线程恢复为null
        setExclusiveOwnerThread(null);
    }
    //设置状态
    setState(c);
    return free;
}
```

##### 2.3  Condition  使用Condition的前提的先加锁  和 object 的wait一样 需要获取synchronized 锁 基于同步队列与等待队列实现

###### 	2.3.1 ReentrantLock 中的 newCondition()

```java
//创建condition方法
final ConditionObject newCondition() {
    //创建ConditionObject对象
    return new ConditionObject();
}
```

###### 	2.3.2 ConditionObject 对象结构

![image-20200507181520252](C:\Users\lzx\AppData\Roaming\Typora\typora-user-images\image-20200507181520252.png)

###### 2.3.3 方法 await()

```java
public final void await() throws InterruptedException {
    //支持中断操作
    if (Thread.interrupted())
        throw new InterruptedException();
    //添加到等待队列 并且 删除不是等待状态的结点
    Node node = addConditionWaiter();
    //尝试从同步队列移除 之前锁的时候是一点点往上加 在await是直接减去 如果去除失败 则将waitStatus状态
    //变为取消
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    //是否在同步队列中
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        //checkInterruptWhileWaiting 判断异常处理形式
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    //尝试加锁
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) 
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

###### 2.3.4 方法signal()

```java
public final void signal() {
    //判断持有线程
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```

######  doSignal() 方法

```java
private void doSignal(Node first) {
    do {
        //首先将首节点从等待队列中移除
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
```

###### transferForSignal()方法

```java
final boolean transferForSignal(Node node) {
    //首先cas将等待状态变成初始状态
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
    //成功后加入到同步队列
    Node p = enq(node);
    //获取状态
    int ws = p.waitStatus;
    //p 为当前要唤醒结点在同步队列中的前一个结点 当p为取消或者变为SIGNAL状态失败将取消当前等到队列线程头结    点
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```