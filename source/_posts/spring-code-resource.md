---
title: spring 源码解析 2.统一io Resource ResourceLoader 
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

##### 2.1.1 InputStreamSource
```java
package org.springframework.core.io;

import java.io.IOException;
import java.io.InputStream;

/**
获取 InputStream 对象的简单接口。
是 Spring Resource 的父接口。
对于一次性流，可以使用 org.springframework.core.io.InputStreamResource 提供任何给定的 InputStream 。 
org.springframework.core.io.ByteArrayResource 或任何基于 Resource 的实现可以当做实体化的实例，允许人们多次读取底层内容流。
这使得该接口可用作，例如：邮件附件的抽象内容源。
 */
public interface InputStreamSource {

	/**
    返回一个 InputStream 。预期每个调用都会创建一个新的流。
    当您考虑如JavaMail等的API，在创建邮件附件时，需要多次读取流时，这一需求尤为重要，。 
    对于这种用例，需要每个getInputStream（）调用返回一个新的流。
	 */
	InputStream getInputStream() throws IOException;
}
```
##### 2.1.2 Resource
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
	 * 预计每个调用都会创建一个<i>新的</ i>频道。
	 * 默认实现返回{@link Channels＃newChannel（InputStream）}，结果为{@link #getInputStream（）}）。
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

##### 2.1.3 WritableResource

```java
package org.springframework.core.io;

import java.io.IOException;
import java.io.OutputStream;

/**
 * 用于支持写入的资源的扩展接口。
 * 提供{@link #getOutputStream（）OutputStream 存取器}。
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

}
```

##### 2.1.4 ContextResource

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

##### 2.2.1 AbstractResource
```java
package org.springframework.core.io;

import java.io.File;
import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.InputStream;
import java.net.URI;
import java.net.URISyntaxException;
import java.net.URL;
import java.nio.channels.Channels;
import java.nio.channels.ReadableByteChannel;

import org.springframework.core.NestedIOException;
import org.springframework.util.Assert;
import org.springframework.util.ResourceUtils;

/**
 *  实现{@link Resource}的便利基类，预先实现典型行为。
 *  “存在”方法将检查是否可以打开File或InputStream;
 *  “isOpen”将永远返回false;
 *  “getURL”和“getFile”抛出异常;
 *  “toString”将返回描述。
 */
public abstract class AbstractResource implements Resource {

	/**
	 * This implementation checks whether a File can be opened,
	 * falling back to whether an InputStream can be opened.
	 * This will cover both directories and content resources.
	 */
	@Override
	public boolean exists() {
		// Try file existence: can we find the file in the file system?
		try {
			return getFile().exists();
		}
		catch (IOException ex) {
			// Fall back to stream existence: can we open the stream?
			try {
				InputStream is = getInputStream();
				is.close();
				return true;
			}
			catch (Throwable isEx) {
				return false;
			}
		}
	}

	/**
	 * This implementation always returns {@code true}.
	 */
	@Override
	public boolean isReadable() {
		return true;
	}

	/**
	 * This implementation always returns {@code false}.
	 */
	@Override
	public boolean isOpen() {
		return false;
	}

	/**
	 * This implementation always returns {@code false}.
	 */
	@Override
	public boolean isFile() {
		return false;
	}

	/**
	 * This implementation throws a FileNotFoundException, assuming
	 * that the resource cannot be resolved to a URL.
	 */
	@Override
	public URL getURL() throws IOException {
		throw new FileNotFoundException(getDescription() + " cannot be resolved to URL");
	}

	/**
	 * This implementation builds a URI based on the URL returned
	 * by {@link #getURL()}.
	 */
	@Override
	public URI getURI() throws IOException {
		URL url = getURL();
		try {
			return ResourceUtils.toURI(url);
		}
		catch (URISyntaxException ex) {
			throw new NestedIOException("Invalid URI [" + url + "]", ex);
		}
	}

	/**
	 * This implementation throws a FileNotFoundException, assuming
	 * that the resource cannot be resolved to an absolute file path.
	 */
	@Override
	public File getFile() throws IOException {
		throw new FileNotFoundException(getDescription() + " cannot be resolved to absolute file path");
	}

	/**
	 * This implementation returns {@link Channels#newChannel(InputStream)} with the result of
	 * {@link #getInputStream()}.
	 */
	@Override
	public ReadableByteChannel readableChannel() throws IOException {
		return Channels.newChannel(getInputStream());
	}

	/**
	 * This implementation reads the entire InputStream to calculate the
	 * content length. Subclasses will almost always be able to provide
	 * a more optimal version of this, e.g. checking a File length.
	 * @see #getInputStream()
	 * @throws IllegalStateException if {@link #getInputStream()} returns null.
	 */
	@Override
	public long contentLength() throws IOException {
		InputStream is = getInputStream();
		Assert.state(is != null, "Resource InputStream must not be null");
		try {
			long size = 0;
			byte[] buf = new byte[255];
			int read;
			while ((read = is.read(buf)) != -1) {
				size += read;
			}
			return size;
		}
		finally {
			try {
				is.close();
			}
			catch (IOException ex) {
			}
		}
	}

	/**
	 * This implementation checks the timestamp of the underlying File,
	 * if available.
	 * @see #getFileForLastModifiedCheck()
	 */
	@Override
	public long lastModified() throws IOException {
		long lastModified = getFileForLastModifiedCheck().lastModified();
		if (lastModified == 0L) {
			throw new FileNotFoundException(getDescription() +
					" cannot be resolved in the file system for resolving its last-modified timestamp");
		}
		return lastModified;
	}

	/**
	 * Determine the File to use for timestamp checking.
	 * <p>The default implementation delegates to {@link #getFile()}.
	 * @return the File to use for timestamp checking (never {@code null})
	 * @throws IOException if the resource cannot be resolved as absolute
	 * file path, i.e. if the resource is not available in a file system
	 */
	protected File getFileForLastModifiedCheck() throws IOException {
		return getFile();
	}

	/**
	 * This implementation throws a FileNotFoundException, assuming
	 * that relative resources cannot be created for this resource.
	 */
	@Override
	public Resource createRelative(String relativePath) throws IOException {
		throw new FileNotFoundException("Cannot create a relative resource for " + getDescription());
	}

	/**
	 * This implementation always returns {@code null},
	 * assuming that this resource type does not have a filename.
	 */
	@Override
	public String getFilename() {
		return null;
	}


	/**
	 * This implementation returns the description of this resource.
	 * @see #getDescription()
	 */
	@Override
	public String toString() {
		return getDescription();
	}

	/**
	 * This implementation compares description strings.
	 * @see #getDescription()
	 */
	@Override
	public boolean equals(Object obj) {
		return (obj == this ||
			(obj instanceof Resource && ((Resource) obj).getDescription().equals(getDescription())));
	}

	/**
	 * This implementation returns the description's hash code.
	 * @see #getDescription()
	 */
	@Override
	public int hashCode() {
		return getDescription().hashCode();
	}

}

```

##### 2.2.2 FileSystemResource


