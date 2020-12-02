# Mybatis源码分析

本文档依赖版本为：3.4.6，mybatis-spring版本为：1.3.2



## 配置文件解析

文件解析是Mybatis的第一个步骤，它是通过SqlSessionFactoryBuilder构建SqlSessionFactory对象，然后由XMLConfigBuilder解析XML配置

大致流程：SqlSessionFactoryBuilder ---> SqlSessionFactory ---> XMLConfigBuilder ---> ....



### 解析入口

```java
// 通过Resources加载config-xml配置
InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml"); 
// 构建SqlSessionFactory
SqlSessionFactory sqlSessionFactory =
new SqlSessionFactoryBuilder().build(inputStream);
```

```java
// step1，入口
public SqlSessionFactory build(InputStream inputStream) {
	// 调用重载方法
	return build(inputStream, null, null); 
}

// step2，解析xml生成Configuration
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
	try {
		// 创建配置文件解析器 
    XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties); 
    // 调用 parse 方法解析配置文件，生成 Configuration 对象
		return build(parser.parse());
	} catch (Exception e) {
    // ...
	}
}

// step3，通过解析xml生成的Configuration构建SqlSessionFactory
public SqlSessionFactory build(Configuration config) { 
  // 创建 DefaultSqlSessionFactory
	return new DefaultSqlSessionFactory(config);
}
```



### 解析<configuration/ >标签

注：这里的xml配置解析是指mybatis-config.xml，而非mapper.xml，请勿混淆，mybatis-config.xml内容这里不做阐述，请自行查阅

```java
// 解析mybatis-config.xml
public Configuration parse() {
	// 省略部分代码...
	// 解析config-xml配置，获取<configuration/>标签解析
  parseConfiguration(parser.evalNode("/configuration")); 
  return configuration;
}
```

```java
// 解析<configuration/>标签下所有标签
private void parseConfiguration(XNode root) { 
  try {
		// 解析 properties 配置 
    // 如果属性名一致，则会覆盖，原因是因为mybatis先加载config.xml里的properties配置
    // 然后再加载application.properties配置
  	propertiesElement(root.evalNode("properties"));
		// 解析 settings 配置，并将其转换为 Properties 对象 
 	 	Properties settings = settingsAsProperties(root.evalNode("settings")); 
  	// 加载 vfs
		loadCustomVfs(settings);
		// 解析 typeAliases 配置 
  	typeAliasesElement(root.evalNode("typeAliases"));
  	// 解析 plugins 配置 
  	pluginElement(root.evalNode("plugins"));
		// 解析 objectFactory 配置 
  	objectFactoryElement(root.evalNode("objectFactory"));
  	// 解析 objectWrapperFactory 配置 
		objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
		// 解析 reflectorFactory 配置 
  	reflectorFactoryElement(root.evalNode("reflectorFactory"));
		// settings 中的信息设置到 Configuration 对象中 
  	settingsElement(settings);
		// 解析 environments 配置 
  	environmentsElement(root.evalNode("environments"));
		// 解析 databaseIdProvider，获取并设置 databaseId 到 Configuration 对象
  	databaseIdProviderElement(root.evalNode("databaseIdProvider"));
  	// 解析 typeHandlers 配置 
  	typeHandlerElement(root.evalNode("typeHandlers")); 
  	// 解析 mappers 配置
  	mapperElement(root.evalNode("mappers"));
	} catch (Exception e) {
		throw new BuilderException("......");
	} 
}
```



### 解析Mapper.xml

解析我们编写的各个sql-mapper.xml文件，这些mapper.xml需要配置在mybatis-config.xml的mappers标签中

```xml
<configuration>
  <mappers>
		<mapper resource="../test.xml"/>
  </mappers>
</configuration>
```

或者通过**@MapperScan(basepackages="/com.test.mapper")** 配置扫描的mapper所在包，**注：此注解在mybatis-spring中，是集成Spring使用的**



#### 入口

解析逻辑mappers的逻辑在 XMLConfigBuilder的mapperElement方法中，此方法会根据配置来决定使用哪一种方式来加载配置文件，如url、class类、resource本地文件



#### Mapper配置方式

```xml
<mappers>
  // resource本地文件（一般使用这种）
  <mapper resource="org/mybatis/mappers/UserMapper.xml"/>
  // url
  <mapper url="file:///var/mappers/ProductMapper.xml"/>
  // class
  <mapper class="org.mybatis.mappers.UserMapper"/>
  // mapper所在包
  <package name="org.mybatis.mappers"/>
</mappers>
```



#### 加载Mapper配置

```java
private void mapperElement(XNode parent) throws Exception {
	if (parent != null) {
  	for (XNode child : parent.getChildren()) { 
 	   if ("package".equals(child.getName())) {
				// 根据mapper接口所在包的全路径获取
				String mapperPackage = child.getStringAttribute("name");
  	  	configuration.addMappers(mapperPackage);
			} else {
				// 获取 resource/url/class 等属性
				String resource = child.getStringAttribute("resource"); 
   	   	String url = child.getStringAttribute("url");
				String mapperClass = child.getStringAttribute("class");
				// resource 不为空，且其他两者为空，则从指定路径中加载配置
				if (resource != null && url == null && mapperClass == null){
					ErrorContext.instance().resource(resource); 
    	    InputStream inputStream =	Resources.getResourceAsStream(resource); 
     	 	  XMLMapperBuilder	mapperParser = new XMLMapperBuilder(inputStream, configuration, resource,	configuration.getSqlFragments()); 
       	 // 解析映射文件
					mapperParser.parse();
				// url 不为空，且其他两者为空，则通过 url 加载配置 
     	 } else if (resource == null && url != null && mapperClass == null) { 						
     	   ErrorContext.instance().resource(url);
					InputStream inputStream = Resources.getUrlAsStream(url); 
     	   XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments()); 
     	   // 解析映射文件
     	   mapperParser.parse();
      	 // mapperClass不为空，且其他两者为空， // 则通过mapperClass解析映射配置
				} else if (resource == null && url == null && mapperClass != null) { 
     	   Class<?> mapperInterface = Resources.classForName(mapperClass); 
     	   configuration.addMapper(mapperInterface);
				// 以上条件不满足，则抛出异常 
     	 } else {
					throw new BuilderException("......"); 
    	  }
    	} 
  	}
	} 
}
```



#### 解析Mapper.XML

接下来是mapper文件的解析过程，在mapper-config.xml配置解析源码中可以看出"resource"和"url"方式加载mapper.xml的配置都是通过XMLMapperBuilder.parse()来对mapper.xml文件进行解析的，下面来看下具体的实现

```java
// mapper.xml文件解析入口
public void parse() {
  // 判断此mapper.xml是否已被加载过
   if (!configuration.isResourceLoaded(resource)) {
     // 加载<mappers/>标签下的所有<mapper>标签配置
     configurationElement(parser.evalNode("/mapper"));
     // 添加到已加载过的列表中，HashSet存储
     configuration.addLoadedResource(resource);
     // 将mapper接口和mapper.xml中的namespace绑定
     bindMapperForNamespace();
   }
   // 处理未解析完成的节点
   parsePendingResultMaps();
   parsePendingCacheRefs();
   parsePendingStatements();
}
```

从上述源码中可以得知文件解析主要分为三个步骤：

- 解析mapper.xml文件
- 绑定mapper和namespace
- 处理未解析完成的节点/标签



下面分别介绍这三个步骤



##### 解析mapper.xml文件

mapper.xml包含很多二级标签，如reusltmap、sql、select、delete、insert、update等。三级标签，如include、if等，本章节只会挑用几个常用的标签进行分析，其他标签可以自行去官网或者其他渠道查阅解析原理

从【解析Mapper.XML】章节源码中可以看到解析mapper.xml是通过configurationElement()方法处理的，来看下实现

```java
// 解析mapper.xml
private void configurationElement(XNode context) {
    try {
      // 获取namespace，如：com.mapper.TestMapper
      String namespace = context.getStringAttribute("namespace");
      if (namespace == null || namespace.equals("")) {
        throw new BuilderException("Mapper's namespace cannot be empty");
      }
      // 设置当前namespace
      builderAssistant.setCurrentNamespace(namespace);
      // 解析cache-ref
      cacheRefElement(context.evalNode("cache-ref"));
      // 解析cache
      cacheElement(context.evalNode("cache"));
      // 已废弃，不做分析
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      // 解析resultmap
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      // 解析sql
      sqlElement(context.evalNodes("/mapper/sql"));
      // 解析sql语句
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
    }
}
```



##### 绑定Mapper和namespace

绑定mapper和namespace的目的是为了将mapper.xml中的sql语句与mapper中的方法映射，后续可以直接通过mapper直接调用方法，在【解析Mapper.XML】中可以看到入口在buildMapperForNamespace()方法

```java
private void bindMapperForNamespace() {
	// 获取映射文件的命名空间
	String namespace = builderAssistant.getCurrentNamespace(); 
  if (namespace != null) {
		Class<?> boundType = null; 
    try {
			// 获取mapper Class，如:TestMapper
			boundType = Resources.classForName(namespace); 
  	} 
  	catch (ClassNotFoundException e) {}
		if (boundType != null) {
      // 检测当前 mapper 类是否被绑定过，判断knownMappers中有没有
      if (!configuration.hasMapper(boundType)) {
        configuration.addLoadedResource("namespace:" + namespace); 
        // 绑定mapper
        configuration.addMapper(boundType);
      } 
    }
	} 
}
```

```java
// 此方法是MapperRegistry的，并非Configuration的addMapper方法
// 此方法是被configuration.addMapper中调用
public <T> void addMapper(Class<T> type) {
	if (type.isInterface()) { 
    if (hasMapper(type)) {
			throw new BindingException("......"); 
    }
    // 是否绑定完成
    boolean loadCompleted = false; 
    try {
			// 将 type 和 MapperProxyFactory 进行绑定，放在knownMappers（HashMap）中
			// MapperProxyFactory可通过type生成代理类
      knownMappers.put(type, new MapperProxyFactory<T>(type));
			// 创建注解解析器。在 MyBatis 中，有 XML 和 注解两种配置方式可选 
      // 所以mybatis是支持注解和xml同时使用的
      MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type); 
      // 解析注解中的信息 流程差不多，这里不做详细阐释，可自行查阅
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

以上就是mapper和namespace绑定的原理，大致分为三步

- 获取namespace，并通过namespace获取到对应的Mapper类
- 将Mapper类生成对应的MapperProxyFactory并保存在knownMappers（HashMap）中，以便后续使用
- 解析注解中的信息



##### 处理未解析完成的标签

举个例子：

```xml
<!-- 映射文件1 -->
<mapper namespace="xyz.coolblog.dao.Mapper1">
	<!-- 引用映射文件 2 中配置的缓存 -->
	<cache-ref namespace="xyz.coolblog.dao.Mapper2"/> 
</mapper>

<!-- 映射文件2 -->
<mapper namespace="xyz.coolblog.dao.Mapper2">
    <cache/>
</mapper>
```

例子中有2个mapper.xml，xml1引用了xml2的缓存配置，假如首先解析xml1，那么在解析cache-ref标签时发现xml2还未解析，所以这时候不能再继续进行解析流程，会抛异常，在catch处理中，会将解析异常的xml缓存在各个LinkedList中，以便在后续解析mapperXML时继续对此部分xml进行解析

具体的处理流程这里不做阐述，可自行查阅源码



### SQL执行流程

整合Spring使用@Autowired或@Resource或通过ApplicationContext获取Bean使用Mapper，这里不做阐述

单独使用Mybatis一般使用以下方式获取Mapper执行SQL

```java
// 伪代码
SqlSession session = SqlSessionFactory.openSession();
ArticleMapper articleMapper = session.getMapper(ArticleMapper.class); 
Article article = articleMapper.findOne(1);
```



##### 创建Mapper代理

从DefualtSqlSession.getMapper()开始

```java
// DefaultSqlSession.getMapper()
public <T> T getMapper(Class<T> type) {
	return configuration.<T>getMapper(type, this); 
}
// Configuration.getMapper()
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
	return mapperRegistry.getMapper(type, sqlSession);
}

// MapperRegistry.getMapper()
public <T> T getMapper(Class<T> type, SqlSession sqlSession) { 
  // 从 knownMappers 中获取与 type 对应的 MapperProxyFactory 
  // knownMappers的值什么时候放进去的在之前的文档有有介绍
  final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type); 
  if (mapperProxyFactory == null) {
		throw new BindingException("......"); 
  }
	try {
    // 创建代理对象
    return mapperProxyFactory.newInstance(sqlSession); 
  } catch (Exception e) {
		throw new BindingException("......"); 
  }
}
```

上述代码主要是通过Mapper.class获取到对应的MapperProxyFactory，然后创建Mapper.class对应的代理对象，接下来看代理对象如何生成的

```java
// MapperProxyFactory.newInstance()
public T newInstance(SqlSession sqlSession) {
  // 创建 MapperProxy 对象，MapperProxy 实现了 InvocationHandler 接口，很明显是JDK代理
  final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache); 
  return newInstance(mapperProxy);
}

// MapperProxyFactory.newInstance()
protected T newInstance(MapperProxy<T> mapperProxy) {
  // 通过 JDK 动态代理创建代理对象
  return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(),new Class[]{mapperInterface}, mapperProxy);
}
```

MapperProxy实现了InvocationHandler，重写了invoke方法，接下来看下MapperProxy重写Invoke之后做了哪些事



##### 执行代理逻辑



##### 创建MapperMethod

```java
// MapperProxy.invoke()
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  try {
    // 如果调用的方法是声明在Object（class类，非Interface）中，则直接调用
    if (Object.class.equals(method.getDeclaringClass())) {
      return method.invoke(this, args);
      // 这里是处理JDK8的Interface.default方法，此处不做分析
      // https://github.com/mybatis/mybatis-3/issues/709 有兴趣的可以去看看
    } else if (isDefaultMethod(method)) {
      return invokeDefaultMethod(proxy, method, args);
    }
  } catch (Throwable t) {
    throw ExceptionUtil.unwrapThrowable(t);
  }
  // 从缓存中（type=map,field=methodCache）获取，如果缓存中不存在则会创建一个
  // 放缓存是为了不重复创建，减少开销
  // 此处也是我们平常正常使用的方法的代理调用入口
  final MapperMethod mapperMethod = cachedMapperMethod(method);
  return mapperMethod.execute(sqlSession, args);
}

// 获取MapperMethod，MapperProxy.cachedMapperMethod()
private MapperMethod cachedMapperMethod(Method method) {
  MapperMethod mapperMethod = methodCache.get(method);
  if (mapperMethod == null) {
    // 创建mapperMethod
    // mapperMethod中包含SqlCommand和MethodSignature2个字段，下面会做简单介绍
    mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
    methodCache.put(method, mapperMethod);
  }
  return mapperMethod;
}

// 主要存储方法和sql类型
public static class SqlCommand {
	// 存储方法名，如：com.demo.getUser()
  private final String name;
  // 存储类型，如：SELECT,DELETE等
  private final SqlCommandType type;
  // ...
}

// 主要存储sql语句的相关属性，如返回值,参数列表等
// 目的是为了避免返回类型错误导致异常
public static class MethodSignature {
  // 是否返回集合或数组
  private final boolean returnsMany;
  // 是否返回Map
  private final boolean returnsMap;
  // 是否无返回值
  private final boolean returnsVoid;
  // 是否以构造方法返回
  private final boolean returnsCursor;
  // 返回类型
  private final Class<?> returnType;
  private final String mapKey;
  private final Integer resultHandlerIndex;
  private final Integer rowBoundsIndex;
  // 参数列表
  private final ParamNameResolver paramNameResolver;
  // ...
}
```



##### 执行execute()

到这里，MapperMethod创建之后，接下来是调用execute()方法来执行相关sql，来看下execute()做了些什么

```java
public Object execute(SqlSession sqlSession, Object[] args) {
  // 返回结果
  Object result;
  switch (command.getType()) {
    // INSERT|UPDATE|DELETE直接略过
    case INSERT: {
    Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.insert(command.getName(), param));
      break;
    }
    case UPDATE: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.update(command.getName(), param));
      break;
    }
    case DELETE: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.delete(command.getName(), param));
      break;
    }
    case SELECT:
      // 如果返回void，但是参数列表中有ResultHandler
      // 表示不想通过result获取结果，想通过ResultHandler获取结果
      if (method.returnsVoid() && method.hasResultHandler()) {
        executeWithResultHandler(sqlSession, args);
        result = null;
      // 下面是处理数组|集合|Map|Cursor
      } else if (method.returnsMany()) {
        result = executeForMany(sqlSession, args);
      } else if (method.returnsMap()) {
        result = executeForMap(sqlSession, args);
      } else if (method.returnsCursor()) {
        result = executeForCursor(sqlSession, args);
      } else {
        // 返回单个结果
        Object param = method.convertArgsToSqlCommandParam(args);
        result = sqlSession.selectOne(command.getName(), param);
      }
      break;
    case FLUSH:
      result = sqlSession.flushStatements();
      break;
    default:
      throw new BindingException("Unknown execution method for: " + command.getName());
  }
  // 如果result为空，有返回值且返回类型为基本类型，则抛异常
  if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
    throw new BindingException("Mapper method '" + command.getName() 
        + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
  }
  return result;
}
```



### 缓存

Mybatis提供了一级和二级缓存功能，在查询结果前会先访问一级缓存（如果开启了二级缓存会优先访问二级缓存，缓存未命中才会访问一级缓存），如果缓存未命中则会访问数据库并将结果缓存

缓存的好处是提高查询效率，同一查询条件不必重复查询数据库，减少开销，提升性能，但缓存使用不当会有事务和数据的问题，接下来会介绍



#### Cache

Cache是缓存顶级接口，定义了一些基本的操作，所有的缓存实现都基于Cache实现，Mybatis提供了基本功能的PerpetualCache、LRU策略的LruCache、保证线程安全的SynchronizedCache、阻塞的BlockingCache等等，除此之外还可以自己实现缓存

此外，Mybatis的缓存使用了装饰器模式

这里就不介绍PerpetualCache了，这是最简单的缓存实现，本章仅介绍比较有特点的LruCache、FIFOCache、BlockingCache



##### LruCache

移除最少使用的缓存，Mybatis通过LinkedHashMap和重写其removeEldestEntry()方法实现该策略，最多缓存1024个数据



```java
public class LruCache implements Cache {

  // 被装饰缓存类
  private final Cache delegate;
  // 缓存数据集合
  private Map<Object, Object> keyMap;
  // 最近最少使用的缓存key
  // 或者说是将要被移除的key
  private Object eldestKey;

  public LruCache(Cache delegate) {
    this.delegate = delegate;
    // 最多缓存1024份数据
    setSize(1024);
  }

  @Override
  public String getId() {
    return delegate.getId();
  }

  @Override
  public int getSize() {
    return delegate.getSize();
  }
	
  // 重点在这里
  public void setSize(final int size) {
    // 使用LinkedHashMap，因为LinkedHashMap可以保证数据的顺序（注：accessOrder=true）
    // 此外，重写removeEldestEntry()，当插入数据size超出时，会获取最少使用的key设置给eldestKey
    // 当调用LruCache.putObject()方法时会通过LruCache.cycleKeyList()方法将需要移除的key移除掉
    // 具体细节原理会在下面介绍
    // LinkedHashMap通过get()来刷新被访问的数据移动到链表尾部
		// 这样，长时间未访问过的数据会在链表的头部，具体的实现原理可以自行查阅LinkedHashMap源码
    keyMap = new LinkedHashMap<Object, Object>(size, .75F, true) {
      private static final long serialVersionUID = 4267176411845948333L;

      @Override
      protected boolean removeEldestEntry(Map.Entry<Object, Object> eldest) {
        // put数据，当size不够时，会获取最少使用的数据，并设置给eldestKey
        boolean tooBig = size() > size;
        if (tooBig) {
          eldestKey = eldest.getKey();
        }
        return tooBig;
      }
    };
  }

  @Override
  public void putObject(Object key, Object value) {
    // 缓存数据，重点是cycleKeyList()方法，原理在该方法介绍
    delegate.putObject(key, value);
    cycleKeyList(key);
  }

  @Override
  public Object getObject(Object key) {
    // 这里调用get方法目的是为了刷新此key的位置到链表尾部
    // 这样一来，长时间未访问过的key就会再链表的头部
    keyMap.get(key);
    return delegate.getObject(key);
  }

  @Override
  public Object removeObject(Object key) {
    return delegate.removeObject(key);
  }

  @Override
  public void clear() {
    delegate.clear();
    keyMap.clear();
  }

  @Override
  public ReadWriteLock getReadWriteLock() {
    return null;
  }
	
  // 这里是重点
  private void cycleKeyList(Object key) {
    // 缓存key，目的是为了获取需要移除的key，这里分2种情况
    // 1.当put数据时，size长度足够，就获取不到需要移除的key，则不需要移除
    // 2.当size长度不够时就会获取到需要移除的key，eldestKey!=null成立
    keyMap.put(key, key);
    if (eldestKey != null) {
      // 置空eldestKey，将被装饰类的对应key缓存移除
      delegate.removeObject(eldestKey);
      eldestKey = null;
    }
  }
}
```

以上就是Mybatis中LRU的原理，值得注意的是从始至终keyMap是为了LRU策略服务的，真正的缓存操作还是基于被装饰类。





