---
title: spring 源码解析 2.统一io ResourceLoader 
date: 2017-05-06 21:04:01
tags:
- spring container
categories:
- spring
- code
---
ResourceLoader参考文档：<http://docs.spring.io/spring/docs/5.0.0.M5/spring-framework-reference/htmlsingle/#resources-resourceloader>

## 1.概述
ResourceLoader接口的实现旨在返回（加载）Resource 实例。
## 2.类图
![](/assets/img/spring/springResourceLoader.png)
## 3.接口
### 3.1 [ResourceLoader](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/ResourceLoader.java)

```java
/**
 * 加载资源的策略接口（e ..类路径或文件系统资源）。
 * 需要一个{@link org.springframework.context.ApplicationContext}来提供此功能，以及扩展的{@link org.springframework.core.io.support.ResourcePatternResolver}支持。
 * {@link DefaultResourceLoader}是一个独立的实现，可以在ApplicationContext之外使用，也由{@link ResourceEditor}使用。
 * 在ApplicationContext中运行时，可以使用特定上下文的资源加载策略，从字符串填充Resource类型的Bean属性和Resource数组。
 */
public interface ResourceLoader {
	/**
	 * 从类路径加载的伪URL前缀："classpath:"
	 */
	String CLASSPATH_URL_PREFIX = ResourceUtils.CLASSPATH_URL_PREFIX;


	/**
	 * 返回指定资源位置的Resource句柄。
	 * 句柄应该始终是可重用的资源描述符，允许多个{@link Resource#getInputStream()}调用。
	 * 必须支持完全限定的URLs，例如 "file:C:/test.dat"。
	 * 必须支持类路径伪URL，例如 "classpath:test.dat"。
	 * 应该支持相对文件路径，例如 "WEB-INF/test.dat"。
	 *（这将是实现特定的，通常由ApplicationContext实现提供）
	 * 请注意，Resource句柄并不表示资源存在; 您需要调用 {@link Resource#exists} 来检查是否存在。
	 */
	Resource getResource(String location);

	/**
	 * 暴露此ResourceLoader使用的ClassLoader。
	 * 需要直接访问ClassLoader的客户端可以使用ResourceLoader以统一的方式执行，而不是依赖线程上下文ClassLoader。
	 */
	ClassLoader getClassLoader();
}
```

### 3.2 [ResourcePatternResolver](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/support/ResourcePatternResolver.java)
```java
/**
 * 用于将位置格式（例如，Ant样式路径格式）解析为Resource对象的策略接口。
 * 这是{@link org.springframework.core.io.ResourceLoader}接口的扩展。可以检查传递的ResourceLoader是否实现此扩展接口（例如，在上下文中运行时通过{@link org.springframework.context.ResourceLoaderAware}传递的{@link org.springframework.context.ApplicationContext}）。
 * {@link PathMatchingResourcePatternResolver}是一个独立的实现，可在ApplicationContext外部使用，也由{@link ResourceArrayPropertyEditor}用于填充资源阵列bean属性。
 * 可以与任何种类的位置模板一起使用（例如“/WEB-INF/*-context.xml”）：输入模板必须与策略实现相匹配。该接口只是指定转换方法而不是特定的模板格式。
 * 此接口还为类路径中的所有匹配资源建议一个新的资源前缀 "classpath*:"。
 * 请注意，在这种情况下，资源位置预计是没有占位符的路径（例如“/beans.xml”）;JAR文件或类目录可以包含同名的多个文件。
 */
public interface ResourcePatternResolver extends ResourceLoader {

	/**
	 * 来自类路径的所有匹配资源的伪URL前缀："classpath*:"
	 * 与ResourceLoader的类路径URL前缀不同之处在于它检索给定名称的所有匹配资源（例如“/beans.xml”），例如在所有已部署的JAR文件的根目录中。
	 */
	String CLASSPATH_ALL_URL_PREFIX = "classpath*:";

	/**
	 * 将给定的位置模板解析为资源对象。
	 * 应尽可能避免重叠指向相同物理资源的资源条目。结果应该设置语义。
	 */
	Resource[] getResources(String locationPattern) throws IOException;

}

```

## 4. 抽象类、实现
### 4.1 [DefaultResourceLoader](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/DefaultResourceLoader.java)
```java
/**
 * {@link ResourceLoader}接口的默认实现。
 * 由{@link ResourceEditor}使用，并作为{@link org.springframework.context.support.AbstractApplicationContext}的基类。 也可以独立使用。
 * 如果位置值是URL，则返回{@link UrlResource}，如果它是非URL路径或 "classpath:" 伪URL，则返回{@link ClassPathResource}。
 */
public class DefaultResourceLoader implements ResourceLoader {
	private ClassLoader classLoader;
	private final Set<ProtocolResolver> protocolResolvers = new LinkedHashSet<>(4);
	private final Map<Class<?>, Map<Resource, ?>> resourceCaches = new ConcurrentHashMap<>(4);
}
```
```java
/**
 * 协议特定资源句柄的解析策略。
 * 用作{@link DefaultResourceLoader}的SPI，允许在不对载入程序实现（或应用程序上下文实现）创建子类的情况下处理自定义协议。
 */
@FunctionalInterface
public interface ProtocolResolver {
	/**
	 * 如果此实现的协议匹配，则针对给定的资源加载器解析给定位置。
	 */
	Resource resolve(String location, ResourceLoader resourceLoader);
}
```
### 4.2 [ClassRelativeResourceLoader](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/ClassRelativeResourceLoader.java)
```java
/**
 * {@link ResourceLoader}实现，将普通资源路径解释为相对于给定的{@code java.lang.Class}。
 */
public class ClassRelativeResourceLoader extends DefaultResourceLoader {
	private final Class<?> clazz;
}
```
### 4.3 [FileSystemResourceLoader](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/FileSystemResourceLoader.java)
```java
/**
 * {@link ResourceLoader}实现，它将简单路径解析为文件系统资源，而不是类路径资源（后者是{@link DefaultResourceLoader}的默认策略））。
 * 注意： 即使以斜线开始，平滑路径将始终被解释为相对于当前VM工作目录。 （这与Servlet容器中的语义一致。）
 * 使用显式的"file:"前缀强制执行绝对文件路径。
 * {@link org.springframework.context.support.FileSystemXmlApplicationContext}是一个完整的ApplicationContext实现，提供相同的资源路径解析策略。
 */
public class FileSystemResourceLoader extends DefaultResourceLoader {
}
```

### 4.4
