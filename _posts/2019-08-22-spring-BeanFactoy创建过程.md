---
layout:     post
title:      spring-BeanFactory创建过程
subtitle:   本文为 Spring 源码阅读中的一节，主要记录了 spring 启动过程中的 beanFactory 创建过程
date:       2019-07-09
author:     NONGFAH
header-img: img/post-bg-mma-2.jpg
catalog: true
tags:
    - Spring
    - 源码
---

# spring 源码解析

## BeanFactory 的创建阶段

### 示例代码

##### main    函数

```java
public static void main(String[] args) {
    @SuppressWarnings("resource")
    ApplicationContext applicationContext = new       ClassPathXmlApplicationContext("applicationContext.xml");
    App app = (App) applicationContext.getBean("app");
    app.say();
}
```

##### App.java    示例类代码

```java
public class App {
public void say() {
    System.out.println("App.app()");
}
}
```

##### applicationContext.xml    示例配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">    
    <bean id="app" class="com.mycompany.app.App"></bean>
</beans>
```

##### pom.xml

```xml

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.mycompany.app</groupId>
  <artifactId>myapp</artifactId>
  <packaging>jar</packaging>
  <version>1.0-SNAPSHOT</version>
  <name>myapp</name>
  <url>http://maven.apache.org</url>

  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <spring.version>5.0.0.RELEASE</spring.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-core</artifactId>
      <version>${spring.version}</version>
    </dependency>
</project>
```

### 开始

![示例程序](http://nongfah.github.io/img/1565926459861.png)

14行创建 ClassPathXmlApplicationContext 实例打断点。

#### 调用顺序

##### 1.new ClassPathXmlApplicationContext（String configLocation）调用内部构造器

```java
public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
--->this(new String[] {configLocation}, true, null);
}
```

##### 2.this(new String[] {configLocation}, true, null)

```java
public ClassPathXmlApplicationContext(
		String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
		throws BeansException {
	super(parent);
    // 替换配置文件路径中的占位符,并将其保存
	setConfigLocations(configLocations); 
	if (refresh) {
        // 重点分析 refresh 方法
----->refresh();
	}
}
```

#####  3.refresh()

```java
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// 准备上下文进行刷新，准备工作包括设置启动时间，是否激活标识位，初始化属性源(property source)配置
			prepareRefresh();
			//  本次分析的重点 创建beanFactory
            //（过程是根据 xml 为每个 bean 生成 BeanDefinition 并注册到生成的 beanFactory 的 beanDefinitionMap 中）
----------->ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
           	// 其他代码.....
}
```

#####  4.obtainFreshBeanFactory()

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    // 刷新 beanFactory
--->refreshBeanFactory();
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	if (logger.isDebugEnabled()) {
		logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
	}
	return beanFactory;
}
```

#####  5.refreshBeanFactory()

```java
/**
 *官方注释 subclasses must implement this method to perform the actual configuration load.
 * 子类必须实现这个方法才能进行配置的加载
 */
protected final void refreshBeanFactory() throws BeansException {
       //如果已经存在 beanFactory ，销毁所有的 bean ，然后将该 beanFactory 关闭
	if (hasBeanFactory()) {
		destroyBeans();
		closeBeanFactory();
	}
	try {
        // 创建一个 beanfactory ，并进行初始化
		DefaultListableBeanFactory beanFactory = createBeanFactory();
		beanFactory.setSerializationId(getId());
		customizeBeanFactory(beanFactory);
           // 加载 beanDefinitions bean定义
---------->loadBeanDefinitions(beanFactory);
		synchronized (this.beanFactoryMonitor) {
			this.beanFactory = beanFactory;
		}
	}
	catch (IOException ex) {
		throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
	}
}
```

#####  6.loadBeanDefinitions(DefaultListableBeanFactory beanFactory)

```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
	// 使用 beanFactory 创建一个 xml 的 bean 定义读取器 ， 因为 beanfactory 实现了BeanDefinitionRegistry 接口,所以此处可以使用它来new 
	XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
	// 读取器的配置及初始化
	beanDefinitionReader.setEnvironment(this.getEnvironment());
	beanDefinitionReader.setResourceLoader(this);
	beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
    initBeanDefinitionReader(beanDefinitionReader);
    // 使用创建的 reader 加载 bean定义
--->loadBeanDefinitions(beanDefinitionReader);
}
```

#####  7.loadBeanDefinitions(XmlBeanDefinitionReader reader)

```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
	Resource[] configResources = getConfigResources();
	if (configResources != null) {
		reader.loadBeanDefinitions(configResources);
	}
       // 此处 getConfigLocations() 返回的就是
       // 在步骤二 setConfigLocations(configLocations)中保存的 configLocations 
       // 也就是在 main() 函数中 传入的配置文件路径参数
	String[] configLocations = getConfigLocations();
	if (configLocations != null) {
      // 使用 configLocations 加载beanDefinions 
-------->reader.loadBeanDefinitions(configLocations);
	}
}
```

##### 8.loadBeanDefinitions(String... locations)

```java
public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
	Assert.notNull(locations, "Location array must not be null");
	int counter = 0;
	for (String location : locations) {
       // 从指定的位置加载 bean 定义 ,并计数
------>counter += loadBeanDefinitions(location);
	}
	return counter;
}
```

##### 9.loadBeanDefinitions(String location)

```java
public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {
------->return loadBeanDefinitions(location, null);
}
```

##### 10.loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources)

```java
public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
		ResourceLoader resourceLoader = getResourceLoader();
		if (resourceLoader == null) {
			throw new BeanDefinitionStoreException(
					"Cannot load bean definitions from location [" + location + "]: no ResourceLoader available");
		}

		if (resourceLoader instanceof ResourcePatternResolver) {
			// Resource pattern matching available.
			try {
				Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
                // 从指定路径加载 bean 定义
--------------->int count = loadBeanDefinitions(resources);
				if (actualResources != null) {
					Collections.addAll(actualResources, resources);
				}
				if (logger.isTraceEnabled()) {
					logger.trace("Loaded " + count + " bean definitions from location pattern [" + location + "]");
				}
				return count;
			}
			catch (IOException ex) {
				throw new BeanDefinitionStoreException(
						"Could not resolve bean definition resource pattern [" + location + "]", ex);
			}
		}
		else {
			// Can only load single resources by absolute URL.
			Resource resource = resourceLoader.getResource(location);
			int count = loadBeanDefinitions(resource);
			if (actualResources != null) {
				actualResources.add(resource);
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Loaded " + count + " bean definitions from location [" + location + "]");
			}
			return count;
		}
}
```

##### 11.loadBeanDefinitions(Resource... resources)

```java
public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
	Assert.notNull(resources, "Resource array must not be null");
	int count = 0;
	for (Resource resource : resources) {
        // 从指定路径加载 bean 定义
------->count += loadBeanDefinitions(resource);
	}
	return count;
}
```

##### 12.loadBeanDefinitions(Resource resource)

```java
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
    //使用指定编码加载 bean 定义
--->return loadBeanDefinitions(new EncodedResource(resource));
}
```

##### 13.loadBeanDefinitions(EncodedResource encodedResource)

```java
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
	Assert.notNull(encodedResource, "EncodedResource must not be null");
	if (logger.isTraceEnabled()) {
		logger.trace("Loading XML bean definitions from " + encodedResource);
	}
	Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
	if (currentResources == null) {
		currentResources = new HashSet<>(4);
		this.resourcesCurrentlyBeingLoaded.set(currentResources);
	}
	if (!currentResources.add(encodedResource)) {
		throw new BeanDefinitionStoreException(
				"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
	}
	try {
		InputStream inputStream = encodedResource.getResource().getInputStream();
		try {
			InputSource inputSource = new InputSource(inputStream);
			if (encodedResource.getEncoding() != null) {
				inputSource.setEncoding(encodedResource.getEncoding());
			}
            // 加载 bean 定义
------->return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
		}
		finally {
			inputStream.close();
		}
	}
	catch (IOException ex) {
		throw new BeanDefinitionStoreException(
				"IOException parsing XML document from " + encodedResource.getResource(), ex);
	}
	finally {
		currentResources.remove(encodedResource);
		if (currentResources.isEmpty()) {
			this.resourcesCurrentlyBeingLoaded.remove();
		}
	}
}
```

##### 14.doLoadBeanDefinitions(InputSource inputSource, Resource resource)

```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
            // 加载文档
			Document doc = doLoadDocument(inputSource, resource);.
            // 注册 bean 定义
----------->int count = registerBeanDefinitions(doc, resource);
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + count + " bean definitions from " + resource);
			}
			return count;
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		catch (SAXParseException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
		}
		catch (SAXException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"XML document from " + resource + " is invalid", ex);
		}
		catch (ParserConfigurationException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Parser configuration exception parsing XML from " + resource, ex);
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"IOException parsing XML document from " + resource, ex);
		}
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Unexpected exception parsing XML document from " + resource, ex);
		}
}
```

##### 15.registerBeanDefinitions(Document doc, Resource resource)

```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
	BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
	int countBefore = getRegistry().getBeanDefinitionCount();
    // 注册 bean 定义
--->documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
	return getRegistry().getBeanDefinitionCount() - countBefore;
}
```

##### 16.registerBeanDefinitions(Document doc, XmlReaderContext readerContext)

```java
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
	this.readerContext = readerContext;
    // 使用文档注册bean
--->doRegisterBeanDefinitions(doc.getDocumentElement());
}
```

​    

##### 17.doRegisterBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate)

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
	if (delegate.isDefaultNamespace(root)) {
		NodeList nl = root.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (node instanceof Element) {
				Element ele = (Element) node;
				if (delegate.isDefaultNamespace(ele)) {
                 //循环bean.xml中定义的每一个元素
------------------>parseDefaultElement(ele, delegate);
				}
				else {
					delegate.parseCustomElement(ele);
				}
			}
		}
	}
	else {
		delegate.parseCustomElement(root);
	}
}
```

##### 18.parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate)

```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
    // 对标签<bean><import>等等，分别解析
	if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
		importBeanDefinitionResource(ele);
	}
	else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
		processAliasRegistration(ele);
	}
	else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
    // 解析 bean 标签
------->processBeanDefinition(ele, delegate);
	}
	else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
		// recurse
		doRegisterBeanDefinitions(ele);
	}
}
```

##### 19.processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate)

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
	BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
	if (bdHolder != null) {
		bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
		try {
			// 注册最终修饰的实例。
----------->BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
		}
		catch (BeanDefinitionStoreException ex) {
			getReaderContext().error("Failed to register bean definition with name '" +
					bdHolder.getBeanName() + "'", ele, ex);
		}
		// Send registration event.
		getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
	}
}
```

##### 20.registerBeanDefinition(BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)

```java
public static void registerBeanDefinition(BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
		throws BeanDefinitionStoreException {
	// 在主名称（真实名字，区别于别名）下注册bean定义
	String beanName = definitionHolder.getBeanName();
--->registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());
	// 如果有别名，注册别名
	String[] aliases = definitionHolder.getAliases();
	if (aliases != null) {
		for (String alias : aliases) {
			registry.registerAlias(beanName, alias);
		}
	}
}
```

##### 21.registerBeanDefinition(String beanName, BeanDefinition beanDefinition)

```java
@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
		throws BeanDefinitionStoreException {
	Assert.hasText(beanName, "Bean name must not be empty");
	Assert.notNull(beanDefinition, "BeanDefinition must not be null");
	if (beanDefinition instanceof AbstractBeanDefinition) {
		try {
			((AbstractBeanDefinition) beanDefinition).validate();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
					"Validation of bean definition failed", ex);
		}
	}
       // 判断系统中是否存在该 name 的 bean
	BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
      // 如果存在，检测是否允许覆盖
	if (existingDefinition != null) {
           // 不允许覆盖，抛异常
		if (!isAllowBeanDefinitionOverriding()) {
			throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
		}
		else if (existingDefinition.getRole() < beanDefinition.getRole()) {
			// e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
			if (logger.isInfoEnabled()) {
				logger.info("Overriding user-defined bean definition for bean '" + beanName +
						"' with a framework-generated bean definition: replacing [" +
						existingDefinition + "] with [" + beanDefinition + "]");
			}
		}
		else if (!beanDefinition.equals(existingDefinition)) {
			if (logger.isDebugEnabled()) {
				logger.debug("Overriding bean definition for bean '" + beanName +
						"' with a different definition: replacing [" + existingDefinition +
						"] with [" + beanDefinition + "]");
			}
		}
		else {
			if (logger.isTraceEnabled()) {
				logger.trace("Overriding bean definition for bean '" + beanName +
						"' with an equivalent definition: replacing [" + existingDefinition +
						"] with [" + beanDefinition + "]");
			}
		}
           // 覆盖之前的 bean
---------->this.beanDefinitionMap.put(beanName, beanDefinition);
	}
       // 如果 容器中 不存在该 bean ，放入容器中
	else {
           // 如果注册已经开始
		if (hasBeanCreationStarted()) {
			// Cannot modify startup-time collection elements anymore (for stable iteration)
               //因为beanDefinitionMap是全局变量，这里定会存在并发访问的情况
			synchronized (this.beanDefinitionMap) {
				this.beanDefinitionMap.put(beanName, beanDefinition);
				List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
				updatedDefinitions.addAll(this.beanDefinitionNames);
				updatedDefinitions.add(beanName);
				this.beanDefinitionNames = updatedDefinitions;
				removeManualSingletonName(beanName);
			}
		}
		else {
			// Still in startup registration phase
-------------->this.beanDefinitionMap.put(beanName, beanDefinition);
			this.beanDefinitionNames.add(beanName);
			removeManualSingletonName(beanName);
		}
		this.frozenBeanDefinitionNames = null;
	}
	if (existingDefinition != null || containsSingleton(beanName)) {
          	// 重置所有beanName对应的缓存
		resetBeanDefinition(beanName);
	}
}
```

### 过程总结
Spring BeanFactory 创建 基本流程就三步
1、创建 beanFactory 
2、解析文档生成 beanDefinition
3、将解析生成的 beanDefinition 放入 map 中保存起来为 bean 的实例化做准备
