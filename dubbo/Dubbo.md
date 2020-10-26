# Dubbo

### 集群容错

1. failover 失败自动切换，尝试其他服务器  retries 设置 重试次数 默认为2 实际为3 之前执行的1 +2  ***FailoverCluster***
2. failback 记录日志并定时重试 **FailfastCluster**
3. failsafe 失败安全 失败忽略异常 **FailsafeCluster**
4. failfast 失败立即抛出异常 **FailbackCluster**
5. Forking(并行调用多个服务，一个成功立即返回)  **ForkingCluster**  dubbo.consumer.forks设置最大并行数
6. Broadcast(广播调用所有提供者，任意一个报错则报错) ***BroadcastCluster***

### 负载均衡

1. random 随机默认为随机 ***RandomLoadBalance*** 加权随机 可以指定权重

   ~~~java
     @Override
       protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
           int length = invokers.size();
           int totalWeight = 0;
           boolean sameWeight = true;
           // 下面这个循环有两个作用，第一是计算总权重 totalWeight，
           // 第二是检测每个服务提供者的权重是否相同
           for (int i = 0; i < length; i++) {
               int weight = getWeight(invokers.get(i), invocation);
               // 累加权重
               totalWeight += weight;
               // 检测当前服务提供者的权重与上一个服务提供者的权重是否相同，
               // 不相同的话，则将 sameWeight 置为 false。
               if (sameWeight && i > 0
                       && weight != getWeight(invokers.get(i - 1), invocation)) {
                   sameWeight = false;
               }
           }
           
           // 下面的 if 分支主要用于获取随机数，并计算随机数落在哪个区间上
           if (totalWeight > 0 && !sameWeight) {
               // 随机获取一个 [0, totalWeight) 区间内的数字
               int offset = random.nextInt(totalWeight);
               // 循环让 offset 数减去服务提供者权重值，当 offset 小于0时，返回相应的 Invoker。
               // 举例说明一下，我们有 servers = [A, B, C]，weights = [5, 3, 2]，offset = 7。
               // 第一次循环，offset - 5 = 2 > 0，即 offset > 5，
               // 表明其不会落在服务器 A 对应的区间上。
               // 第二次循环，offset - 3 = -1 < 0，即 5 < offset < 8，
               // 表明其会落在服务器 B 对应的区间上
               for (int i = 0; i < length; i++) {
                   // 让随机值 offset 减去权重值
                   offset -= getWeight(invokers.get(i), invocation);
                   if (offset < 0) {
                       // 返回相应的 Invoker
                       return invokers.get(i);
                   }
               }
           }
           
           // 如果所有服务提供者权重值相同，此时直接随机返回一个即可
           return invokers.get(random.nextInt(length));
       }
   ~~~

   

2. leastactive 最小链接数 ***LeastActiveLoadBalance*** 

   ~~~java
   public class LeastActiveLoadBalance extends AbstractLoadBalance {
   
       public static final String NAME = "leastactive";
   
       private final Random random = new Random();
   
       @Override
       protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
           int length = invokers.size();
           // 最小的活跃数
           int leastActive = -1;
           // 具有相同“最小活跃数”的服务者提供者（以下用 Invoker 代称）数量
           int leastCount = 0; 
           // leastIndexs 用于记录具有相同“最小活跃数”的 Invoker 在 invokers 列表中的下标信息
           int[] leastIndexs = new int[length];
           int totalWeight = 0;
           // 第一个最小活跃数的 Invoker 权重值，用于与其他具有相同最小活跃数的 Invoker 的权重进行对比，
           // 以检测是否“所有具有相同最小活跃数的 Invoker 的权重”均相等
           int firstWeight = 0;
           boolean sameWeight = true;
   
           // 遍历 invokers 列表
           for (int i = 0; i < length; i++) {
               Invoker<T> invoker = invokers.get(i);
               // 获取 Invoker 对应的活跃数
               int active = RpcStatus.getStatus(invoker.getUrl(), invocation.getMethodName()).getActive();
               // 获取权重 - ⭐️
               int weight = invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.WEIGHT_KEY, Constants.DEFAULT_WEIGHT);
               // 发现更小的活跃数，重新开始
               if (leastActive == -1 || active < leastActive) {
               	// 使用当前活跃数 active 更新最小活跃数 leastActive
                   leastActive = active;
                   // 更新 leastCount 为 1
                   leastCount = 1;
                   // 记录当前下标值到 leastIndexs 中
                   leastIndexs[0] = i;
                   totalWeight = weight;
                   firstWeight = weight;
                   sameWeight = true;
   
               // 当前 Invoker 的活跃数 active 与最小活跃数 leastActive 相同 
               } else if (active == leastActive) {
               	// 在 leastIndexs 中记录下当前 Invoker 在 invokers 集合中的下标
                   leastIndexs[leastCount++] = i;
                   // 累加权重
                   totalWeight += weight;
                   // 检测当前 Invoker 的权重与 firstWeight 是否相等，
                   // 不相等则将 sameWeight 置为 false
                   if (sameWeight && i > 0
                       && weight != firstWeight) {
                       sameWeight = false;
                   }
               }
           }
           
           // 当只有一个 Invoker 具有最小活跃数，此时直接返回该 Invoker 即可
           if (leastCount == 1) {
               return invokers.get(leastIndexs[0]);
           }
   
           // 有多个 Invoker 具有相同的最小活跃数，但它们之间的权重不同
           if (!sameWeight && totalWeight > 0) {
           	// 随机生成一个 [0, totalWeight) 之间的数字
               int offsetWeight = random.nextInt(totalWeight);
               // 循环让随机数减去具有最小活跃数的 Invoker 的权重值，
               // 当 offset 小于等于0时，返回相应的 Invoker
               for (int i = 0; i < leastCount; i++) {
                   int leastIndex = leastIndexs[i];
                   // 获取权重值，并让随机数减去权重值 - ⭐️
                   offsetWeight -= getWeight(invokers.get(leastIndex), invocation);
                   if (offsetWeight <= 0)
                       return invokers.get(leastIndex);
               }
           }
           // 如果权重相同或权重为0时，随机返回一个 Invoker
           return invokers.get(leastIndexs[random.nextInt(leastCount)]);
       }
   ~~~

   

3.  consistenthash 一致性hash算法 ConsistentHashLoadBalance  

   ~~~java
    private final ConcurrentMap<String, ConsistentHashSelector<?>> selectors = new ConcurrentHashMap<String, ConsistentHashSelector<?>>();
   
       @SuppressWarnings("unchecked")
       @Override
       protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
           String methodName = RpcUtils.getMethodName(invocation);
           //接口名 + 方法名
           String key = invokers.get(0).getUrl().getServiceKey() + "." + methodName;
           int invokersHashCode = invokers.hashCode();
           ConsistentHashSelector<T> selector = (ConsistentHashSelector<T>) selectors.get(key);
           //不相等说明 服务增加或者减少
           if (selector == null || selector.identityHashCode != invokersHashCode) {
               selectors.put(key, new ConsistentHashSelector<T>(invokers, methodName, invokersHashCode));
               selector = (ConsistentHashSelector<T>) selectors.get(key);
           }
           //开启选举
           return selector.select(invocation);
       }
   ~~~

   ```java
   //通过散hash 合 treeMap.ceilingEntry(key) 实现 这个方法 返回大于或者等于key的值 
   // 为空 返回firstEntity
   //ConsistentHashSelector
   ```

4. roundrobin 加权轮询 RoundRobinLoadBalance

   ```java
   //采用平滑式加权轮训 key=com.xxx.interface.methodname
   String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
   //WeightedRoundRobin 为墨盒 装载权重 以及上一次 的当前值
   ConcurrentMap<String, WeightedRoundRobin> map = methodWeightMap.computeIfAbsent(key, k -> new ConcurrentHashMap<>());
   int totalWeight = 0;
   long maxCurrent = Long.MIN_VALUE;
   long now = System.currentTimeMillis();
   Invoker<T> selectedInvoker = null;
   WeightedRoundRobin selectedWRR = null;
   //假设当前两个实例
   for (Invoker<T> invoker : invokers) {
       //identifyString = dobbo//:ip:port?interfacename=""
       String identifyString = invoker.getUrl().toIdentityString();
       //获取权重 默认为100
       int weight = getWeight(invoker, invocation);
       //判断墨盒是否存在 
       WeightedRoundRobin weightedRoundRobin = map.get(identifyString);
   	//不存在创建
       if (weightedRoundRobin == null) {
           weightedRoundRobin = new WeightedRoundRobin();
           weightedRoundRobin.setWeight(weight);
           map.putIfAbsent(identifyString, weightedRoundRobin);
           weightedRoundRobin = map.get(identifyString);
       }
       //权重改变 重新赋值
       if (weight != weightedRoundRobin.getWeight()) {
           weightedRoundRobin.setWeight(weight);
       }
       //上一个 当前值+ 权重  当前值 为上一个当前值减权重和 
       long cur = weightedRoundRobin.increaseCurrent();
       //为了判断 上一次使用时间 为了防止一直占用
       weightedRoundRobin.setLastUpdate(now);
       //maxCurrent 默认为 Long.Minvalue();
       // 如果当前 大于这个值
       if (cur > maxCurrent) {
           //赋值最大值
           maxCurrent = cur;
           //当前服务实例
           selectedInvoker = invoker;
           //当前实例对应的墨盒
           selectedWRR = weightedRoundRobin;
       }
       //计算总的 权重
       totalWeight += weight;
   }
   //判断当前实例 和缓存之间的记录是否相同 不相同 说明存在 下线服务或者新服务
   if (!updateLock.get() && invokers.size() != map.size()) {
       if (updateLock.compareAndSet(false, true)) {
           try {
               ConcurrentMap<String, WeightedRoundRobin> newMap = new ConcurrentHashMap<>(map);		
               //长期不用的 服务进行删除 默认为1分钟
               newMap.entrySet().removeIf(item -> now - item.getValue().getLastUpdate() > RECYCLE_PERIOD);
               methodWeightMap.put(key, newMap);
           } finally {
               updateLock.set(false);
           }
       }
   }
   //不为空 
   if (selectedInvoker != null) {
       //设置当前值 减去 权重总和
       selectedWRR.sel(totalWeight);
       return selectedInvoker;
   }
   //没有筛选出来 选择第一个
   return invokers.get(0);
   ```

registry://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=dubbo-server&check=false&dubbo=2.0.2&pid=3176&qos.enable=false&registry=zookeeper&release=2.7.6&timestamp=1590642426988





```java
HeartbeatHandler(AllChannelHandler(ChannelHandlerDispatcher(DecodeHandler(HeaderExchangeHandler(ExchangeHandlerAdapter()))))
```
