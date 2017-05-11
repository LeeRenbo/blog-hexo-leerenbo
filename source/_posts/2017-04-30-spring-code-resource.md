---
title: spring 源码解析 2.统一io Resource
date: 2017-04-30 14:05:40
tags:
- spring container
categories:
- spring
- code
---
Resource参考文档：<http://docs.spring.io/spring/docs/5.0.0.M5/spring-framework-reference/htmlsingle/#resources>

## 1.概述
Java的标准java.net.URL类和各种URL前缀的标准处理程序不足以满足对低级资源的所有访问。 例如，没有可用于访问需要从类路径或相对于ServletContext获取的资源的标准化URL实现。 虽然可以为专门的URL前缀注册新的处理程序（类似于前缀如http :)的现有处理程序，但这通常是相当复杂的，并且URL接口仍然缺少一些所需的功能，例如检查存在的方法 的资源。

基于以上原因，spring 提供了 ：
- Resource - 统一资源，规范化操作
- ResourceLoader - 统一资源加载操作，屏蔽加载具体实现

## 2.Resource
![](/assets/img/spring/springResource.png)

### 2.1 接口
- InputStreamSource - 用于获取InputStream，每次都是新的InputStream
- Resource - 用于获取资源描述，判断资源状态
- WritableResource - 可写入资源，用于获取 OutputStream
- ContextResource - 用于获取封闭上下文的相对路径

##### 2.1.1 [InputStreamSource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/InputStreamSource.java)
```java
package org.springframework.core.io;

import java.io.IOException;
import java.io.InputStream;

/**
 * 获取 {@link InputStream} 对象的简单接口。
 * 是 {@link Resource} 的父接口。
 * 对于一次性流，可以使用 {@link InputStreamResource} 提供任何给定的 {@code InputStream} 。
 * {@link ByteArrayResource} 或任何基于 {@code Resource} 的实现可以当做实体化的实例使用，允许人们多次读取底层内容流。
 * 这使得该接口可用作，例如：邮件附件的抽象内容源。
 */
public interface InputStreamSource {

	/**
	 * 返回一个 InputStream 。预期每个调用都会创建一个新的流。
	 * 当您考虑如JavaMail等的API，在创建邮件附件时，需要多次读取流时，这一需求尤为重要。
	 * 对于这种用例，需要每个getInputStream（）调用返回一个新的流。
	 */
	InputStream getInputStream() throws IOException;
}
```
##### 2.1.2 [Resource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/Resource.java)
```java
package org.springframework.core.io;

import java.io.File;
import java.io.IOException;
import java.io.InputStream;
import java.net.URI;
import java.net.URL;
import java.nio.channels.Channels;
import java.nio.channels.ReadableByteChannel;

/**
 * 资源描述符的接口，从基础资源的实际类型（例如文件或类路径资源）抽象。 
 * 如果每个资源以物理形式存在，则可以打开InputStream。
 * 但只能为某些资源返回URL或File句柄。 实际行为是根据具体实现。
 */
public interface Resource extends InputStreamSource {

	/**
	* 判断此资源是否以物理形式存在。
	* 该方法执行确定的存在检查。
	* 而Resource句柄的存在仅保证有效的描述符句柄。
	*/
	boolean exists();

	/**
	 * 指示是否可以通过 getInputStream() 读取此资源的内容。 
	 * 典型资源描述符将为  true ; 
	 * 请注意，尝试时实际内容读取可能仍然失败。
	 * 但是， false 的值是无法读取资源内容的明确指示。
	 */
	default boolean isReadable() {
		return true;
	}

	/**
	 * 指示此资源是否表示具有开放流的句柄。 
	 * 如果 true ，InputStream不能被多次读取，并且必须读取和关闭以避免资源泄漏。
	 * 对于典型的资源描述符，将是 false 。
	 */
	default boolean isOpen() {
		return false;
	}

	/**
	 * 确定此资源是否表示文件系统中的文件。
	 * 值{@code true}强烈建议（但不能保证）{@link #getFile（）}调用将成功。
	 * 默认情况下，是保守的{@code false}。
	 * @since 5.0
	 */
	default boolean isFile() {
		return false;
	}

	/**
	 * 返回此资源的URL句柄。
	 */
	URL getURL() throws IOException;

	/**
	 * 返回此资源的URI句柄.
	 */
	URI getURI() throws IOException;

	/**
	 * 返回此资源的文件句柄.
	 */
	File getFile() throws IOException;

	/**
	 * 返回{@link ReadableByteChannel}。
	 * 预计每个调用都会创建一个新的channel。
	 * 默认实现返回{@link Channels＃newChannel(InputStream)}，使用{@link #getInputStream()}的结果}）。
	 * @since 5.0
	 */
	default ReadableByteChannel readableChannel() throws IOException {
		return Channels.newChannel(getInputStream());
	}

	/**
	 * 确定此资源的内容长度
	 */
	long contentLength() throws IOException;

	/**
	 * 确定此资源的最后修改的时间戳.
	 */
	long lastModified() throws IOException;

	/**
	 * 创建相对于此资源的资源
	 */
	Resource createRelative(String relativePath) throws IOException;

	/**
	 * 确定此资源的文件名，即通常是路径的最后一部分：例如“myfile.txt”。
	 * 如果这种类型的资源没有文件名，返回{@code null}。
	 */
	String getFilename();

	/**
	 * 返回此资源的描述，用于处理资源时的错误输出。
	 * 还鼓励实现类的{@code toString}方法返回此值。
	 */
	String getDescription();

}
```

##### 2.1.3 [WritableResource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/WritableResource.java)

```java
package org.springframework.core.io;

import java.io.IOException;
import java.io.OutputStream;

/**
 * 用于支持写入的扩展资源的接口。
 * 提供{@link #getOutputStream（）OutputStream 访问方法}。
 */
public interface WritableResource extends Resource {

	/**
	 * 指示此资源的内容是否可以通过{@link #getOutputStream（）}写入。
	 * 对于典型的资源描述符，将是{@code true}; 
	 * 请注意，尝试时实际的内容写入可能仍然失败。
	 * 但是，值{@code false}是资源内容无法修改的明确指示。
	 */
	default boolean isWritable() {
		return true;
	}

	/**
	 * 返回底层资源的{@link OutputStream}，允许（覆盖）写入内容。
	 */
	OutputStream getOutputStream() throws IOException;

	/**
	 * 返回一个{@link WritableByteChannel}。
	 * 预计每次调用都会创建一个新鲜的 channel。
	 * 默认实现返回{@link Channels#newChannel(OutputStream)}，使用{@link #getOutputStream()}的结果）。
	 * @since 5.0
	 */
	default WritableByteChannel writableChannel() throws IOException {
		return Channels.newChannel(getOutputStream());
	}

}
```

##### 2.1.4 [ContextResource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/ContextResource.java)

```java
package org.springframework.core.io;

/**
 * 从封闭的“上下文”加载的资源的扩展接口，
 * 例如 来自{@link javax.servlet.ServletContext}，但也可以使用纯类路径路径或相对文件系统路径（没有明确的前缀指定，因此适用于相对于本地{@link ResourceLoader}的上下文）。
 */
public interface ContextResource extends Resource {

	/**
	 * 返回包围的“上下文”中的路径。
	 * 这通常是相对于上下文特定根目录的路径，例如。 一个ServletContext根或一个PortletContext根。
	 */
	String getPathWithinContext();
	
}
```
### 2.2 抽象实现类

##### 2.2.1 [AbstractResource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/AbstractResource.java)
```java
/**
 *  实现{@link Resource}的便利基类，预先实现典型行为。
 *  “存在”方法将检查是否可以打开File或InputStream;
 *  “isOpen”将永远返回false;
 *  “getURL”和“getFile”抛出异常;
 *  “toString”将返回描述。
 */
public abstract class AbstractResource implements Resource {
    
}

```

##### 2.2.2 [DescriptiveResource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/DescriptiveResource.java)
```java
/**
 * 简单的 {@link Resource} 实现，保存资源描述，但不指向实际可读的资源。
 * 如果API需要 {@code Resource} 参数，但不一定用于实际读取时被用作占位符。
 */
public class DescriptiveResource extends AbstractResource {
	private final String description;
}
```

##### 2.2.3 [VfsResource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/VfsResource.java)
```java
/**
 * 基于JBoss VFS的{@link Resource}实现。
 * 从Spring 4.0开始，该类支持JBoss AS 6+上的VFS 3.x（软件包{@code org.jboss.vfs}），特别兼容JBoss AS 7和WildFly 8。
 */
public class VfsResource extends AbstractResource {
    	private final Object resource;
}
```


##### 2.2.4 [InputStreamResource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/InputStreamResource.java)
```java
/**
 * {@link Resource}实现给定的{@link InputStream}。
 * 只有在没有其他特定的 {@code Resource} 实现适用的情况下才使用。
 * 特别是，尽可能选择{@link ByteArrayResource}或任何基于文件的{@code Resource}实现。
 *
 * 与其他{@code Resource}实现相反，这是一个已经打开的资源的描述符，因此从{@link #isOpen()}返回 {@code true}。
 * 如果您需要将资源描述符保留在某处，或者您需要多次读取数据流，请勿使用{@code InputStreamResource}。
 *
 */
public class InputStreamResource extends AbstractResource {
	private final InputStream inputStream;
	private final String description;
	private boolean read = false;
}
```

##### 2.2.5 [ByteArrayResource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/ByteArrayResource.java)
```java
/**
 * 一个给定的字节数组的 {@link Resource}实现。
 * 为给定的字节数组创建{@link ByteArrayInputStream}。
 * 用于从任何给定的字节数组加载内容，而无需使用单次使用的{@link InputStreamResource}。
 * 特别适用于从本地内容创建邮件附件，JavaMail需要能够多次读取流。
 */
public class ByteArrayResource extends AbstractResource {
	private final byte[] byteArray;
	private final String description;
}
```

##### 2.2.6 [AbstractFileResolvingResource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/AbstractFileResolvingResource.java)
```java
/**
 * 用于将URL解析为文件引用的资源的抽象基类，例如{@link UrlResource}或{@link ClassPathResource}。
 * 在URL中检测“文件”协议以及 JBoss“vfs”协议，相应地解析文件系统引用。
 */
public abstract class AbstractFileResolvingResource extends AbstractResource {
}

```

##### 2.2.7 [UrlResource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/UrlResource.java)
```java
/**
 * {@code java.net.URL}定位器的{@link Resource}实现。
 * 在{@code“file：”}协议的情况下支持解析为{@code URL}，还可以作为{@code File}。
 */
public class UrlResource extends AbstractFileResolvingResource {

	/**
	 * 如果有值，原始URI; 用于URI和文件访问。
	 */
	private final URI uri;

	/**
	 * 原始URL，用于实际访问。
	 */
	private final URL url;

	/**
	 * 已清理的URL（具有标准化路径），用于比较。
	 */
	private final URL cleanedUrl;
}
```
所有URL都具有标准化的字符串表示形式，以便使用适当的标准化前缀来区分URL类型。
例如：文件系统路径 `file:`，HTTP协议`http:`，FTP`ftp:`
除了几个已知前缀如`classpatch`会创建适当的Resource，不认识的前缀会作为标准 URL 串创。建UrlResource。

##### 2.2.8 [ClassPathResource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/ClassPathResource.java)
```java
/**
 * 路径资源的{@link Resource}实现。 使用给定的{@link ClassLoader}或给定的{@link Class}来加载资源。
 * 如果类路径资源驻留在文件系统中，则支持{@code java.io.File}的解析。
 * 但不支持JAR中的资源
 * 始终支持解析为URL。
 */
public class ClassPathResource extends AbstractFileResolvingResource {
	private final String path;
	private ClassLoader classLoader;
	private Class<?> clazz;
}
```
`classpath:`创建ClassPathResource。
从classpath下加载的资源。不管是从线程classloader，给定classloader还是class。都可以使用此Resource。

##### 2.2.9 [PathResource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/PathResource.java)
```java
/**
 * {@code java.nio.file.Path}句柄的{@link Resource}实现。
 * 支持分辨率为File，也可以作为URL。
 * 实现扩展的{@link WritableResource}接口。
 */
public class PathResource extends AbstractResource implements WritableResource {
	private final Path path;
}
```

###### 2.2.10 [FileSystemResource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/FileSystemResource.java)
```java
/**
 * {@code java.io.File}句柄的{@link Resource}实现。
 * 支持作为{@code File}和{@code URL}的解析。
 * 实现扩展的{@link WritableResource}界面。
 */
public class FileSystemResource extends AbstractResource implements WritableResource {
	private final File file;
	private final String path;
}

```

###### 2.2.10 [FileSystemContextResource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/FileSystemResourceLoader.java)
```java
	/**
	 * FileSystemResource，通过实现ContextResource接口显式表达上下文相对路径。
	 */
	private static class FileSystemContextResource extends FileSystemResource implements ContextResource {
	}
```

###### 2.2.11 [ClassRelativeContextResource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/ClassRelativeResourceLoader.java)
```java
	/**
	 * ClassPathResource通过实现ContextResource接口显式表达上下文相对路径。
	 */
	private static class ClassRelativeContextResource extends ClassPathResource implements ContextResource {
		private final Class<?> clazz;
}
```

###### 2.2.12 [ClassPathContextResource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/DefaultResourceLoader.java)
```java
	/**
	 * ClassPathResource通过实现ContextResource接口显式表达上下文相对路径。
	 */
	protected static class ClassPathContextResource extends ClassPathResource implements ContextResource {
	}
```