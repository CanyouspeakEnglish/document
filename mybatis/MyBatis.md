# MyBatis

### 官方文档

https://mybatis.org/mybatis-3/zh/sqlmap-xml.html

##### 1.sqlSessionFactoryBean

![image-20200508150627269](C:\Users\lzx\AppData\Roaming\Typora\typora-user-images\image-20200508150627269.png)

##### 1.InitializingBean-interface

###### 	1.1 方法afterPropertiesSet() 初始化方法	

```java
public void afterPropertiesSet() throws Exception {
  notNull(dataSource, "Property 'dataSource' is required");
  notNull(sqlSessionFactoryBuilder, "Property 'sqlSessionFactoryBuilder' is required");
  this.sqlSessionFactory = buildSqlSessionFactory();
}
```

```java
//以Springboot为例因为不是以xml为配置 去掉一些无用代码
protected SqlSessionFactory buildSqlSessionFactory() throws IOException {
   Configuration configuration;
   configuration = new Configuration();
   configuration.setVariables(this.configurationProperties);
  //Spring事务工厂
  if (this.transactionFactory == null) {
    this.transactionFactory = new SpringManagedTransactionFactory();
  }
 //设置环境变量
  configuration.setEnvironment(new Environment(this.environment, this.transactionFactory, this.dataSource));
  return this.sqlSessionFactoryBuilder.build(configuration);
}
```

##### sqlSessionFactoryBuilder--构建者

```java
//赋值给sqlSessionFactory
public SqlSessionFactory build(Configuration config) {
  return new DefaultSqlSessionFactory(config);
}
```

##### factoryBean--工厂bean

```java
public SqlSessionFactory getObject() throws Exception {
  if (this.sqlSessionFactory == null) {
    afterPropertiesSet();
  }

  return this.sqlSessionFactory;
}
```

##### 2.mapperFactoryBean--mapperFactory工厂

​	![image-20200508165049006](C:\Users\lzx\AppData\Roaming\Typora\typora-user-images\image-20200508165049006.png)

###### 	2.1 daoSupport--实现了InitializingBean

###### 	 	方法afterPropertiesSet()

```java
//
public final void afterPropertiesSet() throws IllegalArgumentException, BeanInitializationException {
    //检查dao 如果是mapper就注册 1
   checkDaoConfig();
   //此处是一个空实现 留给自己实现在解析之前
   initDao();
    .......
}
```

###### 	方法checkDaoConfig()

```java
//sqlSessionDaoSupport 中的方法 2
protected void checkDaoConfig() {
  //检查sqlSession 不能为空
  notNull(this.sqlSession, "Property 'sqlSessionFactory' or 'sqlSessionTemplate' are required");
}
```

```java
//mapperFactoryBean 中的方法 3
protected void checkDaoConfig() {
  super.checkDaoConfig();
  //mapperInterface不能为空
  notNull(this.mapperInterface, "Property 'mapperInterface' is required");
  Configuration configuration = getSqlSession().getConfiguration();
    //加入配置开启 并且 没有解析过
  if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
       //添加mapperInterface并且解析
      configuration.addMapper(this.mapperInterface);
      .....
  }
}
```

###### configuration--配置类 --single

```java
public <T> void addMapper(Class<T> type) {
  mapperRegistry.addMapper(type);
}
```

###### 	mapperRegistry--注册类

###### 	构造方法

```java
//config为当前配置类this
public MapperRegistry(Configuration config) {
  this.config = config;
}
```

```java
public <T> void addMapper(Class<T> type) {
  if (type.isInterface()) {
      //已经包含了 直接抛出绑定异常
    if (hasMapper(type)) {
      throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
    }
    boolean loadCompleted = false;
    try {
      //先构造mapperProxyFactory 传入接口类型
      knownMappers.put(type, new MapperProxyFactory<T>(type));
      //进行解析 传入配置和接口
      MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
      parser.parse();
      loadCompleted = true;
    } finally {
      if (!loadCompleted) {
        knownMappers.remove(type);
      }
    }
  }
}
```

###### 	mapperAnnotationBuilder--注解解析类

###### 		构造方法

```java
public MapperAnnotationBuilder(Configuration configuration, Class<?> type) {
  //将类的全路径转换成正常格式
  String resource = type.getName().replace('.', '/') + ".java (best guess)";
  //mapper解析辅助类
  this.assistant = new MapperBuilderAssistant(configuration, resource);
  this.configuration = configuration;
  this.type = type;
  //添加注解类型
  sqlAnnotationTypes.add(Select.class);
  sqlAnnotationTypes.add(Insert.class);
  sqlAnnotationTypes.add(Update.class);
  sqlAnnotationTypes.add(Delete.class);
  //添加注解类型
  sqlProviderAnnotationTypes.add(SelectProvider.class);
  sqlProviderAnnotationTypes.add(InsertProvider.class);
  sqlProviderAnnotationTypes.add(UpdateProvider.class);
  sqlProviderAnnotationTypes.add(DeleteProvider.class);
}
```

###### 		方法parse()

​		

```java
public void parse() {
  String resource = type.toString();
   //首先检查是否已经加载过 loadedResources中 是否包含resource 为set集合
  if (!configuration.isResourceLoaded(resource)) {
    //不管是不是xml文件都会按原路径解析一下 这就是为什么接口和xml文件要放到一个包下
    loadXmlResource();
    //添加到已经解析的集合内
    configuration.addLoadedResource(resource);
    //设置当前的解析助手的命名空间
    assistant.setCurrentNamespace(type.getName());
    //配置CacheNameSpace注解进行刷新
    parseCache();
    //配置CacheNameSpaceRef注解查询cache
    parseCacheRef();
    //查找接口中的方法
    Method[] methods = type.getMethods();
    for (Method method : methods) {
      try {
          //方法是否为泛型方法
        if (!method.isBridge()) {
          //解析Statement
          parseStatement(method);
        }
      } catch (IncompleteElementException e) {
        configuration.addIncompleteMethod(new MethodResolver(this, method));
      }
    }
  }
  parsePendingMethods();
}
```

###### 		方法loadXmlResource()

```java
private void loadXmlResource() {
  if (!configuration.isResourceLoaded("namespace:" + type.getName())) {
    String xmlResource = type.getName().replace('.', '/') + ".xml";
    InputStream inputStream = null;
    try {
      inputStream = Resources.getResourceAsStream(type.getClassLoader(), xmlResource);
    } catch (IOException e) {
     //不存在不处理
    }
    //如果存在文件进行文件解析
    if (inputStream != null) {
      XMLMapperBuilder xmlParser = new XMLMapperBuilder(inputStream, assistant.getConfiguration(), xmlResource, configuration.getSqlFragments(), type.getName());
      xmlParser.parse();
    }
  }
}
```

###### 		方法parseStatement()

```java
void parseStatement(Method method) {
  //获取方法参数类型（不是RowBounds和ResultHandle的派生类）参数为null赋值为ParamMap--MapperMethod中的静态类
  Class<?> parameterTypeClass = getParameterType(method);
  //默认为XMLLanguageDriver类 如果指定在LanguageDriverRegistry中注册@Lang
  LanguageDriver languageDriver = getLanguageDriver(method);
  SqlSource sqlSource = getSqlSourceFromAnnotations(method, parameterTypeClass, languageDriver);
  if (sqlSource != null) {
    Options options = method.getAnnotation(Options.class);
    //key 名为类名.方法名
    final String mappedStatementId = type.getName() + "." + method.getName();
    Integer fetchSize = null;
    Integer timeout = null;
    //默认的statementType为prepared
    StatementType statementType = StatementType.PREPARED;
    //resultSet 类型为 forward_only
    ResultSetType resultSetType = ResultSetType.FORWARD_ONLY;
    //sql 命令类型  insert update delete select
    SqlCommandType sqlCommandType = getSqlCommandType(method);
    //判断是否为查询
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
    //不是查询缓存
    boolean flushCache = !isSelect;
    //查询使用缓存
    boolean useCache = isSelect;

    KeyGenerator keyGenerator;
    String keyProperty = "id";
    String keyColumn = null;
    //如果是插入或者更新
    if (SqlCommandType.INSERT.equals(sqlCommandType) || SqlCommandType.UPDATE.equals(sqlCommandType)) {
      // 检查是否含有selectKey 指定主键类型
      SelectKey selectKey = method.getAnnotation(SelectKey.class);
      if (selectKey != null) {
        //返回自增keyGenerator
        keyGenerator = handleSelectKeyAnnotation(selectKey, mappedStatementId, getParameterType(method), languageDriver);
        keyProperty = selectKey.keyProperty();
      } else if (options == null) {//options是否为空 是否生产主键 是 jdbc 否 no
        keyGenerator = configuration.isUseGeneratedKeys() ? new Jdbc3KeyGenerator() : new NoKeyGenerator();
      } else {
        keyGenerator = options.useGeneratedKeys() ? new Jdbc3KeyGenerator() : new NoKeyGenerator();
       //生成key的属性
        keyProperty = options.keyProperty();
        //生成key的列
        keyColumn = options.keyColumn();
      }
    } else {
      //不生成key
      keyGenerator = new NoKeyGenerator();
    }
	//options不为空
    if (options != null) {
      //是否刷新缓存
      flushCache = options.flushCache();
      //是否使用缓存
      useCache = options.useCache();
      //这是一个给驱动的建议值，尝试让驱动程序每次批量返回的结果行数等于这个设置值。 默认值为未设置（unset）（依赖驱动）
      fetchSize = options.fetchSize() > -1 || options.fetchSize() == Integer.MIN_VALUE ? options.fetchSize() : null;
      timeout = options.timeout() > -1 ? options.timeout() : null;
      //MyBatis 支持 STATEMENT，PREPARED 和 CALLABLE 类型的映射语句，分别代表 Statement, PreparedStatement 和 CallableStatement 类型
      statementType = options.statementType();
      //FORWARD_ONLY，SCROLL_SENSITIVE, SCROLL_INSENSITIVE 或 DEFAULT（等价于 unset） 中的一个，默认值为 unset （依赖数据库驱动）
      resultSetType = options.resultSetType();
    }

    String resultMapId = null;
    ResultMap resultMapAnnotation = method.getAnnotation(ResultMap.class);
    if (resultMapAnnotation != null) {
      String[] resultMaps = resultMapAnnotation.value();
      StringBuilder sb = new StringBuilder();
      for (String resultMap : resultMaps) {
        if (sb.length() > 0) {
          sb.append(",");
        }
        sb.append(resultMap);
      }
      resultMapId = sb.toString();
    } else if (isSelect) {
      resultMapId = parseResultMap(method);
    }

    assistant.addMappedStatement(
        mappedStatementId,
        sqlSource,
        statementType,
        sqlCommandType,
        fetchSize,
        timeout,
        // ParameterMapID
        null,
        parameterTypeClass,
        resultMapId,
        getReturnType(method),
        resultSetType,
        flushCache,
        useCache,
        // TODO issue #577
        false,
        keyGenerator,
        keyProperty,
        keyColumn,
        // DatabaseID
        null,
        languageDriver,
        // ResultSets
        null);
  }
}
```

###### 			方法getSqlSourceFromAnnotations()

```java
//从注解中获取sqlsource 1 当前方法 2 参数类型 （如果多个的话其实就是返回一个ParamMap）
private SqlSource getSqlSourceFromAnnotations(Method method, Class<?> parameterType, 
                                              //解析器 可以自定义
                                              LanguageDriver languageDriver) {
  try {
    //获取注解类型从你的方法中 Select,Insert,Update,Delete
    Class<? extends Annotation> sqlAnnotationType = getSqlAnnotationType(method);
    //获取Provider注解类型从你的方法中SelectProvider,InsertProvider,UpdateProvider,DeleteProvider
    Class<? extends Annotation> sqlProviderAnnotationType = getSqlProviderAnnotationType(method);
    if (sqlAnnotationType != null) {
      if (sqlProviderAnnotationType != null) {
        throw new BindingException("You cannot supply both a static SQL and SqlProvider to method named " + method.getName());
      }
      //获取方法上的注解
      Annotation sqlAnnotation = method.getAnnotation(sqlAnnotationType);
      //获取值
      final String[] strings = (String[]) sqlAnnotation.getClass().getMethod("value").invoke(sqlAnnotation);
      return buildSqlSourceFromStrings(strings, parameterType, languageDriver);
    } else if (sqlProviderAnnotationType != null) {
      Annotation sqlProviderAnnotation = method.getAnnotation(sqlProviderAnnotationType);
      return new ProviderSqlSource(assistant.getConfiguration(), sqlProviderAnnotation);
    }
    return null;
  } catch (Exception e) {
    throw new BuilderException("Could not find value method on SQL annotation.  Cause: " + e, e);
  }
}
```

```java
//构建sqlSource
private SqlSource buildSqlSourceFromStrings(String[] strings, Class<?> parameterTypeClass, LanguageDriver languageDriver) {
  final StringBuilder sql = new StringBuilder();
  //sql 语句
  for (String fragment : strings) {
    sql.append(fragment);
    sql.append(" ");
  }
  return languageDriver.createSqlSource(configuration, sql.toString().trim(), parameterTypeClass);
}
```