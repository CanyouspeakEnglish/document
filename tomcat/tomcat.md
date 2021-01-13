# Bootstrap

### 启动方法

> main() 入参start
>
> org.apache.catalina.startup.Bootstrap#init() 
>
> > org.apache.catalina.startup.Bootstrap#initClassLoaders
> >
> > > 加载各种classload commonLoader  catalinaLoader sharedLoader
> > >
> > > org.apache.catalina.startup.Bootstrap#createClassLoader (common)
> > >
> > > org.apache.catalina.startup.Bootstrap#createClassLoader (server)
> > >
> > > org.apache.catalina.startup.Bootstrap#createClassLoader (shared)
>
> > Thread.currentThread().setContextClassLoader(catalinaLoader)//设置当前线程的类加载器
> >
> > org.apache.catalina.security.SecurityClassLoad#securityClassLoad(java.lang.ClassLoader)
> >
> > > org.apache.catalina.security.SecurityClassLoad#securityClassLoad(java.lang.ClassLoader) 设置权限权限
> > >
> > > org.apache.catalina.security.SecurityClassLoad#securityClassLoad(java.lang.ClassLoader, boolean) 
> > >
> > > > org.apache.catalina.security.SecurityClassLoad#loadCorePackage 加载核心类catalina.core包下
> > > >
> > > > org.apache.catalina.security.SecurityClassLoad#loadCoyotePackage
> > > >
> > > > org.apache.catalina.security.SecurityClassLoad#loadLoaderPackage  catalina.loader
> > > >
> > > > org.apache.catalina.security.SecurityClassLoad#loadRealmPackage  catalina.realm.
> > > >
> > > > org.apache.catalina.security.SecurityClassLoad#loadServletsPackage catalina.servlets.DefaultServlet 等
> > > >
> > > > org.apache.catalina.security.SecurityClassLoad#loadSessionPackage catalina.session.StandardSession 等
> > > >
> > > > org.apache.catalina.security.SecurityClassLoad#loadUtilPackage .catalina.util.ParameterMap  RequestUtil TLSUtil
> > > >
> > > > org.apache.catalina.security.SecurityClassLoad#loadJakartaPackage  .http.Cookie
> > > >
> > > > org.apache.catalina.security.SecurityClassLoad#loadConnectorPackage  connector.
> > > >
> > > > org.apache.catalina.security.SecurityClassLoad#loadTomcatPackage  .tomcat.

> > loadClass("org.apache.catalina.startup.Catalina") 加载 Catalina 
> >
> > 反射调用Catalina 方法 setParentClassLoader()
> >
> > catalinaDaemon = Catalina 实例

> org.apache.catalina.startup.Bootstrap#daemon 这个全局变量等于刚刚创建的bootstrap
>
> 假如传入的是 agrs = new String[]{"start"} 
>
> org.apache.catalina.startup.Bootstrap#setAwait(true) 
>
> org.apache.catalina.startup.Bootstrap#load(args)
>
> > 反射调用Catalina中的load方法 入参为start Catalina>load(args)

> > > org.apache.catalina.startup.Catalina#arguments

> > > > 因为入参为start  isGenerateCode = false;

> > > org.apache.catalina.startup.Catalina#load()
> > >
> > > > org.apache.catalina.startup.Catalina#initNaming
> > > >
> > > > org.apache.catalina.startup.Catalina#parseServerXml (比较重要用于解析config/server.xml 解析的思想很有意思)
> > > >
> > > > > org.apache.catalina.startup.Catalina#createStartDigester
> > > > >
> > > > > > 创建Digester 这个对象承载着解析的配置以及每个标签对应处理方法的配置 new Digester();
> > > > > >
> > > > > > > org.apache.tomcat.util.digester.Digester#setFakeAttributes  Map<Class<?>, List<String>> key = StandardContext.class value = {source} key= Connector.class , value = {portOffset}

> > > > > > org.apache.tomcat.util.digester.Digester#addObjectCreate(java.lang.String, java.lang.String, java.lang.String) 
> > > > > >
> > > > > > ```java
> > > > > > digester.addObjectCreate("Server",
> > > > > >                          "org.apache.catalina.core.StandardServer",
> > > > > >                          "className");
> > > > > > ```

> > > > > > > org.apache.tomcat.util.digester.Digester#addRule(pattern, new ObjectCreateRule(className, attributeName)) //此处思想 每个类型不同的处理方法在 Rule中 例如ObjectCreateRule
> > > > > > >
> > > > > > > > org.apache.tomcat.util.digester.Rule#setDigester(this) 把digester设置进去
> > > > > > > >
> > > > > > > > org.apache.tomcat.util.digester.Digester#getRules
> > > > > > > >
> > > > > > > > > 判断Digester中的rules是不是为空 为空创建一个 new RulesBase() 并设置Digester

> > > > > > > > org.apache.tomcat.util.digester.Rules#add 判断缓存中是否存在 key = pattern value为 规则 

> > > > > > org.apache.tomcat.util.digester.Digester#push 设置操作对象为当前的 Catalina对象
> > > > > >
> > > > > > org.apache.tomcat.util.digester.Digester#parse(org.xml.sax.InputSource) 进行SAX解析
> > > > > >
> > > > > > > SAXParserFactory.newInstance() 创建解析工厂 1
> > > > > > >
> > > > > > > getFactory().newSAXParser() 创建解析器
> > > > > > >
> > > > > > > getParser().getXMLReader() 通过解析器获取reader
> > > > > > >
> > > > > > > 设置处理事件 由于 Digester 实现了DefaultHandler2 
> > > > > > >
> > > > > > > ```java
> > > > > > > reader.setDTDHandler(this);
> > > > > > > reader.setContentHandler(this);
> > > > > > > reader.setEntityResolver(entityResolver);
> > > > > > > reader.setErrorHandler(this);
> > > > > > > //由于Digester重写了 startElement(String namespaceURI, String localName, String qName, Attributes list)
> > > > > > > //endElement(String namespaceURI, String localName, String qName)
> > > > > > > ```

> > > > org.apache.catalina.startup.Catalina#getServer

> > > > > org.apache.catalina.Server#setCatalina
> > > > >
> > > > > org.apache.catalina.Server#setCatalinaHome //这是文件
> > > > >
> > > > > org.apache.catalina.Server#setCatalinaBase

> > > > org.apache.catalina.startup.Catalina#initStreams //替换打印流 SystemLogHandler(System.out) SystemLogHandler(System.err)

> > > > > org.apache.catalina.Lifecycle#init StandardServer 继承了LifecycleMBeanBase
> > > > >
> > > > > > org.apache.catalina.util.LifecycleBase#setStateInternal
> > > > > >
> > > > > > > org.apache.catalina.util.LifecycleBase#fireLifecycleEvent

> > > > org.apache.catalina.Lifecycle#init
> > > >
> > > > > org.apache.catalina.util.LifecycleBase#init
> > > > >
> > > > > org.apache.catalina.util.LifecycleBase#initInternal
> > > > >
> > > > > org.apache.catalina.core.StandardServer#initInternal
> > > > >
> > > > > > org.apache.catalina.core.StandardService#initInternal
> > > > > >
> > > > > > > org.apache.catalina.core.StandardEngine#initInternal
> > > > > > >
> > > > > > > > org.apache.catalina.core.ContainerBase#initInternal

> > > > > > > org.apache.catalina.core.StandardThreadExecutor#initInternal

> > > > > > org.apache.catalina.connector.Connector#initInternal 对外接收请求组件
> > > > > >
> > > > > > > org.apache.catalina.connector.CoyoteAdapter
> > > > > > >
> > > > > > > org.apache.coyote.AbstractProtocol#init
> > > > > > >
> > > > > > > > org.apache.tomcat.util.net.AbstractEndpoint#init
> > > > > > > >
> > > > > > > > org.apache.tomcat.util.net.AbstractEndpoint#bindWithCleanup
> > > > > > > >
> > > > > > > > > org.apache.tomcat.util.net.AbstractEndpoint#bind
> > > > > > > > >
> > > > > > > > > org.apache.tomcat.util.net.NioEndpoint#bind
> > > > > > > > >
> > > > > > > > > org.apache.tomcat.util.net.NioEndpoint#initServerSocket

`时序图`：

https://tomcat.apache.org/tomcat-10.0-doc/architecture/startup/serverStartup.pdf

`server`: server 代表整个容器,该接口很少被定制 org.apache.catalina.Server

`service`: service 一个server 可以存在多个service service也很少被定制重写 org.apache.catalina.Service

`Engine`：用来处理用户请求

`Host`: 虚拟主机

`Connector`: 连接器对外连接

`Context`：一个context代表着一个web应用



![Tomcat 架构图](D:\sourcecode\document\tomcat\5a26687d0001ca2712300718.jpg)
![image](https://github.com/liuxingxingghs/document/blob/master/tomcat/5a26687d0001ca2712300718.jpg)
ps -ef | grep tomcat

cat /pro/pid/status

* 参数说明

https://blog.csdn.net/u011425939/article/details/75335079

top -p pid

### Tomcat优化

没用的项目删掉

随机数 -Djava.security.egd=file/dev/./urandom

删除无用的jar包依赖 假如不需要websocket支持那么可以将lib文件夹下的 tomcat-websocket.jar websocker-api.jar删除

io 选择 通过protocal 进行设置 通过压测选择不同的协议支持 

connector 标签中线程池的设置 大小调整需要压测

静态分离 静态资源提前

删除无用的配置标签

* deploy的时候,默认项webapps

  ```java
  Host 里面的 startStopThreads 设置为0 可以启动当前cpu数量的线程并行启动
  autoDeploy 默认为true 定时检查appBase下是否存在更新 建议关闭 因为会存在线程检查带来额外的开销
  ```

* **Context** 

  ~~~java
  Context 代表着一个web应用 reloadable 默认为false 动态加载不建议在生产上使用
  ~~~

  默认的web.xml 中配置了 jspServerlet 现在的摸板基本上不怎么用了可以去掉

  现在由于单点登录需要将session打通那么就是将session标识放在redis中那么配置的manage就不存在作用可以删除 

  由于静态资源已经提到ngnix中或者cdn服务中那么配置的css解析器等资源也可以删除减少内存占用

`数据压缩`：

​	Connector：数据压缩配置 compression="100"  数据超过 100byte 就会进行压缩

假如：

​	(1) tomcat:web --->cpu 使用率高了怎么办?

​			top

​		原因：负载高--->线程导致 线程数量为什么多？业务代码：创建了比较多的线程，上下文频繁切换

​		GC频繁：

	> 解决：
	>
	> (1) ps -ef | grep tomcat --> pid
	>
	> (2) top  -H -p pid 察看某个进程线程使用cpu情况
	>
	> (3) 已经知道了是tomcat进程中哪个线程cpu占用率高 
	>
	> (4) jstack pid ---> 察看当前线程在干什么 
	>
	> (5) 并发编程中线程的各个状态 

### 拒绝连接

* BindException： Address already JVm bind

  端口被占用 netstat -an tomcat换个端口

* ConnectException: refused

  检查一下服务器的机器：ping ip

* SocketException: to many open files 

  文件句柄数不够 关闭无用的文件句柄 ulimit -n 10000 增大文件句柄数

### Tomcat 内存溢出

​	JVM 排查一样

### 类加载器

> bootstrapClassLoader
>
> ExtentionClassLoader
>
> AppClassLoader
>
> CustomClassLoader自定义

 tomcat 打破了双亲委派模型 加载多个web 相同的类

需要进行类隔离【web应用之间/tomcat 类和web】 以及类的共享 spring

~~~java
CommonClassLoader //通用ClassLoader
SharedClassLoader //继承CommonClassLoader共享的ClassLoader还是使用原来的双亲委派模型
WebAppClassLoader //继承SharedClassLoader 打破双亲委派模型
CatalinaCalssLoader// 继承CommonClassLoader 资源隔离
~~~

### 分层模式

* 对外

  Connector

  ​	ProtocolHandler 处理协议

  ​		Endpoint

    		AbstractEndpoint->bind()

  ​			APrEndPoint

  ​			NioEndPoint

  ​			Nio2EndPoint

* 对内

  对内所有内部组件为Container

  Engine StandardEngine

  StandardPipline

  Host     StandardHost

  HostConfig 实现了 LifecycleListener  lifecycleEvent() 根据事件实现不同的配置

  Context  StandardContext 

  ContextConfig 实现了 LifecycleListener  lifecycleEvent() 根据事件实现不同的配置

  Wapper  StandardWapper

  ContainerBase

  全部实现Container

  ------

  接收请求 CoyoteAdapt.service()

  TaskThread-run SocketProcessor#doRun()->ConnectionHandler#process()->http11Processor AbstractProcessorLight#process() service() -> CoyoteAdapt.service() ->Connector->StandardService->StandardEngine->ContainerBase->StandardPipeline->StandardEngineValve->StandardHost->ContainerBase->StandardPipeline->ErrorReportValve->ValveBase->StandardHostValve->TomcatEmbeddedContext->ContainerBase->StandardPipeline->NonLoginAuthenticator->AuthenticatorBase->ValveBase->StandardContextValve->StandardWapper->ContainerBase->StandardPipeline->StandardWapperValve->ApplicationFilterFactory->OncePerRequestFilter->CharacterEncodingFilter->FormContentFilter->RequestContextFilter->WsFilter->DispatcherServlet->HttpServlet#service(req,res)->FrameworkServlet#service(res,req)->HttpServlet#service(req,res)->Httpservlet#doGet()->FrameworkServlet#doGet()->FrameworkServlet#processRequest()->DispatcherServlet#doService()->DispatcherServlet#doDispatcher()

