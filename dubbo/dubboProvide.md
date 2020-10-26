dubbo服务提供 执行链

ProtocolFilterWrapper->EchoFilter->ClassLoaderFilter->GenericFilter->ContextFilter->TraceFilter->TimeoutFilter->MonitorFilter->ExceptionFilter->InvokerWrapper->DelegateProviderMetaDataInvoker->AbstractProxyInvoker->Wrapper#invokeMethod()

```java
-110000   -30000  -20000  -10000  0 0 
```

```java
HeaderExchanger#bind
    handle=ExchangeHandlerAdapter$lamble
    
     newHandle = DecodeHandler->HeaderExchangeHandler->handle
    
NettyTransporter#bind
    NettyServer(URL url, ChannelHandler handler)
    AbstractServer(ExecutorUtil.setThreadName(url, SERVER_THREAD_POOL_NAME), ChannelHandlers.wrap(handler, url))
    ChannelHandlers#wrapInternal
  MultiMessageHandler-> HeartbeatHandler-> AllDispatcher#dispatch
        AllChannelHandler(handler, url)
    
    AbstractServer#doOpen
MultiMessageHandler-> HeartbeatHandler->AllChannelHandler->DecodeHandler->HeaderExchangeHandler<-ExchangeHandlerAdapter$lamble
    
ProtocolFilterWrapper->EchoFilter->ClassLoaderFilter->GenericFilter->ContextFilter->TraceFilter->TimeoutFilter->MonitorFilter->ExceptionFilter->InvokerWrapper->DelegateProviderMetaDataInvoker->AbstractProxyInvoker->Wrapper#invokeMethod()
    



```

#### 客户端

```java
MockClusterInvoker->AbstractCluster->AbstractClusterInvoker(Directory拉取获取invoke集合LoadBalance 获取负载均衡算法 默认random)->FailoverClusterInvoker(配置容错策略 doInvoke)
```





Failed to check the status of the service com.lzx.rpc.api.InterfaceApi. No provider available for the service dubbo1/com.lzx.rpc.api.InterfaceApi from the url nacos://118.190.155.155:8848/org.apache.dubbo.registry.RegistryService?application=dubbo-client&cluster=failfast&dubbo=2.0.2&group=dubbo1&init=false&interface=com.lzx.rpc.api.InterfaceApi&loadbalance=roundrobin&metadata-type=remote&methods=sayHello&mock=com.hk.dubbo.server.dubboserver.mock.InterfaceApiMock&pid=27156&qos.enable=false&register.ip=192.168.126.187&release=2.7.8&revision=1.0-SNAPSHOT&side=consumer&sticky=false&timeout=100000&timestamp=1602316056568 to the consumer 192.168.126.187 use dubbo version 2.7.8