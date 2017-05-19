---
title: spring 源码解析 4.环境与属性解析
date: 2017-05-18 10:26:12
tags:
- spring container
categories:
- spring
- code
---
## 1.概述
Environment 是集成在容器中的抽象，用于模拟应用程序环境的两个关键方面：配置文件和属性。

只有给定的配置文件处于活动状态，配置文件才是要向容器注册的一个命名逻辑组的bean定义。 Bean 可以被分配到不管是以XML定义还是通过注释的配置文件。 环境对象与配置文件的关系在于确定哪些配置文件（如果有）当前处于活动状态，哪些配置文件（如果有的话）默认情况下应该处于活动状态。

属性在几乎所有应用程序中起着重要作用，可能来自各种来源：属性文件，JVM系统属性，系统环境变量，JNDI，servlet上下文参数，ad-hoc属性对象，Maps等。 环境对象与属性关系的作用是为用户提供方便的服务接口，用于配置属性源并从中解析属性。

## 2. PropertyResolver
![右侧部分](/assets/img/spring/springEnvPropertyResolver.png)

### 2.1 属性解析接口

##### 2.1.1 [PropertyResolver](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/env/PropertyResolver.java)
```java
/**
 * 用于解析任何基础源的属性的接口。
 */
public interface PropertyResolver {

	/**
	 * 返回给定的属性键是否可用于解析，即如果给定键的值不是{@code null}。
	 */
	boolean containsProperty(String key);

	/**
	 * 返回与给定 key 相关联的属性值，如果 key 无法解析，则返回{@code null}。
	 */
	String getProperty(String key);

	/**
	 * 返回与给定键相关联的属性值，如果键无法解析，则返回{@code defaultValue}。
	 */
	String getProperty(String key, String defaultValue);

	/**
	 * 返回与给定键相关联的属性值，如果键无法解析，则返回{@code null}。
	 */
	<T> T getProperty(String key, Class<T> targetType);

	/**
	 * 返回与给定键相关联的属性值，如果键无法解析，则返回{@code defaultValue}。
	 */
	<T> T getProperty(String key, Class<T> targetType, T defaultValue);

	/**
	 * 返回与给定键相关联的属性值（从不{@code null}）。
	 */
	String getRequiredProperty(String key) throws IllegalStateException;

	/**
	 * 返回与给定键相关联的属性值，转换为给定的targetType（从不{@code null}）。
	 */
	<T> T getRequiredProperty(String key, Class<T> targetType) throws IllegalStateException;

	/**
	 * 解析给定文本中的 ${...} 占位符，用{@link #getProperty}解析的相应属性值替换它们。
	 * 没有默认值的未解决的占位符将被忽略并通过不变。
	 */
	String resolvePlaceholders(String text);

	/**
	 * 解析给定文本中的 ${...} 占位符，用{@link #getProperty}解析的相应属性值替换它们。没有默认值的不可解决的占位符将导致抛出IllegalArgumentException。
	 */
	String resolveRequiredPlaceholders(String text) throws IllegalArgumentException;

}

```

##### 2.1.2[ConfigurablePropertyResolver](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/env/ConfigurablePropertyResolver.java)
```java
/**
 * 配置接口大多数（如果不是全部）由 {@link PropertyResolver} 类型实现。
 * 提供访问和自定义将属性值从一种类型转换为另一种时使用的{@link org.springframework.core.convert.ConversionService ConversionService}的功能。
 */
public interface ConfigurablePropertyResolver extends PropertyResolver {

	/**
	 * 返回在属性上执行类型转换时使用的{@link ConfigurableConversionService}。
	 * 返回的转换服务的可配置性质允许方便地添加和删除单个{@code Converter}实例：
	 *
	 * ConfigurableConversionService cs = env.getConversionService();
	 * cs.addConverter(new FooConverter());
	 */
	ConfigurableConversionService getConversionService();

	/**
	 * 设置在属性上执行类型转换时使用的{@link ConfigurableConversionService}。
	 * 注意：与其完全替换{@code ConversionService}的替代方法，不如考虑通过获取 {@link #getConversionService()} ，并调用方法，如{@code #addConverter}，添加或删除单个 {@code Converter} 实例。
	 */
	void setConversionService(ConfigurableConversionService conversionService);

	/**
	 * 设置由此解析器替换的占位符的前缀必须以它开头。
	 */
	void setPlaceholderPrefix(String placeholderPrefix);

	/**
	 * 设置此解析器替换的占位符的后缀必须以其结尾。
	 */
	void setPlaceholderSuffix(String placeholderSuffix);

	/**
	 * 指定由此解析器替换的占位符与其关联的默认值之间的分隔符，或者，如果没有此类特殊字符作为值分隔符处理，则为{@code null}。
	 */
	void setValueSeparator(String valueSeparator);

	/**
	 * 设置是否在遇到嵌套在给定属性的值中的未解析占位符时抛出异常。 {@code false}值表示严格的不，即将抛出异常。 {@code true}值表示无法解析的嵌套占位符应以未解析的 ${...} 形式传递。
	 * {@link #getProperty(String)} 及其变体的实现必须检查此处设置的值，以在属性值包含不可解析占位符时确定正确的行为。
	 */
	void setIgnoreUnresolvableNestedPlaceholders(boolean ignoreUnresolvableNestedPlaceholders);

	/**
	 * 指定哪些属性必须存在，由 {@link #validateRequiredProperties()} 验证。
	 */
	void setRequiredProperties(String... requiredProperties);

	/**
	 * 验证 {@link #setRequiredProperties} 指定的每个属性是否存在，并解析为 非{@code null} 值。
	 */
	void validateRequiredProperties() throws MissingRequiredPropertiesException;

}

```

### 2.2 属性解析实现与抽象类

##### 2.2.1 [PropertyPlaceholderHelper](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/util/PropertyPlaceholderHelper.java)
```java

/**
 * 使用具有占位符值的Strings的实用程序类。 占位符采用 {@code ${name}} 的形式。 使用 {@code PropertyPlaceholderHelper} 这些占位符可以替代用户提供的值。
 * 可以使用 {@link Properties} 实例或使用 {@link PlaceholderResolver} 提供替换值。
 */
public class PropertyPlaceholderHelper {

	private static final Log logger = LogFactory.getLog(PropertyPlaceholderHelper.class);

	private static final Map<String, String> wellKnownSimplePrefixes = new HashMap<>(4);

	static {
		wellKnownSimplePrefixes.put("}", "{");
		wellKnownSimplePrefixes.put("]", "[");
		wellKnownSimplePrefixes.put(")", "(");
	}

	private final String placeholderPrefix;

	private final String placeholderSuffix;

	private final String simplePrefix;

	private final String valueSeparator;

	private final boolean ignoreUnresolvablePlaceholders;

	public String replacePlaceholders(String value, PlaceholderResolver placeholderResolver) {
		Assert.notNull(value, "'value' must not be null");
		return parseStringValue(value, placeholderResolver, new HashSet<>());
	}

	protected String parseStringValue(
			String value, PlaceholderResolver placeholderResolver, Set<String> visitedPlaceholders) {

		StringBuilder result = new StringBuilder(value);

		int startIndex = value.indexOf(this.placeholderPrefix);
		while (startIndex != -1) {
			int endIndex = findPlaceholderEndIndex(result, startIndex);
			if (endIndex != -1) {
				String placeholder = result.substring(startIndex + this.placeholderPrefix.length(), endIndex);
				String originalPlaceholder = placeholder;
				if (!visitedPlaceholders.add(originalPlaceholder)) {
					throw new IllegalArgumentException(
							"Circular placeholder reference '" + originalPlaceholder + "' in property definitions");
				}
				// Recursive invocation, parsing placeholders contained in the placeholder key.
				// 递归调用，解析包含在占位符键中的占位符。
				placeholder = parseStringValue(placeholder, placeholderResolver, visitedPlaceholders);
				// Now obtain the value for the fully resolved key...
				// 现在获取完全解析key的值...
				String propVal = placeholderResolver.resolvePlaceholder(placeholder);
				if (propVal == null && this.valueSeparator != null) {
					int separatorIndex = placeholder.indexOf(this.valueSeparator);
					if (separatorIndex != -1) {
						String actualPlaceholder = placeholder.substring(0, separatorIndex);
						String defaultValue = placeholder.substring(separatorIndex + this.valueSeparator.length());
						propVal = placeholderResolver.resolvePlaceholder(actualPlaceholder);
						if (propVal == null) {
							propVal = defaultValue;
						}
					}
				}
				if (propVal != null) {
					// Recursive invocation, parsing placeholders contained in the
					// previously resolved placeholder value.
					// 递归调用，解析先前解析的占位符值中包含的占位符。
					propVal = parseStringValue(propVal, placeholderResolver, visitedPlaceholders);
					result.replace(startIndex, endIndex + this.placeholderSuffix.length(), propVal);
					if (logger.isTraceEnabled()) {
						logger.trace("Resolved placeholder '" + placeholder + "'");
					}
					startIndex = result.indexOf(this.placeholderPrefix, startIndex + propVal.length());
				}
				else if (this.ignoreUnresolvablePlaceholders) {
					// Proceed with unprocessed value.
					// 继续处理未处理的值。
					startIndex = result.indexOf(this.placeholderPrefix, endIndex + this.placeholderSuffix.length());
				}
				else {
					throw new IllegalArgumentException("Could not resolve placeholder '" +
							placeholder + "'" + " in value \"" + value + "\"");
				}
				visitedPlaceholders.remove(originalPlaceholder);
			}
			else {
				startIndex = -1;
			}
		}

		return result.toString();
	}

	/**
	 * 用于解决字符串中包含的占位符替换值的策略界面。
	 */
	@FunctionalInterface
	public interface PlaceholderResolver {

		/**
		 * 将提供的占位符名称解析为替换值。
		 */
		String resolvePlaceholder(String placeholderName);
	}

}

```
##### 2.2.2 [AbstractPropertyResolver](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/env/AbstractPropertyResolver.java)
```java
/**
 * 用于解析任何基础源的属性的抽象基类。
 */
public abstract class AbstractPropertyResolver implements ConfigurablePropertyResolver {

	protected final Log logger = LogFactory.getLog(getClass());

	private volatile ConfigurableConversionService conversionService;

	private PropertyPlaceholderHelper nonStrictHelper;

	private PropertyPlaceholderHelper strictHelper;

	private boolean ignoreUnresolvableNestedPlaceholders = false;

	private String placeholderPrefix = SystemPropertyUtils.PLACEHOLDER_PREFIX;

	private String placeholderSuffix = SystemPropertyUtils.PLACEHOLDER_SUFFIX;

	private String valueSeparator = SystemPropertyUtils.VALUE_SEPARATOR;

	private final Set<String> requiredProperties = new LinkedHashSet<>();

	@Override
	public String resolvePlaceholders(String text) {
		if (this.nonStrictHelper == null) {
			this.nonStrictHelper = createPlaceholderHelper(true);
		}
		return doResolvePlaceholders(text, this.nonStrictHelper);
	}

	private PropertyPlaceholderHelper createPlaceholderHelper(boolean ignoreUnresolvablePlaceholders) {
		return new PropertyPlaceholderHelper(this.placeholderPrefix, this.placeholderSuffix,
				this.valueSeparator, ignoreUnresolvablePlaceholders);
	}

	private String doResolvePlaceholders(String text, PropertyPlaceholderHelper helper) {
		return helper.replacePlaceholders(text, new PropertyPlaceholderHelper.PlaceholderResolver() {
			@Override
			public String resolvePlaceholder(String placeholderName) {
				return getPropertyAsRawString(placeholderName);
			}
		});
	}

	/**
	 * Retrieve the specified property as a raw String,
	 * i.e. without resolution of nested placeholders.
	 * @param key the property name to resolve
	 * @return the property value or {@code null} if none found
	 */
	protected abstract String getPropertyAsRawString(String key);

}

```

##### 2.2.3 [PropertySourcesPropertyResolver](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/env/PropertySourcesPropertyResolver.java)
```java
/**
 * {@link PropertyResolver} 实现，它根据{@link PropertySources}的底层解析属性值。
 */
public class PropertySourcesPropertyResolver extends AbstractPropertyResolver {

	private final PropertySougggtirces propertySources;

}
```

## 3. PropertySource


## 4. Environment
![](/assets/img/spring/springEnvPropertyResolver.png)

###