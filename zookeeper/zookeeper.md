# zookeeper

https://issues.apache.org/jira/secure/attachment/12436620/ZooInspector.zip  可视化工具

## curator spi使用

### create

```java
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
CuratorFramework client = CuratorFrameworkFactory.builder()
        .connectString("localhost:2181")
        .sessionTimeoutMs(5000)  // 会话超时时间
        .connectionTimeoutMs(5000) // 连接超时时间
        .retryPolicy(retryPolicy)
        .namespace("base") // 包含隔离名称
        .build();
  client.start();
  register.create().creatingParentContainersIfNeeded() // 递归创建所需父节点
                    .withMode(CreateMode.PERSISTENT) // 创建类型为持久节点
                    .forPath("/nodeA", "init".getBytes()); // 目录及内容
```

## getData

```java
String value = new String(register.getData().forPath("/nodeA"));
```

## delete

```java
register.delete().forPath("/nodeA");
```

### setData

```java
register.setData().forPath("/nodeA","liuzaixing".getBytes());
```

## watch机制

***1.PathCache监控一个ZNode的子节点. 当一个子节点增加， 更新，删除时， PathCache会改变它的状态， 会包含最新的子节点,子节点的数据和状态***

***2.NodeCache只监控指定的结点***

***3.TreeCache包括上面两种***

### PathCache

```java
PathChildrenCacheListener listener = (k,v)->{
    System.out.println("Event type:"+v.getType());
    if(null != v.getData()){
        System.out.println("data:"+v.getData().getPath()+"="+new String(v.getData().getData()));
    }
};
PathChildrenCache pathChildrenCache = new PathChildrenCache(
        curatorFramework,"/nodeA",true);
pathChildrenCache.start();
pathChildrenCache.getListenable().addListener(listener);
```

### NodeCache

```java
NodeCache nodeCache = new NodeCache(curatorFramework, "/nodeA", true);
NodeCacheListener listener = ()->{
    ChildData currentData = nodeCache.getCurrentData();
    if(null != currentData){
        System.out.println("结点数据"+new String(currentData.getData()));
    }else {
        System.out.println("结点被删除");
    }
};
nodeCache.getListenable().addListener(listener);
nodeCache.start();
```

### TreeCache

```java
TreeCache treeCache = new TreeCache(curatorFramework, "/nodeA");
TreeCacheListener listener = (curator,treeCacheEvent)->{
    System.out.println("type:"+treeCacheEvent.getType());
    System.out.println("path:"+treeCacheEvent.getData().getPath());
    System.out.println("data:"+new String(treeCacheEvent.getData().getData()));
};
treeCache.getListenable().addListener(listener);
treeCache.start();
```

### curator实现分布式锁

***InterProcessMultiLock***

```java
 ArrayList<String> strings = new ArrayList<>();
 strings.add("/lock");
 InterProcessMultiLock interProcessMultiLock =
         new InterProcessMultiLock(client, strings);
 for (int i=0;i<10;i++){
     new Thread(()->{
         try {
             tryLock(client,interProcessMultiLock);
         } catch (Exception e) {
             e.printStackTrace();
         }
     },"thread"+i).start();
 }
```

***使用临时顺序结点实现序号越小越先获取到锁***

```java
private boolean internalLockLoop(long startMillis, Long millisToWait, String ourPath) throws Exception
{
    boolean     haveTheLock = false;
    boolean     doDelete = false;
    try
    {
        if ( revocable.get() != null )
        {
            client.getData().usingWatcher(revocableWatcher).forPath(ourPath);
        }
        while ( (client.getState() == CuratorFrameworkState.STARTED) && !haveTheLock )
        {
            List<String>        children = getSortedChildren();
            String              sequenceNodeName = ourPath.substring(basePath.length() + 1); 
PredicateResults    predicateResults = driver.getsTheLock(client, children, sequenceNodeName, maxLeases);
            if ( predicateResults.getsTheLock() )
            {
                haveTheLock = true;
            }
            else
            {
                String  previousSequencePath = basePath + "/" + predicateResults.getPathToWatch();
                synchronized(this)
                {
                    try 
                    {
                   //添加监听
                    client.getData().usingWatcher(watcher).forPath(previousSequencePath);
                        if ( millisToWait != null )
                        {
                            millisToWait -= (System.currentTimeMillis() - startMillis);
                            startMillis = System.currentTimeMillis();
                            if ( millisToWait <= 0 )
                            {
                                //超时删除
                                doDelete = true;   
                                break;
                            }
							//等待
                            wait(millisToWait);
                        }
                        else
                        {
                            //等待和synchronized配合使用
                            wait();
                        }
                    }
                }
            }
        }
    }
    catch ( Exception e )
    {
        ThreadUtils.checkInterrupted(e);
        doDelete = true;
        throw e;
    }
    finally
    {
        //超时和异常删除 防止死锁
        if ( doDelete )
        {
            deleteOurPath(ourPath);
        }
    }
    return haveTheLock;
}
```

***解锁***

```java
public void release() throws Exception
{
    Thread currentThread = Thread.currentThread();
    LockData lockData = threadData.get(currentThread);
    try
    {   //解锁
        internals.releaseLock(lockData.lockPath);
    }
    finally
    {
        //从当前线程集合中删除
        threadData.remove(currentThread);
    }
}
```

### lead选举

​	***LeaderLatch***

```java
for (int i=0 ;i<10;i++){
    LeaderLatch leaderLatch = new LeaderLatch(curatorFramework, "/latch", "test" + i);
    leaderLatchs.add(leaderLatch);
    leaderLatch.start();
}
Thread.sleep(1000);
for (LeaderLatch leaderLatch : leaderLatchs) {
    if (leaderLatch.hasLeadership()){
        temp = leaderLatch;
        System.out.println("当前lead是 "+leaderLatch.getId());
        break;
    }
    //关闭后自动选举
    temp.close();
}
```

​	***LeaderElection***

```java
//1客户端2选举路径3监听
new LeaderSelector(client,"/leaderElection",this);
```

```java
//监听类
public class LeaderElectionListen extends LeaderSelectorListenerAdapter implements Closeable{

}
```

```java
@Override
public void takeLeadership(CuratorFramework client) throws Exception {
   //选为leader后走到这个方法
    //执行完成这个方法后 会自动放弃leader
}
```

#### zookeeper 入口

##### QuorumPeerMain

```java
public static void main(String[] args) {
    QuorumPeerMain main = new QuorumPeerMain();
    main.initializeAndRun(args);
}
```

```java
protected void initializeAndRun(String[] args)
    throws ConfigException, IOException
{
    //获取对应的配置
    QuorumPeerConfig config = new QuorumPeerConfig();
    if (args.length == 1) {
        config.parse(args[0]);
    }
    DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
            .getDataDir(), config.getDataLogDir(), config
            .getSnapRetainCount(), config.getPurgeInterval());
    purgeMgr.start();

    if (args.length == 1 && config.servers.size() > 0) {
        //如果是集群 就会走到这个方法
        runFromConfig(config);
    } else {
        //单机走这个方法
        ZooKeeperServerMain.main(args);
    }
}
```

```java
public void runFromConfig(QuorumPeerConfig config) throws IOException {
 	  //注册jmx
      ManagedUtil.registerLog4jMBeans();
  try {
      //对外暴露服务工厂
      ServerCnxnFactory cnxnFactory = ServerCnxnFactory.createFactory();
      cnxnFactory.configure(config.getClientPortAddress(),
                            config.getMaxClientCnxns());
      quorumPeer = getQuorumPeer();
      quorumPeer.setQuorumPeers(config.getServers());
      quorumPeer.setTxnFactory(new FileTxnSnapLog(
              new File(config.getDataLogDir()),
              new File(config.getDataDir())));
      quorumPeer.setElectionType(config.getElectionAlg());
      quorumPeer.setMyid(config.getServerId());
      quorumPeer.setTickTime(config.getTickTime());
      quorumPeer.setInitLimit(config.getInitLimit());
      quorumPeer.setSyncLimit(config.getSyncLimit());
      quorumPeer.setQuorumListenOnAllIPs(config.getQuorumListenOnAllIPs());
      quorumPeer.setCnxnFactory(cnxnFactory);
      quorumPeer.setQuorumVerifier(config.getQuorumVerifier());
      quorumPeer.setClientPortAddress(config.getClientPortAddress());
      quorumPeer.setMinSessionTimeout(config.getMinSessionTimeout());
      quorumPeer.setMaxSessionTimeout(config.getMaxSessionTimeout());
      quorumPeer.setZKDatabase(new ZKDatabase(quorumPeer.getTxnFactory()));
      quorumPeer.setLearnerType(config.getPeerType());
      quorumPeer.setSyncEnabled(config.getSyncEnabled());
      quorumPeer.setQuorumSaslEnabled(config.quorumEnableSasl);
      if(quorumPeer.isQuorumSaslAuthEnabled()){
          quorumPeer.setQuorumServerSaslRequired(config.quorumServerRequireSasl);
          quorumPeer.setQuorumLearnerSaslRequired(config.quorumLearnerRequireSasl);
          quorumPeer.setQuorumServicePrincipal(config.quorumServicePrincipal);
          quorumPeer.setQuorumServerLoginContext(config.quorumServerLoginContext);
          quorumPeer.setQuorumLearnerLoginContext(config.quorumLearnerLoginContext);
      }

      quorumPeer.setQuorumCnxnThreadsSize(config.quorumCnxnThreadsSize);
      quorumPeer.initialize();
      //以上全是配置 这是启动的位置
      quorumPeer.start();
      quorumPeer.join();
  } catch (InterruptedException e) {
  }
}
```

```java
//quorumPeer
@Override
public synchronized void start() {
    //加载数据
    loadDataBase();
    //开启对外服务 cnxnFactory 有两种 默认为NIOServerCnxnFactory  NettyServerCnxnFactory
    //zookeeper.serverCnxnFactory 这个可以配置
    cnxnFactory.start();
    //开启zab选举
    startLeaderElection();
    //执行当前实例的run方法
    super.start();
}
```

#### NIOServerCnxnFactory

***run()***

```java
//执行2181端口的 客服端发过来的数据 NIO
public void run() {
    while (!ss.socket().isClosed()) {
            selector.select(1000);
            Set<SelectionKey> selected;
            synchronized (this) {
                selected = selector.selectedKeys();
            }
            ArrayList<SelectionKey> selectedList = new ArrayList<SelectionKey>(
                    selected);
            Collections.shuffle(selectedList);
            for (SelectionKey k : selectedList) {
                if ((k.readyOps() & SelectionKey.OP_ACCEPT) != 0) {
                    SocketChannel sc = ((ServerSocketChannel) k
                            .channel()).accept();
                    InetAddress ia = sc.socket().getInetAddress();
                    int cnxncount = getClientCnxnCount(ia);
                    if (maxClientCnxns > 0 && cnxncount >= maxClientCnxns){
                        LOG.warn("Too many connections from " + ia
                                 + " - max is " + maxClientCnxns );
                        sc.close();
                    } else {
                        LOG.info("Accepted socket connection from "
                                 + sc.socket().getRemoteSocketAddress());
                        sc.configureBlocking(false);
                        SelectionKey sk = sc.register(selector,
                                SelectionKey.OP_READ);
                        NIOServerCnxn cnxn = createConnection(sc, sk);
                        sk.attach(cnxn);
                        addCnxn(cnxn);
                    }
                } else if ((k.readyOps() & (SelectionKey.OP_READ | SelectionKey.OP_WRITE)) != 0) {
                    NIOServerCnxn c = (NIOServerCnxn) k.attachment();
                    //执行doIO方法 客户端也存在一样的方法
                    c.doIO(k);
                } else {
                    if (LOG.isDebugEnabled()) {
                        LOG.debug("Unexpected ops in select "
                                  + k.readyOps());
                    }
                }
            }
            selected.clear();
    }
    closeAll();
}
```

#### startLeaderElection 开始选举

```java
synchronized public void startLeaderElection() {
    //构建选票
     currentVote = new Vote(myid, getLastLoggedZxid(), getCurrentEpoch());
    for (QuorumServer p : getView().values()) {
        //判断当前结点是不是在配置中
        if (p.id == myid) {
            myQuorumAddr = p.addr;
            break;
        }
    }
    if (electionType == 0) {
        try {
            udpSocket = new DatagramSocket(myQuorumAddr.getPort());
            responder = new ResponderThread();
            responder.start();
        } catch (SocketException e) {
            throw new RuntimeException(e);
        }
    }
    this.electionAlg = createElectionAlgorithm(electionType);
}
```

```java
protected Election createElectionAlgorithm(int electionAlgorithm){
    Election le=null;
    switch (electionAlgorithm) {
    case 3:
        qcm = createCnxnManager();
        QuorumCnxManager.Listener listener = qcm.listener;
        if(listener != null){
            //开启监听
            listener.start();
            le = new FastLeaderElection(this, qcm);
        } else {
            LOG.error("Null listener when initializing cnx manager");
        }
        break;
    default:
        assert false;
    }
    return le;
}
```

```java
//选举端口暴露 使用socket
@Override
public void run() {
    int numRetries = 0;
    InetSocketAddress addr;
    while((!shutdown) && (numRetries < 3)){
            ss = new ServerSocket();
            ss.setReuseAddress(true);
            if (listenOnAllIPs) {
                int port = view.get(QuorumCnxManager.this.mySid)
                    .electionAddr.getPort();
                addr = new InetSocketAddress(port);
            } else {
                addr = view.get(QuorumCnxManager.this.mySid)
                    .electionAddr;
            }
            setName(view.get(QuorumCnxManager.this.mySid)
                    .electionAddr.toString());
            ss.bind(addr);
            while (!shutdown) {
                Socket client = ss.accept();
                setSockOpts(client);
                if (quorumSaslAuthEnabled) {
                    receiveConnectionAsync(client);
                } else {
                    receiveConnection(client);
                }
                numRetries = 0;
            }
 
}
```

```java
//发送线程 生产者消费者 模型
SendWorker sw = new SendWorker(sock, sid);
//接收线程 生产者消费者模型
RecvWorker rw = new RecvWorker(sock, din, sid, sw);
sw.setRecv(rw);

SendWorker vsw = senderWorkerMap.get(sid);

if(vsw != null)
    vsw.finish();

senderWorkerMap.put(sid, sw);
queueSendMap.putIfAbsent(sid, new ArrayBlockingQueue<ByteBuffer>(SEND_CAPACITY));
//启动
sw.start();
//启动
rw.start();
```

#### 开启选举 QuorumPeer * run()

```java
public void run() {
    setName("QuorumPeer" + "[myid=" + getId() + "]" +
            cnxnFactory.getLocalAddress());

    LOG.debug("Starting quorum peer");
    try {
        jmxQuorumBean = new QuorumBean(this);
        MBeanRegistry.getInstance().register(jmxQuorumBean, null);
        for(QuorumServer s: getView().values()){
            ZKMBeanInfo p;
            if (getId() == s.id) {
                p = jmxLocalPeerBean = new LocalPeerBean(this);
                try {
                    MBeanRegistry.getInstance().register(p, jmxQuorumBean);
                } catch (Exception e) {
                    LOG.warn("Failed to register with JMX", e);
                    jmxLocalPeerBean = null;
                }
            } else {
                p = new RemotePeerBean(s);
                try {
                    MBeanRegistry.getInstance().register(p, jmxQuorumBean);
                } catch (Exception e) {
                }
            }
        }
    } catch (Exception e) {
        jmxQuorumBean = null;
    }

    try {
        while (running) {
            switch (getPeerState()) {
            case LOOKING:
                //开启选举
                if (Boolean.getBoolean("readonlymode.enabled")) {
                    final ReadOnlyZooKeeperServer roZk = new ReadOnlyZooKeeperServer(
                            logFactory, this,
                            new ZooKeeperServer.BasicDataTreeBuilder(),
                            this.zkDb);
                    Thread roZkMgr = new Thread() {
                        public void run() {
                            try {
                                sleep(Math.max(2000, tickTime));
                                if (ServerState.LOOKING.equals(getPeerState())) {
                                    roZk.startup();
                                }
                    };
                    try {
                        roZkMgr.start();
                        setBCVote(null);
                        //开启选举
                        //electionAlg 为  FastLeaderElection 
                        setCurrentVote(makeLEStrategy().lookForLeader());
                    } catch (Exception e) {
                        setPeerState(ServerState.LOOKING);
                    } finally {
                        roZkMgr.interrupt();
                        roZk.shutdown();
                    }
                } else {
                    try {
                        setBCVote(null);
                        setCurrentVote(makeLEStrategy().lookForLeader());
                    } catch (Exception e) {
                        setPeerState(ServerState.LOOKING);
                    }
                }
                break;
            case OBSERVING:
                try {
                    //makeObserver  设置 ObserverZooKeeperServer
                    setObserver(makeObserver(logFactory));
                    observer.observeLeader();
                } finally {
                    observer.shutdown();
                    setObserver(null);
                    setPeerState(ServerState.LOOKING);
                }
                break;
            case FOLLOWING:
                try {
                    setFollower(makeFollower(logFactory));
                    follower.followLeader();
                }finally {
                    follower.shutdown();
                    setFollower(null);
                    setPeerState(ServerState.LOOKING);
                }
                break;
            case LEADING:
                LOG.info("LEADING");
                try {
                    setLeader(makeLeader(logFactory));
                    leader.lead();
                    setLeader(null);
                 finally {
                    if (leader != null) {
                        leader.shutdown("Forcing shutdown");
                        setLeader(null);
                    }
                    setPeerState(ServerState.LOOKING);
                }
                break;
            }
        }
    } finally {
       //jmx监控
        MBeanRegistry.getInstance().unregisterAll();
        //赋值为空 帮助gc
        jmxQuorumBean = null;
        //赋值为空 帮助gc
        jmxLocalPeerBean = null;
    }
}
```

##### FastLeaderElection

```java
//开启选举
public Vote lookForLeader() throws InterruptedException {
    try {
        self.jmxLeaderElectionBean = new LeaderElectionBean();
        MBeanRegistry.getInstance().register(
                self.jmxLeaderElectionBean, self.jmxLocalPeerBean);
    } catch (Exception e) {
        self.jmxLeaderElectionBean = null;
    }
    if (self.start_fle == 0) {
       self.start_fle = Time.currentElapsedTime();
    }
    try {
        //接收的选票
        HashMap<Long, Vote> recvset = new HashMap<Long, Vote>();
        //发送的选票
        HashMap<Long, Vote> outofelection = new HashMap<Long, Vote>();
        int notTimeout = finalizeWait;
        synchronized(this){
            //逻辑时钟增加
            logicalclock.incrementAndGet();
            //更新当前的 myid 事务id 逻辑时钟
            updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
        }
        //发送选票 生产者消费者 模型
        sendNotifications();
        //如果是选举状态
        while ((self.getPeerState() == ServerState.LOOKING) &&
                (!stop)){
            //接收通知
            Notification n = recvqueue.poll(notTimeout,
                    TimeUnit.MILLISECONDS);
            if(n == null){
                if(manager.haveDelivered()){
                    sendNotifications();
                } else {
                    manager.connectAll();
                }
                int tmpTimeOut = notTimeout*2;
                notTimeout = (tmpTimeOut < maxNotificationInterval?
                        tmpTimeOut : maxNotificationInterval);
            }
            //验证myid
            else if(validVoter(n.sid) && validVoter(n.leader)) {
                switch (n.state) {
                //选举状态
                case LOOKING:
                     //接收过来的时钟 大于当前的时钟 
                    if (n.electionEpoch > logicalclock.get()) {
                        //更新当前时钟 
                        logicalclock.set(n.electionEpoch);
                        //清空接收集合
                        recvset.clear();
                        //验证当前时钟 zxid myid 
                        if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
                            //更新为接收的消息
                            updateProposal(n.leader, n.zxid, n.peerEpoch);
                        } else {
                            //更新为初始化参数
                            updateProposal(getInitId(),
                                    getInitLastLoggedZxid(),
                                    //更新时钟
                                    getPeerEpoch());
                        }
                        //发送通知
                        sendNotifications();
                        //接收时钟小 直接返回
                    } else if (n.electionEpoch < logicalclock.get()) {
                        break;
                        //判断
                    } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                            proposedLeader, proposedZxid, proposedEpoch)) {
                        updateProposal(n.leader, n.zxid, n.peerEpoch);
                        sendNotifications();
                    }
                    if(LOG.isDebugEnabled()){
                    }
                    recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));
                    //进行宣票 过半 
                    if (termPredicate(recvset,
                            new Vote(proposedLeader, proposedZxid,
                                    logicalclock.get(), proposedEpoch))) {
                        //为了防止在宣票过程中 存在接收票
                        while((n = recvqueue.poll(finalizeWait,
                                TimeUnit.MILLISECONDS)) != null){
                            //判断选票
                            if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                    proposedLeader, proposedZxid, proposedEpoch)){
                                //接收的条件更优 放入接收队列
                                recvqueue.put(n);
                                break;
                            }
                        }
                        if (n == null) {
                            self.setPeerState((proposedLeader == self.getId()) ?
                                    ServerState.LEADING: learningState());

                            Vote endVote = new Vote(proposedLeader,
                                                    proposedZxid,
                                                    logicalclock.get(),
                                                    proposedEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                    }
                    break;
                case OBSERVING:
                    break;
                case FOLLOWING:
                case LEADING:
                    if(n.electionEpoch == logicalclock.get()){
                        recvset.put(n.sid, new Vote(n.leader,
                                                      n.zxid,
                                                      n.electionEpoch,
                                                      n.peerEpoch));
                       
                        if(ooePredicate(recvset, outofelection, n)) {
                            self.setPeerState((n.leader == self.getId()) ?
                                    ServerState.LEADING: learningState());

                            Vote endVote = new Vote(n.leader, 
                                    n.zxid, 
                                    n.electionEpoch, 
                                    n.peerEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                    }
                    outofelection.put(n.sid, new Vote(n.version,
                                                        n.leader,
                                                        n.zxid,
                                                        n.electionEpoch,
                                                        n.peerEpoch,
                                                        n.state));
       
                    if(ooePredicate(outofelection, outofelection, n)) {
                        synchronized(this){
                            logicalclock.set(n.electionEpoch);
                            self.setPeerState((n.leader == self.getId()) ?
                                    ServerState.LEADING: learningState());
                        }
                        Vote endVote = new Vote(n.leader,
                                                n.zxid,
                                                n.electionEpoch,
                                                n.peerEpoch);
                        leaveInstance(endVote);
                        return endVote;
                    }
                    break;
                default:
                    break;
                }
            } else {
                }
                }
            }
        }
        return null;
    } finally {
            if(self.jmxLeaderElectionBean != null){
                MBeanRegistry.getInstance().unregister(
                        self.jmxLeaderElectionBean);
            }
        self.jmxLeaderElectionBean = null;
    }
}
```

####  observer.observeLeader() 通知leader

```java
void followLeader() throws InterruptedException {
    self.end_fle = Time.currentElapsedTime();
    long electionTimeTaken = self.end_fle - self.start_fle;
    self.setElectionTimeTaken(electionTimeTaken);
    self.start_fle = 0;
    self.end_fle = 0;
    fzk.registerJMX(new FollowerBean(this, zk), self.jmxLocalPeerBean);
    try {
        QuorumServer leaderServer = findLeader();            
        try {
            //连接leader
            connectToLeader(leaderServer.addr, leaderServer.hostname);
            long newEpochZxid = registerWithLeader(Leader.FOLLOWERINFO);
            long newEpoch = ZxidUtils.getEpochFromZxid(newEpochZxid);
            if (newEpoch < self.getAcceptedEpoch()) {
             
            }
            //同步leader
            syncWithLeader(newEpochZxid);                
            QuorumPacket qp = new QuorumPacket();
            while (this.isRunning()) {
                readPacket(qp);
                processPacket(qp);
            }
        } catch (Exception e) {
            try {
                sock.close();
            } catch (IOException e1) {
                e1.printStackTrace();
            }
            pendingRevalidations.clear();
        }
    } finally {
        zk.unregisterJMX((Learner)this);
    }
}
```

#### Learner 中的 syncWithLeader

```java
protected void syncWithLeader(long newLeaderZxid) throws IOException, InterruptedException{
    QuorumPacket ack = new QuorumPacket(Leader.ACK, 0, null, null);
    QuorumPacket qp = new QuorumPacket();
    long newEpoch = ZxidUtils.getEpochFromZxid(newLeaderZxid);
    boolean snapshotNeeded = true;
    readPacket(qp);
    LinkedList<Long> packetsCommitted = new LinkedList<Long>();
    LinkedList<PacketInFlight> packetsNotCommitted = new LinkedList<PacketInFlight>();
    synchronized (zk) {
        if (qp.getType() == Leader.DIFF) {
            snapshotNeeded = false;
        }
        else if (qp.getType() == Leader.SNAP) {
            zk.getZKDatabase().clear();
            zk.getZKDatabase().deserializeSnapshot(leaderIs);
            String signature = leaderIs.readString("signature");
            if (!signature.equals("BenWasHere")) {
                throw new IOException("Missing signature");                   
            }
            zk.getZKDatabase().setlastProcessedZxid(qp.getZxid());
        } else if (qp.getType() == Leader.TRUNC) {
            boolean truncated=zk.getZKDatabase().truncateLog(qp.getZxid());
            if (!truncated) {
                System.exit(13);
            }
            zk.getZKDatabase().setlastProcessedZxid(qp.getZxid());
        }
        else {
            System.exit(13);

        }
        zk.createSessionTracker();
        long lastQueued = 0;
        boolean isPreZAB1_0 = true;
        boolean writeToTxnLog = !snapshotNeeded;
        outerLoop:
        while (self.isRunning()) {
            readPacket(qp);
            switch(qp.getType()) {
            case Leader.PROPOSAL:
                PacketInFlight pif = new PacketInFlight();
                pif.hdr = new TxnHeader();
                pif.rec = SerializeUtils.deserializeTxn(qp.getData(), pif.hdr);
                if (pif.hdr.getZxid() != lastQueued + 1) {
                }
                lastQueued = pif.hdr.getZxid();
                packetsNotCommitted.add(pif);
                break;
            case Leader.COMMIT:
                if (!writeToTxnLog) {
                    pif = packetsNotCommitted.peekFirst();
                    if (pif.hdr.getZxid() != qp.getZxid()) {
                    } else {
                        //执行事务操作
                        zk.processTxn(pif.hdr, pif.rec);
                        packetsNotCommitted.remove();
                    }
                } else {
                    packetsCommitted.add(qp.getZxid());
                }
                break;
            case Leader.INFORM:
                PacketInFlight packet = new PacketInFlight();
                packet.hdr = new TxnHeader();
                packet.rec = SerializeUtils.deserializeTxn(qp.getData(), packet.hdr);
                if (packet.hdr.getZxid() != lastQueued + 1) {
                }
                lastQueued = packet.hdr.getZxid();
                if (!writeToTxnLog) {
                    zk.processTxn(packet.hdr, packet.rec);
                } else {
                    packetsNotCommitted.add(packet);
                    packetsCommitted.add(qp.getZxid());
                }
                break;
            case Leader.UPTODATE:
                if (isPreZAB1_0) {
                    zk.takeSnapshot();
                    self.setCurrentEpoch(newEpoch);
                }
                self.cnxnFactory.setZooKeeperServer(zk);                
                break outerLoop;
            case Leader.NEWLEADER: 
                File updating = new File(self.getTxnFactory().getSnapDir(),
                                    QuorumPeer.UPDATING_EPOCH_FILENAME);
                if (!updating.exists() && !updating.createNewFile()) {
                    throw new IOException("Failed to create " +
                                          updating.toString());
                }
                if (snapshotNeeded) {
                    zk.takeSnapshot();
                }
                self.setCurrentEpoch(newEpoch);
                if (!updating.delete()) {
                    throw new IOException("Failed to delete " +
                                          updating.toString());
                }
                writeToTxnLog = true;
                isPreZAB1_0 = false;
                writePacket(new QuorumPacket(Leader.ACK, newLeaderZxid, null, null), true);
                break;
            }
        }
    }
    ack.setZxid(ZxidUtils.makeZxid(newEpoch, 0));
    writePacket(ack, true);
    sock.setSoTimeout(self.tickTime * self.syncLimit);
    //初始化
    zk.startup();
    self.updateElectionVote(newEpoch);
    if (zk instanceof FollowerZooKeeperServer) {
        FollowerZooKeeperServer fzk = (FollowerZooKeeperServer)zk;
        for(PacketInFlight p: packetsNotCommitted) {
            fzk.logRequest(p.hdr, p.rec);
        }
        for(Long zxid: packetsCommitted) {
            fzk.commit(zxid);
        }
    } else if (zk instanceof ObserverZooKeeperServer) {
        ObserverZooKeeperServer ozk = (ObserverZooKeeperServer) zk;
        for (PacketInFlight p : packetsNotCommitted) {
            Long zxid = packetsCommitted.peekFirst();
            if (p.hdr.getZxid() != zxid) {
                continue;
            }
            packetsCommitted.remove();
            Request request = new Request(null, p.hdr.getClientId(),
                    p.hdr.getCxid(), p.hdr.getType(), null, null);
            request.txn = p.rec;
            request.hdr = p.hdr;
            //提交请求
            ozk.commitRequest(request);
        }
    } else {
        throw new UnsupportedOperationException("Unknown server type");
    }
}
```

#### zk.startup()

```java
//ZooKeeperServer 类中
public synchronized void startup() {
    if (sessionTracker == null) {
        createSessionTracker();
    }
    //开启session
    startSessionTracker();
    //设置执行器链
    setupRequestProcessors();
    //注册jmx 
    registerJMX();
    //设置状态为运行
    setState(State.RUNNING);
    //通知阻塞到 synchronized 等待队列中的 线程
    notifyAll();
}
```

#### setupRequestProcessors 以 ObserverZookeeperServer为例

```java
protected void setupRequestProcessors() {  
    //数据校验 生产者消费者模型
    RequestProcessor finalProcessor = new FinalRequestProcessor(this);
    //提交请求 只是做转发
    commitProcessor = new CommitProcessor(finalProcessor,
            Long.toString(getServerId()), true,
            getZooKeeperServerListener());
    commitProcessor.start();
    //设置初始结点
    firstProcessor = new ObserverRequestProcessor(this, commitProcessor);
    ((ObserverRequestProcessor) firstProcessor).start();
    if (syncRequestProcessorEnabled) {
        //同步数据
        syncProcessor = new SyncRequestProcessor(this, null);
        syncProcessor.start();
    }
}
```