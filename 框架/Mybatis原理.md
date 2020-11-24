# Mybatis



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



#### 配置方式

```xml
<mappers>
  // resource本地文件
  <mapper resource="org/mybatis/mappers/UserMapper.xml"/>
  // url
  <mapper url="file:///var/mappers/ProductMapper.xml"/>
  // class
  <mapper class="org.mybatis.mappers.UserMapper"/>
  // mapper所在包
  <package name="org.mybatis.mappers"/>
</mappers>
```

#### **加载Mapper配置源码**

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









