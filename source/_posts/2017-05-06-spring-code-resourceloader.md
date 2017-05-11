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

### 4.4 [PathMatchingResourcePatternResolver](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/support/PathMatchingResourcePatternResolver.java)
```java
/**
 * 一个{@link ResourcePatternResolver}实现，能够将指定的资源位置路径解析为一个或多个匹配的资源。
 * 源路径可以是一个简单的路径，它具有与目标{@link org.springframework.core.io.Resource}的一对一映射，或者可以包含特殊的"{@code classpath*:}"前缀 和/或 内部Ant风格的正则表达式（使用Spring的{@link org.springframework.util.AntPathMatcher}实用程序进行匹配）。后两者都是有效的通配符。
 *
 * 没有通配符：
 * 在简单的情况下，如果指定的位置路径不以{@code "classpath*:}"前缀开头，并且不包含PathMatcher模式，则此解析器将简单地通过{@code getResource()}调用底层的{@code ResourceLoader}。
 * 示例：真实的URL，例如"{@code file:C:/context.xml}"，伪URL（例如"{@code classpath:/context.xml}"）和简单的无前缀路径，如"{@code /WEB-INF/context.xml}"。
 * 后者将以针对{@code ResourceLoader}的特定方式解析（例如{@code WebApplicationContext}的{@code ServletContextResource}）。
 *
 * Ant风格模式：
 * 当路径位置包含Ant样式模式时，例如：
 * /WEB-INF/*-context.xml
 * com/mycompany/** /applicationContext.xml
 * file:C:/some/path/*-context.xml
 * classpath:com/mycompany/** /applicationContext.xml
 * 解析器遵循更复杂但定义的过程来尝试解决通配符。 它为最后一个非通配符段的路径生成一个{@code Resource} ，并从中获取一个{@code URL}。 如果此URL不“{@code jar：}”URL或容器中的特定变体（例如：WebLogic中的{@code zip：}，WebSphere中的{@code wsjar}等），则从它获取{@code java.io.File}，并用于通过走文件系统来解析通配符。
 * 在一个jar URL的情况下，解析器从它获取一个{@code java.net.JarURLConnection} 或手动解析jar URL，然后遍历jar文件的内容，以解决通配符。
 *
 * 对可移植性的影响：
 * 如果指定的路径已经是文件URL（明确地或隐含地），因为基本的{@code ResourceLoader}是一个文件系统的路径，那么通配符将保证以完全可移植的方式工作。
 * 如果指定的路径是类路径位置，则解析器必须通过{@code Classloader.getResource()}调用获取最后一个非通配符路径段URL。由于这只是路径的一个节点（而不是最后的文件），在这种情况下，它实际上是未定义的（在ClassLoader Javadocs中）返回的是什么样的URL。实际上，它通常是一个{@code java.io.File}，表示类路径资源解析为文件系统位置的目录，或某个类别的jar URL其中类路径资源解析为一个jar位置。尽管如此，这种操作仍然存在可移植性问题。
 * 如果为最后一个非通配符段获取了一个jar URL，解析器必须能够从中获取一个{@code java.net.JarURLConnection}，或者手动解析jar URL，以便能够便利该jar的内容，并解决通配符。这将在大多数环境中工作，但在其他环境中将会失败，并且强烈建议您在依赖它之前，彻底地在您的特定环境中彻底测试来自jar的资源的通配符解析。
 *
 *
 * {@code classpath*:}前缀：
 * 通过"{@code classpath*:}"前缀，可以检索具有相同名称的多个类路径资源。
 * 例如，“{@code classpath *：META-INF / beans.xml}”将在类路径中找到所有“beans.xml”文件，无论是在“classes”目录还是在JAR文件中。这对于在每个jar文件中的同一位置自动检测同名的配置文件特别有用。在内部，这是通过{@code ClassLoader.getResources()}调用发生的，并且是完全可移植的。
 * "classpath*:"前缀也可以与其他位置路径中的PathMatcher模式相结合，例如"classpath*:META-INF/*-beans.xml"。在这种情况下，分辨率策略相当简单：在最后一个非通配符路径段上使用{@code ClassLoader.getResources()}调用，以获取类加载器层次结构中的所有匹配资源，然后便利每个资源上面描述的相同的PathMatcher分辨率策略用于通配符子路径。
 *
 * 其他注意：
 * 警告：请注意，与匹配模式启动相结合时，"{@code classpath*:}" 只能与模式启动前的至少一个根目录一起工作，除非实际的目标 文件驻留在文件系统中。 这意味着像 "{@code classpath*:*.xml}" 这样的模式将不会从jar文件的根目录中检索文件，而只能从扩展目录的根目录中获取文件。 这源于JDK的{@code ClassLoader.getResources()}方法中的限制，该方法传入的空String仅返回文件系统位置（指示潜在的搜索根）。此{@code ResourcePatternResolver}实现是通过{@link URLClassLoader}内省和“java.class.path”清单评估来减轻jar根查找限制; 然而，没有可移植性的保证。
 *
 * 警告： 如果要搜索的根包存在多个类路径位置，则不能保证具有 "classpath:" 资源的Ant样式模式可以找到匹配的资源。
 * 这是因为一个资源，如com/mycompany/package1/service-context.xml 可能只在一个位置，但是当一个路径如 classpath:com/mycompany/** /service-context.xml
 * 用于尝试解决它，解析器将解决 {@code getResource("com/mycompany");} 返回的（第一个）URL。 如果此基本包节点存在于多个类加载器位置中，则实际的最终资源可能不在下面。
 * 因此，在这种情况下，最好使用具有相同Ant样式模式的“{@code classpath *：}”，这将搜索包含根包的所有类路径位置。
 */
public class PathMatchingResourcePatternResolver implements ResourcePatternResolver {
	private static final Log logger = LogFactory.getLog(PathMatchingResourcePatternResolver.class);
	private final ResourceLoader resourceLoader;
	private PathMatcher pathMatcher = new AntPathMatcher();
}
```