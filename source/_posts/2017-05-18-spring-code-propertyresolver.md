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
### 3.1 PropertySource抽象与实现
##### 3.1.1 [PropertySource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/env/PropertySource.java)
```java
/**
 * 表示 name/value 属性对的来源的抽象基类。底层的 {@linkplain #getSource（）source object} 可以任何 {@code T} 类型的封装属性。示例包括 {@link java.util.Properties} 对象，{@link java.util.Map}对象，{@code ServletContext} 和 {@code ServletConfig} 对象（用于访问init参数）。浏览 {@code PropertySource} 类型层次结构以查看提供的实现。
 *
 * 通常不会孤立地使用 {@code PropertySources} 对象，而是通过 {@link PropertySources} 对象来集成属性源，并结合使用 {@link PropertyResolver} 。实现跨越 {@code PropertySources} 集合的基于优先级搜索。
 *
 * {@code PropertySource} 标识不是基于封装属性的内容而是基于 {@code PropertySource} 的 {@link #getName() name}。这对于在集合上下文中操作 {@code PropertySource} 对象很有用。有关详细信息，请参阅 {@link MutablePropertySources} 中的操作以及 {@link #named(String)} 和 {@link #toString()} 方法。
 *
 * 请注意，使用 @{@link
 * org.springframework.context.annotation.Configuration Configuration} 类时，@{@link org.springframework.context.annotation.PropertySource PropertySource} 注释提供了一种方便和声明性的方式向封闭的 {@code Environment} 添加属性源。
 */
public abstract class PropertySource<T> {
	protected final String name;
	protected final T source;
}
```
##### 3.1.2 [EnumerablePropertySource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/env/EnumerablePropertySource.java)
```java
/**
 * 可询问其底层源对象，来枚举所有可能的属性 name/value对 的{@link PropertySource}抽象类。暴露 {@link #getPropertyNames()} 方法，以允许调用者内省自己的可用属性，而不必访问底层的源对象。这也有助于更有效地实现 {@link #containsProperty(String)}，因为它可以调用 {@link #getPropertyNames()} 并遍历返回的数组，而不是尝试调用  {@link #getProperty(String)}，这可能更昂贵。实现可能会考虑缓存 {@link #getPropertyNames()} 的结果，以充分利用此性能机会。
 *
 * 大多数框架提供的{@code PropertySource}实现是可枚举的;一个反例是{@code JndiPropertySource}，由于JNDI的性质，在任何给定时间都不可能确定所有可能的属性名称;而只能尝试访问一个属性（通过{@link #getProperty（String）}）来评估它是否存在。
 */
public abstract class EnumerablePropertySource<T> extends PropertySource<T> {
	public abstract String[] getPropertyNames();
}
```

##### 3.1.3 [MapPropertySource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/env/MapPropertySource.java)
```java
/**
 * {@link PropertySource}从{@code Map}对象读取 keys 和 values 。
 */
public class MapPropertySource extends EnumerablePropertySource<Map<String, Object>> {
}
```
##### 3.1.4 [SystemEnvironmentPropertySource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/env/SystemEnvironmentPropertySource.java)
```java
/**
 * 专为 {@linkplain AbstractEnvironment#getSystemEnvironment() system environment variables} 使用而设计的特殊{@link MapPropertySource} 。补偿Bash和其他shell中的约束，不允许包含句点字符和/或连字符的变量;同样允许shell更常用的使用的大写属性名称变体。
 *
 * 例如，调用 {@code getProperty("foo.bar")} 将尝试找到原始属性或任何“等效”属性的值，返回首次找到的值：
 * {@code foo.bar} - 原始名称
 * {@code foo_bar} - 句点转下划线（如果有）
 * {@code FOO.BAR} - 原件，大写
 * {@code FOO_BAR} - 带下划线和大写
 * 上述任何连字符变体也可以工作，甚至混合点/连字符变体。
 *
 * 同样适用于 {@link #containsProperty(String)} 的调用，如果存在任何上述属性，则返回{@code true}，否则{@code false}。
 *
 * 将活动或默认配置文件指定为环境变量时，此功能特别有用。 Bash下是不允许的：
 * spring.profiles.active=p1 java -classpath ... MyApp
 *
 * 但是，以下语法是允许的，也是更常规的：
 * SPRING_PROFILES_ACTIVE=p1 java -classpath ... MyApp
 *
 * 为此类（或包）启用调试或跟踪级别日志记录，以解释何时发生这些“属性名称解析”。
 *
 * 此属性源默认包含在 {@link StandardEnvironment} 及其所有子类中。
 */
public class SystemEnvironmentPropertySource extends MapPropertySource {
}
```
##### 3.1.5 [PropertiesPropertySource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/env/PropertiesPropertySource.java)
```java
/**
 * {@link PropertySource} 实现，从 {@link java.util.Properties} 对象中提取属性。
 * 请注意，由于 {@code Properties} 对象在技术上是 {@code <Object，Object>} {@link java.util.Hashtable Hashtable}，可能包含非{@code String}键或值。 然而，这种实现仅限于访问 {@code String} 的键和值，与{@link Properties＃getProperty}和{@link Properties＃setProperty}的方式相同。
 */
public class PropertiesPropertySource extends MapPropertySource {
}
```
##### 3.1.6 [CommandLinePropertySource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/env/CommandLinePropertySource.java)
```java
/**
 * 由命令行参数支持的{@link PropertySource}实现的抽象基类。 参数化类型{@code T}表示命令行选项的基础源。 在{@link SimpleCommandLinePropertySource}的情况下，这可能与String数组一样简单，或者在{@link JOptCommandLinePropertySource}的情况下特定于特定的API，例如JOpt的{@code OptionSet}。
 * 
 * 
 * 目的和一般用法
 * 用于独立的基于Spring的应用程序，即通过传统{@code main}方法从命令行接受参数的 {@code String[]} 引导的应用程序。 在许多情况下，直接在{@code main}方法中处理命令行参数可能就足够了，但在其他情况下，可能需要将参数作为值注入Spring bean。 这是后一种情况，{@code CommandLinePropertySource}变得有用。 通常将 {@code CommandLinePropertySource} 添加到Spring {@code ApplicationContext} 的 {@link Environment} 中，此时所有命令行参数可通过 {@link Environment#getProperty(String)} 方法获得。 例如：
 *
 * public static void main(String[] args) {
 *     CommandLinePropertySource clps = ...;
 *     AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
 *     ctx.getEnvironment().getPropertySources().addFirst(clps);
 *     ctx.register(AppConfig.class);
 *     ctx.refresh();
 * }
 *
 * 通过上面的引导逻辑，{@code AppConfig} 类可以 {@code @Inject} Spring {@code Environment}并直接查询属性：
 *
 * @Configuration
 * public class AppConfig {
 *
 *     @Inject Environment env;
 *
 *     @Bean
 *     public void DataSource dataSource() {
 *         MyVendorDataSource dataSource = new MyVendorDataSource();
 *         dataSource.setHostname(env.getProperty("db.hostname", "localhost"));
 *         dataSource.setUsername(env.getRequiredProperty("db.username"));
 *         dataSource.setPassword(env.getRequiredProperty("db.password"));
 *         // ...
 *         return dataSource;
 *     }
 * }
 *
 *
 * 由于 {@code CommandLinePropertySource} 已使用 {@code #addFirst} 方法添加到 {@code Environment} 的 {@link MutablePropertySources} 集合中，因此具有最高的搜索优先级，这意味着 “db.hostname” 或可能存在于其他属性源（如系统环境变量）的属性中，它将首先从命令行属性源中选择。 这是一个合理的方法，因为在命令行上指定的参数自然比指定为环境变量的参数更具体。
 * 作为注入 {@code Environment} 的替代方法，Spring的 {@code @Value} 注释可以用于注入这些属性，因为已经注册了 {@link PropertySourcesPropertyResolver} bean，直接或通过使用 {@code <context：property-placeholder>} 元素。 例如：
 * @Component
 * public class MyComponent {
 *
 *     @Value("my.property:defaultVal")
 *     private String myProperty;
 *
 *     public void getMyProperty() {
 *         return this.myProperty;
 *     }
 *
 *     // ...
 * }
 *
 *
 * 使用选项参数
 * 单个命令行参数通过通常的 {@link PropertySource#getProperty(String)} 和 {@link PropertySource#containsProperty(String)} 方法表示为属性。 例如，给出以下命令行:
 * --o1=v1 --o2
 * 'o1' 和 'o2' 被视为“选项参数”，并且以下断言将评估为true：
 *
 * CommandLinePropertySource<?> ps = ...
 * assert ps.containsProperty("o1") == true;
 * assert ps.containsProperty("o2") == true;
 * assert ps.containsProperty("o3") == false;
 * assert ps.getProperty("o1").equals("v1");
 * assert ps.getProperty("o2").equals("");
 * assert ps.getProperty("o3") == null;
 *
 * 请注意，'o2'选项没有参数，但{@code getProperty（“o2”）}解析为空字符串 ({@code ""}) 而不是{@code null}，而{@code getProperty “o3”）}解析为{@code null}，因为没有指定。 此行为与所有{@code PropertySource}实现所遵循的一般约定一致。
 *
 * 另请注意，虽然在上述示例中使用 "--" 来表示一个选项参数，但是这种语法可能会在单独的命令行参数库中有所不同。 例如，基于JOpt或Commons CLI的实现可能允许单个破折号 ("-") “短”选项参数等。
 *
 *
 * 使用非选项参数
 *
 * 这种抽象也支持非选项参数。 任何没有选项样式前缀（如 "-" 或 "--" ）提供的参数都被视为“非选项参数”，通过特殊的 {@linkplain
 * #DEFAULT_NON_OPTION_ARGS_PROPERTY_NAME "nonOptionArgs"} 属性可以使用。 如果指定了多个非选项参数，则此属性的值将是包含所有参数的以逗号分隔的字符串。 这种方法确保来自 {@code
 * CommandLinePropertySource} 的所有属性的简单且一致的返回类型（String），同时适用于与Spring {@link Environment} 及其内置 {@code ConversionService}。 请考虑以下示例：
 *
 * --o1=v1 --o2=v2 /path/to/file1 /path/to/file2
 *
 * 在本示例中，“o1”和“o2”将被视为“选项参数”，而两个文件系统路径将被视为“非选项参数”。 因此，以下断言将评估为真：
 *
 * CommandLinePropertySource<?> ps = ...
 * assert ps.containsProperty("o1") == true;
 * assert ps.containsProperty("o2") == true;
 * assert ps.containsProperty("nonOptionArgs") == true;
 * assert ps.getProperty("o1").equals("v1");
 * assert ps.getProperty("o2").equals("v2");
 * assert ps.getProperty("nonOptionArgs").equals("/path/to/file1,/path/to/file2");
 *
 * 如上所述，当与Spring {@code Environment}抽象结合使用时，逗号分隔的字符串可能很容易转换为String数组或列表：
 *
 * Environment env = applicationContext.getEnvironment();
 * String[] nonOptionArgs = env.getProperty("nonOptionArgs", String[].class);
 * assert nonOptionArgs[0].equals("/path/to/file1");
 * assert nonOptionArgs[1].equals("/path/to/file2");
 *
 * 可以通过 {@link #setNonOptionArgsPropertyName(String)} 方法定制特殊“非选项参数”属性的名称。 建议这样做，因为它为非选项参数赋予正确的语义值。 例如，如果将文件系统路径指定为非选项参数，则可能将其称为“file.locations”类似于“nonOptionArgs”的默认值：
 *
 * public static void main(String[] args) {
 *     CommandLinePropertySource clps = ...;
 *     clps.setNonOptionArgsPropertyName("file.locations");
 *
 *     AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
 *     ctx.getEnvironment().getPropertySources().addFirst(clps);
 *     ctx.register(AppConfig.class);
 *     ctx.refresh();
 * }
 *
 *
 * 限制
 * 这种抽象不是为了暴露基础命令行解析API（如JOpt或Commons CLI）的全部功能。 它的意图恰恰相反：提供最简单的可能的抽象，以便在命令行参数解析之后访问。  解析主方法中的参数的{@code String []}，然后简单地将解析结果提供给{@code CommandLinePropertySource}的实现。 在这一点上，所有参数都可以被认为是“选项”或“非选项”参数，如上所述可以通过普通的{@code PropertySource}和{@code Environment} API来访问。
 */
public abstract class CommandLinePropertySource<T> extends EnumerablePropertySource<T> {
	public static final String COMMAND_LINE_PROPERTY_SOURCE_NAME = "commandLineArgs";
	public static final String DEFAULT_NON_OPTION_ARGS_PROPERTY_NAME = "nonOptionArgs";
	private String nonOptionArgsPropertyName = DEFAULT_NON_OPTION_ARGS_PROPERTY_NAME;
}
```

##### 3.1.7 [SimpleCommandLinePropertySource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/env/SimpleCommandLinePropertySource.java)
```java
/**
 * {@link CommandLinePropertySource}实现，由一个简单的String数组支持
 *
 * 
 * 目的
 * 此{@code CommandLinePropertySource}实现旨在提供解析命令行参数的最简单的方法。 与所有 {@code CommandLinePropertySource} 实现一样，命令行参数分为两个不同的组： 选项参数 和 非选项参数 ，如下所述 从Javadoc为{@link SimpleCommandLineArgsParser}复制的部分） ：
 *
 *
 * 使用选项参数
 * 选项参数必须遵守确切的语法：
 * --optName[=optValue]
 * 也就是说，选项必须以"{@code --}"为前缀，并且可以指定或不指定值。 如果指定了一个值，则必须使用等号（“=”）分隔不带空格的名称和值。
 * 选项参数的有效示例
 * --foo
 * --foo=bar
 * --foo="bar then baz"
 * --foo=bar,baz,biz
 * 无效的选项参数示例
 * -foo
 * --foo bar
 * --foo = bar
 * --foo=bar --foo=baz --foo=biz
 *
 *
 * 使用非选项参数
 * 在没有 "{@code --}" 选项前缀的命令行中指定的任何和所有参数将被视为“非选项参数”，并通过 {@link #getNonOptionArgs()} 方法提供。
 *
 *
 * 典型用法
 * public static void main(String[] args) {
 *     PropertySource<?> ps = new SimpleCommandLinePropertySource(args);
 *     // ...
 * }
 * 有关完整的一般用法示例，请参阅{@link CommandLinePropertySource}。
 *
 *
 * 基础以外
 * 当需要更全面的命令行解析时，请考虑使用提供的{@link JOptCommandLinePropertySource}，或者根据您选择的命令行解析库实现您自己的{@code CommandLinePropertySource}
 */
public class SimpleCommandLinePropertySource extends CommandLinePropertySource<CommandLineArgs> {
}
```
```java
class CommandLineArgs {
	private final Map<String, List<String>> optionArgs = new HashMap<>();
	private final List<String> nonOptionArgs = new ArrayList<>();
}
```

##### 3.1.8 [CompositePropertySource](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/env/CompositePropertySource.java)
```java
/**
 * 复合 {@link PropertySource} 实现，迭代一组{@link PropertySource}实例。在多个资源共享同一个名称的情况下，是必要的。例如当 {@code @PropertySource} 提供多个值时。
 *
 * 从Spring 4.1.2开始，该类扩展了{@link EnumerablePropertySource}，而不是纯{@link PropertySource}，根据所有包含的源的累积属性名称展开 {@link #getPropertyNames()} （尽最大可能） ）。
 */
public class CompositePropertySource extends EnumerablePropertySource<Object> {
	private final Set<PropertySource<?>> propertySources = new LinkedHashSet<>();
}
```

### 3.2 PropertySource集合
##### 3.2.1 [PropertySources](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/env/PropertySources.java)
```java
public interface PropertySources extends Iterable<PropertySource<?>> {
	boolean contains(String name);
	PropertySource<?> get(String name);
}
```
##### 3.2.2 [MutablePropertySources](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/env/MutablePropertySources.java)
```java
/**
 * {@link PropertySources}接口的默认实现。 允许处理包含的多属性源，并提供一个构造函数来复制现有的{@code PropertySources}实例。
 * 在 {@link #addFirst} 和 {@link #addLast} 等方法中提及 优先级 的地方，这是关于在使用{@link PropertyResolver}解析给定属性时，搜索属性源的顺序。
 */
public class MutablePropertySources implements PropertySources {
	private final List<PropertySource<?>> propertySourceList = new CopyOnWriteArrayList<>();
}
```

## 4. Environment
![](/assets/img/spring/springEnvPropertyResolver.png)

### 4.1 接口
##### 4.1.1 [Environment](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/env/Environment.java)
```java
/**
 * 表示当前应用程序正在运行的环境的接口。 模拟应用程序环境的两个关键方面：profiles 和 properties 。 与属性访问相关的方法通过 {@link PropertyResolver} 父接口暴露。
 * 只有在给定的 profile 活动时，profile才是一个有名字的、注册在容器上的bean定义逻辑组。 bean可以被分配到不管是以XML定义还是以注释的profile; 有关语法详细信息，请参阅spring-beans 3.1 schema 或 {@link org.springframework.context.annotation.Profile @Profile}注释。 {@code Environment}对象与 profiles 的关系在于确定当前中哪些配置文件（如果有的话）{@linkplain #getActiveProfiles active} 活动 ，哪些配置文件（如果有的话）应该是{@linkplain #getDefaultProfiles 在默认情况下处于活动状态}。
 * Properties 在几乎所有的应用程序中起着重要的作用，可能来自各种来源：属性文件，JVM系统属性，系统环境变量，JNDI，servlet上下文参数，ad-hoc 属性对象，地图等。 环境对象与属性关系的作用是为用户提供方便的服务接口，用于配置属性源并从中解析属性。
 * 在{@code ApplicationContext}中管理的Bean可以注册为 {@link org.springframework.context.EnvironmentAware EnvironmentAware} 或 {@code @Inject} {@code Environment}，以便直接查询配置文件状态或解析属性。
 * 然而，在大多数情况下，应用级别的bean不需要直接与{@code Environment}进行交互，而是可能必须将 {@code ${...}} 属性值替换为属性占位符配置。例如，{@link org.springframework.context.support.PropertySourcesPlaceholderConfigurer PropertySourcesPlaceholderConfigurer}）本身就是{@code EnvironmentAware}，并且在Spring 3.1之前，使用 {@code getEnvironment()} 时，默认被注册。
 * 环境对象的配置必须通过从 {@code AbstractApplicationContext} 子类 {@code getEnvironment()} 方法返回的{@code ConfigurableEnvironment}接口完成。有关使用示例的示例，请参阅{@link ConfigurableEnvironment} Javadoc，演示在应用程序上下文{@code refresh()}之前操纵属性源。
 */
public interface Environment extends PropertyResolver {

	/**
	 * 返回显示为此环境激活的配置文件集。 配置文件用于创建有条件注册的bean定义的逻辑分组，例如基于部署环境。 可以通过将{@linkplain AbstractEnvironment#ACTIVE_PROFILES_PROPERTY_NAME "spring.profiles.active"} 设置为系统属性或调用 {@link ConfigurableEnvironment#setActiveProfiles(String...)} 来激活配置文件。
	 * 如果没有将配置文件明确指定为活动的，那么任何{@linkplain #getDefaultProfiles() 默认配置文件}将自动被激活。
	 */
	String[] getActiveProfiles();

	/**
	 * 当没有显式设置活动的配置文件时，默认情况下返回一组配置文件。
	 */
	String[] getDefaultProfiles();

	/**
	 * 返回一个或多个给定的配置文件是否处于活动状态，或者在没有显式活动配置文件的情况下，是否将一个或多个给定配置文件包含在默认配置文件集中。
	 * 如果配置文件以 '!' 开头 逻辑被反转，即如果给定的配置文件不是 活动，该方法将返回true。
	 * 例如，如果配置文件“p1”处于活动状态或“p2”不活动，则
	 * env.acceptsProfiles("p1", "!p2")
	 * 将返回{@code true}
	 */
	boolean acceptsProfiles(String... profiles);

}

```
##### 5.1.2 [ConfigurableEnvironment](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/env/ConfigurableEnvironment.java)
```java
/**
 * 配置接口由大多数（如果不是全部）{@link Environment}类型实现。
 * 提供设置活动和默认配置文件以及操纵底层属性源的功能。
 * 允许客户端通过{@link ConfigurablePropertyResolver}父接口，来设置和验证所需的属性，自定义转换服务等等。
 *
 * 操纵属性来源
 * 属性来源可被删除，重新排序或替换; 并且可以使用从 {@link #getPropertySources()} 返回的 {@link MutablePropertySources} 实例添加其他属性源。
 * 以下示例针对{@code ConfigurableEnvironment}的实现{@link StandardEnvironment}，但通常适用于任何实现，尽管特定的默认属性来源可能不同。
 *
 * 示例：添加具有最高搜索优先级的新属性源
 * ConfigurableEnvironment environment = new StandardEnvironment();
 * MutablePropertySources propertySources = environment.getPropertySources();
 * Map<String, String> myMap = new HashMap<String, String>();
 * myMap.put("xyz", "myValue");
 * propertySources.addFirst(new MapPropertySource("MY_MAP", myMap));
 *
 * 示例：删除默认的系统属性属性源
 * MutablePropertySources propertySources = environment.getPropertySources();
 * propertySources.remove(StandardEnvironment.SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME)
 *
 * 示例：mock系统环境进行测试
 * MutablePropertySources propertySources = environment.getPropertySources();
 * MockPropertySource mockEnvVars = new MockPropertySource().withProperty("xyz", "myValue");
 * propertySources.replace(StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, mockEnvVars);
 *
 * 当{@code ApplicationContext}正在使用{@link Environment}时，在上下文的{@link
 * org.springframework.context.support.AbstractApplicationContext#refresh() refresh()}方法被调用之前执行任何这样的{@code PropertySource}操作非常重要。
 * 这确保了所有属性源在容器引导过程中可用，包括被{@linkplain org.springframework.context.support.PropertySourcesPlaceholderConfigurer property placeholder configurers}使用。
 */
public interface ConfigurableEnvironment extends Environment, ConfigurablePropertyResolver {

	/**
	 * 为 {@code Environment} 指定一组活动的配置文件。在容器引导期间评估配置文件，以确定bean定义是否应该向容器注册。
	 * 任何现有的活动配置文件将被替换为给定的参数; 零参数调用来清除活动配置文件的当前集合。
	 * 使用{@link #addActiveProfile}添加配置文件同时保留现有集。
	 */
	void setActiveProfiles(String... profiles);

	/**
	 * 将配置文件添加到当前活动配置文件集中。
	 */
	void addActiveProfile(String profile);

	/**
	 * 如果没有其他配置文件通过{@link #setActiveProfiles}显式生效，则默认指定要设置为活动的配置文件集。
	 */
	void setDefaultProfiles(String... profiles);

	MutablePropertySources getPropertySources();

	/**
	 * 如果当前{@link SecurityManager}允许，返回 {@link System#getenv()} 的值，否则返回一个map实现，它将尝试通过调用 {@link System#getenv(String)} 访问各个键。
	 * 请注意，大多数{@link Environment}实现将将此系统环境映射作为默认{@link PropertySource}进行搜索。 因此，建议不要直接使用此方法，除非明确地意图绕过其他属性来源。
	 * 在返回的Map上调用 {@link Map#get(Object)} 将永远不会抛出{@link IllegalAccessException}; 在SecurityManager禁止访问属性的情况下，将返回{@code null}，并将发出INFO级日志消息，注明异常。
	 */
	Map<String, Object> getSystemEnvironment();

	/**
	 * 如果当前的 {@link SecurityManager} 允许，返回 {@link System#getProperties()} 的值，否则返回一个映射实现，它将尝试使用对 {@link System#getProperty(String)} 的调用访问各个键。
	 * 请注意，大多数{@code Environment}实现将将此系统属性映射作为默认{@link PropertySource}进行搜索。 因此，建议不要直接使用此方法，除非明确地意图绕过其他属性来源。
	 * 在返回的Map上调用 {@link Map#get(Object)} 将永远不会抛出{@link IllegalAccessException}; 在SecurityManager禁止访问属性的情况下，将返回{@code null}，并将发出INFO级日志消息，注明异常。
	 */
	Map<String, Object> getSystemProperties();

	/**
	 * 追加给定的父环境的活动配置文件，默认配置文件和属性源附加到此（子）环境各自的集合中。
	 * 对于存在于父项和子节点的任何同名命名的{@code PropertySource}实例，子实例将被保留并且父实例被丢弃。 这具有允许子级覆盖属性源的作用，并且避免通过公共属性源类型的冗余搜索，
	 * 例如 系统环境和系统属性。
	 * 活动和默认配置文件名称也被重复过滤，以避免混淆和冗余存储。
	 * 在任何情况下，父环境都未修改。 请注意，调用{@code merge}之后发生的父环境的任何更改都不会反映在该子环境中。 因此，在调用{@code merge}之前，应该小心配置父属性源和配置文件信息。
	 */
	void merge(ConfigurableEnvironment parent);

}
```

### 5.2 抽象与实现
##### 5.2.1 [AbstractEnvironment](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/env/AbstractEnvironment.java)
```java
/**
 * {@link Environment} 实现的抽象基类。 支持保留默认配置文件名称的概念，并通过 {@link #ACTIVE_PROFILES_PROPERTY_NAME} 和 {@link #DEFAULT_PROFILES_PROPERTY_NAME} 属性开启指定活动和默认配置文件。
 *
 * 具体的子类区别主要在于默认添加的 {@link PropertySource} 对象。 {@code AbstractEnvironment} 没有添加。 子类应通过受保护的{@link #customizePropertySources（MutablePropertySources）}钩子来提供属性源，而客户端应使用{@link ConfigurableEnvironment#getPropertySources()}进行自定义，并使用{@link MutablePropertySources} API。 有关使用示例，请参阅{@link ConfigurableEnvironment} javadoc。
 */
public abstract class AbstractEnvironment implements ConfigurableEnvironment {

	/**
	 * 系统属性指示Spring忽略系统环境变量，即永远不 会通过{@link System#getenv()}检索这样一个变量。
	 * 默认值为“false”，返回到系统环境变量，检查Spring环境属性（例如，配置字符串中的占位符）是否不可解析。 如果您遇到来自Spring的{@code getenv}呼叫的日志警告，请考虑将此标志切换为“true”，例如。 在WebSphere上使用严格的SecurityManager设置和AccessControlExceptions警告。...
	 */
	public static final String IGNORE_GETENV_PROPERTY_NAME = "spring.getenv.ignore";

	/**
	 * 要设置为指定活动配置文件的属性名称：{@value}。 值可以以逗号分隔。
	 * 请注意，某些shell环境（如Bash）不允许在变量名称中使用句点字符。 假设Spring的{@link SystemEnvironmentPropertySource}正在使用中，此属性可能被指定为环境变量{@code SPRING_PROFILES_ACTIVE}。
	 */
	public static final String ACTIVE_PROFILES_PROPERTY_NAME = "spring.profiles.active";

	/**
	 * 要设置的属性名称，以指定默认情况下活动的配置文件：{@value}。 值可以以逗号分隔。
	 * 请注意，某些shell环境（如Bash）不允许在变量名称中使用句点字符。 假设Spring的{@link SystemEnvironmentPropertySource}正在使用，该属性可能被指定为环境变量{@code SPRING_PROFILES_DEFAULT}。
	 */
	public static final String DEFAULT_PROFILES_PROPERTY_NAME = "spring.profiles.default";

	/**
	 * 保留的默认配置文件名称名称：{@value}。 如果没有明确的默认配置文件名称，并且没有显式设置活动配置文件名称，默认情况下将自动激活此配置文件。
	 */
	protected static final String RESERVED_DEFAULT_PROFILE_NAME = "default";

	private final Set<String> activeProfiles = new LinkedHashSet<>();

	private final Set<String> defaultProfiles = new LinkedHashSet<>(getReservedDefaultProfiles());

	private final MutablePropertySources propertySources = new MutablePropertySources(this.logger);

	private final ConfigurablePropertyResolver propertyResolver =
			new PropertySourcesPropertyResolver(this.propertySources);

}
```

##### 5.2.2 [StandardEnvironment](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/env/StandardEnvironment.java)
```java
/**
 * {@link Environment} 实现适用于“标准”（即非Web）应用程序。
 * 除了属性解析和配置文件相关操作之类的{@link ConfigurableEnvironment}的常规功能外，此实现还将配置两个默认属性源，以下列顺序进行搜索：
 * {@linkplain AbstractEnvironment#getSystemProperties() system properties}
 * {@linkplain AbstractEnvironment#getSystemEnvironment() system environment variables}

 * 也就是说，如果JVM系统属性和当前进程的一组环境变量中都存在密钥“xyz”，系统属性中的“xyz”键值将从一个调用返回到 {@code environment.getProperty("xyz")}。 默认情况下选择此排序，因为系统属性是每个JVM，而给定系统上的许多JVM的环境变量可能相同。 给予系统属性优先级允许在每个JVM的基础上覆盖环境变量。
 * 这些默认属性源可能会被删除，重新排序或替换; 并且可以使用 {@link #getPropertySources()} 中提供的 {@link MutablePropertySources} 实例添加其他属性来源。 有关使用示例，请参阅{@link ConfigurableEnvironment} Javadoc。
 *
 * 请参阅{@link SystemEnvironmentPropertySource} javadoc，了解在shell环境（例如Bash）中特殊处理属性名称的详细信息，该属性名称不允许变量名称中的句点字符。
 */
public class StandardEnvironment extends AbstractEnvironment {
	public static final String SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME = "systemEnvironment";
	public static final String SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME = "systemProperties";
}

```