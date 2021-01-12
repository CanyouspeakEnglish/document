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

ps -ef | grep tomcat

cat /pro/pid/status

* 参数说明

https://blog.csdn.net/u011425939/article/details/75335079

top -p pid